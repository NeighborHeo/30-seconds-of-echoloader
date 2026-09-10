---
title: "사용자 버튼 → 화면 렌더링까지: 완전한 흐름 추적"
category: system
tags: [render-flow, threading, gc-parameter-bus, esmain, ovobject, frontpanel, acquisition]
difficulty: intermediate
---

SWE 버튼을 눌렀을 때 화면에 오버레이가 나타나기까지 코드가 지나치는 모든 경유지. 이 경로를 모르면 "왜 즉시 반영 안 되냐"는 질문에 답 못한다.

## Why / 왜 알아야 하나

gipc-app에서 UI 변경이 화면에 나타나는 경로는 **직선이 아니다**. 버튼 핸들러에서 직접 그래픽 객체를 호출하지 않는다. 대신 GC 파라미터 버스라는 비동기 채널을 경유한다. 이 구조를 모르면:

- "SetParameter를 바꿨는데 왜 바로 안 보이지?" → 최소 1 렌더 프레임(≤16ms) 지연은 설계다
- "Render()에서 직접 Manager를 호출하면 되지 않나?" → 스레드 안전성 보장 불가, 즉시 레이스 컨디션
- "왜 이렇게 복잡하게 만들었나?" → 렌더 스레드를 항상 읽기 전용 상태로 유지하기 위한 설계적 결정

## 경로 1: 사용자 버튼 입력 → 화면 표시

```
사용자가 SWE 버튼 누름
        │
        ▼
EchoFrontPanel::OnButtonPress("SWE")    ← UI 스레드 (MFC 메시지 펌프)
        │
        │  Windows 메시지 또는 직접 호출
        ▼
ESMain::HandleUserInput(ButtonEvent)    ← UI 스레드
        │
        │  ESMain::InElasto() — 게이팅: 현재 Elasto 모드인가?
        │  (아니면 조기 반환, 버튼 이벤트 무시)
        ▼
Manager::Instance(HandlerType::SWE)
  ->HandleInput(event)                  ← UI 스레드
        │
        │  내부 상태 전환
        │  m_state = AcqAssistState::Active
        │
        │  GC 파라미터 버스에 발행 ─────────────────────┐
        │  SetParameterValue(                          │
        │    "SWEAcqAssist.State",                     │
        │    static_cast<int>(Active))                 │
        │                                              │
        ▼                                              ▼
[Manager 함수 종료]                       GC 파라미터 버스가
[UI 스레드 반환]                           구독자에게 라우팅
                                                      │
                                                      ▼
                                     OVObject::SetParameter() 호출
                                     SWEAcquisitionAssistant::SetParameter()
                                          ← 호출자 스레드 (UI 또는 Acquisition)
                                       │
                                       │  m_state.store(Active,
                                       │    memory_order_release)
                                       │
                                       ▼
                               [다음 Render 틱까지 대기]
                               [최대 16ms — 한 프레임]
                                       │
                                       ▼
                          OVObject::Render()           ← GC Render 스레드
                          SWEAcquisitionAssistant::Render()
                            if (m_state.load(memory_order_acquire) == Active)
                                DrawSWEOverlay()
                                       │
                                       ▼
                              [모니터에 SWE 오버레이 표시]
```

### 왜 비동기 경유인가

렌더 스레드에 직접 함수를 호출하면 두 가지 문제가 생긴다:

1. **스레드 안전성**: 렌더 스레드가 읽는 도중 UI 스레드가 쓰면 torn read 발생
2. **렌더 예산 침범**: UI 스레드 작업이 렌더 루프에 섞이면 16ms 예산 계산 불가

GC 파라미터 버스는 **버퍼 역할**을 한다. UI/Acquisition 스레드가 값을 쓰면, 렌더 스레드는 다음 프레임에서 그 값을 읽는다. 두 스레드는 절대 동시에 같은 메모리에 접근하지 않는다 — `atomic`이나 Mutex가 그 경계를 지킨다.

## 경로 2: 프로브 → 프레임 취득 → 화면 표시

```
프로브 (초음파 트랜스듀서)
        │ 음향 신호 수신
        ▼
하드웨어 드라이버 / 신호 처리 DSP
        │ 디지털 프레임 생성
        ▼
ESMain::OnNewFrame(frame)               ← Acquisition 스레드
        │
        │  현재 모드 확인 (InElasto()?)
        ▼
Manager::Instance(HandlerType::SWE)
  ->OnNewFrame(frame)                   ← Acquisition 스레드
        │
        │  SWEROIProcessor로 계산 수행
        │  ROI 내 전단파 속도 측정
        │  신뢰도 지표 계산
        │
        ▼
결과 완성 → SetParameterValue(
              "SWEAcqAssist.Result",
              resultVariant)            ← GC 파라미터 버스에 발행
        │
        ▼
OVObject::SetParameter()               ← Acquisition 스레드 (버스 라우팅)
  m_result.store(parsedResult)
        │
        ▼ (다음 프레임)
OVObject::Render()                     ← GC Render 스레드
  auto result = m_result.load()
  DrawResult(result)
        │
        ▼
[SWE 컬러맵 / 수치 오버레이 표시]
```

### 두 경로의 타이밍

```
시간축 →

UI 스레드:    [버튼 눌림]──[HandleInput]──[SetParamValue]────────────────────▶
                                                │
GC Bus:                                    [라우팅]──[SetParameter 호출]──────▶
                                                              │
Acq 스레드:   [OnNewFrame]──────────────────[계산]──[SetParamValue]──────────▶
                                                         │
Render 스레드:                 [Render①]──────────[Render②]──[Render③(표시)]▶
                                                              ↑
                                                    최소 1 프레임 지연
                                                    (≤16ms, 설계 사항)
```

## 실패 시나리오: Mutex 없는 m_state 업데이트

```cpp
// ❌ SetParameter()에서 Mutex 없이 구조체 업데이트

struct RenderState
{
    int   mode;
    float opacity;
    bool  showLabel;
};

RenderState m_renderState;  // ❌ 보호 없음

void SetParameter(const std::string& key, const GcVariant& value) override
{
    // Acquisition 스레드에서 실행 중
    m_renderState.mode     = value.GetField<int>("mode");
    // ← 이 순간 Render()가 m_renderState를 읽으면?
    m_renderState.opacity  = value.GetField<float>("opacity");
    m_renderState.showLabel= value.GetField<bool>("showLabel");
}

void Render() override
{
    // GC Render 스레드에서 실행 중
    // mode=2, opacity=아직 이전 값, showLabel=아직 이전 값 → 찢긴 상태
    if (m_renderState.mode == 2)
        DrawWithOpacity(m_renderState.opacity, m_renderState.showLabel);
    // 결과: 간헐적으로 opacity=0.0으로 오버레이가 보이지 않거나
    //       showLabel=true인데 mode가 섞인 상태로 렌더링
}
```

이 버그의 특성:
- **재현율**: 로컬 디버그 빌드에서 거의 0% (CPU 여유 ↑, 타이밍이 맞지 않음)
- **임상 재현율**: 멀티 모니터 + 고부하 시 간헐적으로 발생
- **증상**: 오버레이가 깜빡이거나 순간적으로 사라졌다 나타남
- **추적**: 크래시 로그 없음, 덤프 없음 → 원인 추적 불가

## 왜 이 아키텍처가 임상 정확성에 유리한가

```
직접 호출 방식 (금지)           GC 버스 경유 방식 (설계)
──────────────────────         ──────────────────────────
Render() ←── UI 스레드가        SetParameterValue() 발행
             직접 조작          → 버스가 라우팅
             ↓                  → SetParameter() (Acquisition 스레드)
          스레드 안전성          → atomic/Mutex 업데이트
          보장 불가              → Render() (Render 스레드)
          렌더 예산              ↓
          침범 가능            스레드 경계 명확
                               렌더 스레드는 읽기만 함
                               버그 추적 경로 단순
```

렌더 스레드를 "읽기 전용" 스레드로 유지하면, 어떤 상태가 화면에 나타나는지 항상 예측할 수 있다. 의료기기에서 "예측 가능한 렌더링"은 성능이 아니라 **임상 안전성**의 문제다.

## Key Points

- 버튼 입력과 화면 반영 사이에는 최소 1 렌더 프레임(≤16ms) 지연이 있다. 이는 버그가 아니라 스레드 안전성을 위한 설계다.
- `ESMain::InElasto()` 게이팅을 통과하지 못하면 버튼 이벤트는 조용히 무시된다. "버튼이 안 먹힌다"는 버그의 50%는 여기서 막힌다.
- `SetParameter()`는 Render 스레드가 아니라 파라미터를 발행한 쪽(Acquisition/UI)의 스레드에서 실행된다. 이 함수 안에서 공유 멤버를 수정할 때는 atomic 또는 Mutex가 필수다.
- 복합 구조체를 Render()에서 읽을 때는 Mutex로 스냅샷을 만들고 Mutex 밖에서 그린다. Mutex 안에서 DrawXxx()를 호출하면 렌더 예산과 Acquisition 스레드 처리량이 동시에 깨진다.
- 프레임 취득 경로(프로브 → ESMain → Manager → SetParameterValue)와 사용자 입력 경로(FrontPanel → ESMain → Manager → SetParameterValue)는 모두 같은 GC 버스를 사용한다. 두 경로가 동시에 같은 키를 발행하면 마지막 쓴 값이 이긴다.
