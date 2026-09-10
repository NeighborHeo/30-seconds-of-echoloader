---
title: "gipc-app 스레드 모델: 어느 코드가 어느 스레드에서 실행되나"
category: system
tags: [threading, race-condition, render-thread, acquisition-thread, sccommon, atomic, mutex]
difficulty: advanced
---

신입 개발자가 레이스 컨디션을 만드는 1순위 원인. SetParameter()와 Render()가 다른 스레드에서 실행된다는 사실을 모르면 반드시 버그를 만든다.

## Why / 왜 알아야 하나

gipc-app은 26년된 실시간 의료기기 소프트웨어다. 멀티스레딩은 "성능 최적화"가 아니라 **구조적 필수 조건**이다:

- 렌더링이 UI를 막으면 의사가 진단 중 패널이 응답 없는 장비를 보게 된다
- 프레임 취득이 UI 스레드와 엮이면 60fps 초음파 스트림이 끊긴다
- AI 추론이 렌더 루프를 막으면 16ms 프레임 예산이 깨진다

이 규칙을 위반한 버그는 **임상 중에만 재현되는 간헐적 렌더링 오류**로 나타난다.

## 4개의 스레드

```
┌─────────────────────────────────────────────────────────────────────┐
│                        gipc-app 스레드 맵                            │
│                                                                     │
│  ┌──────────────────┐    ┌──────────────────┐                       │
│  │   UI Thread      │    │  GC Render Thread│                       │
│  │                  │    │                  │                       │
│  │ WinMain()        │    │ OVObject::        │                       │
│  │ MFC 메시지 펌프   │    │   Render()        │ ← 프레임마다 호출     │
│  │ EchoFrontPanel   │    │                  │   16ms 예산           │
│  │ 버튼 핸들러       │    │ [alloc 금지]      │                       │
│  │                  │    │ [I/O 금지]        │                       │
│  │ [100ms+ 작업 금지]│    │ [mutex 대기 금지] │                       │
│  └────────┬─────────┘    └────────▲─────────┘                       │
│           │                       │                                 │
│           │ Windows 메시지         │ SetParameterValue() 결과        │
│           ▼                       │ (GC 버스가 Render 전 반영)       │
│  ┌──────────────────┐    ┌────────┴─────────┐                       │
│  │ Acquisition      │───▶│  GC Parameter    │                       │
│  │ Thread           │    │  Bus             │                       │
│  │                  │    │  (fire-and-      │                       │
│  │ ESMain,          │    │   forget)        │                       │
│  │ EchoScanner,     │    └──────────────────┘                       │
│  │ Frame 처리        │                                               │
│  │ SWE/UGAP 계산    │    ┌──────────────────┐                       │
│  │                  │    │ ScCommon         │                       │
│  │ [UI 직접 조작 금지]│    │ Thread Pool      │                       │
│  └──────────────────┘    │                  │                       │
│                          │ AI 추론           │                       │
│                          │ 로그 플러시        │                       │
│                          │ 비동기 작업        │                       │
│                          │                  │                       │
│                          │ [렌더 상태 직접   │                       │
│                          │  수정 금지]       │                       │
│                          └──────────────────┘                       │
└─────────────────────────────────────────────────────────────────────┘
```

## 스레드 책임 일람표

```
스레드            | 실행되는 코드                    | 하면 안 되는 것
------------------|--------------------------------|---------------------------
UI Thread         | EchoFrontPanel 버튼 핸들러      | 100ms+ 작업 (UI 응답 불가)
                  | ESMain::HandleUserInput()       | 직접적인 렌더 상태 수정
                  | MFC 메시지 처리                 | ScCommon 스레드 생성
                  |                                |
GC Render Thread  | OVObject::Render()             | 힙 할당 (alloc/free)
                  | DrawXxx() 계열                  | 파일 I/O
                  | 오버레이 그리기                  | Mutex 대기 (데드락 위험)
                  |                                | 무거운 연산 (>16ms)
                  |                                |
Acquisition       | EchoScanner 프레임 처리         | UI 직접 조작 (MFC/Win32 금지)
Thread            | ESMain::OnNewFrame()            | OVObject::Render() 직접 호출
                  | SWE/UGAP 계산                   | GcUdtHandle 장시간 보유
                  | Manager::Instance()->OnNewFrame()|
                  |                                |
ScCommon          | AI 추론, 로그 플러시             | 렌더 루프 상태 직접 수정
Thread Pool       | 비동기 백그라운드 작업            | 장시간 UI 스레드 블로킹
```

## 위험 지대: SetParameter() ≠ Render() 스레드

**가장 많이 틀리는 부분이 여기다.**

GC 파라미터 버스로 전달된 값은 `OVObject::SetParameter()`를 통해 반영된다.
그런데 `SetParameter()`를 호출하는 것은 **호출자 스레드**(Acquisition 또는 UI 스레드)이고,
`Render()`를 호출하는 것은 **GC Render 스레드**다. 두 스레드가 동시에 같은 멤버에 접근한다.

```
Acquisition Thread          GC Render Thread
      │                           │
      │ SetParameter()            │ Render()
      │   m_state = 2  ─ ─ ─ ─ ─▶│   read m_state  ← 동시 접근!
      │                           │
```

이 창이 최대 16ms (한 프레임)밖에 안 되기 때문에 로컬 환경에서는 거의 재현 안 된다.
임상 중 환경(CPU 부하 ↑, 멀티 모니터 렌더링)에서만 나타난다.

### 잘못된 코드 vs 올바른 코드

```cpp
class SWEAcquisitionAssistant : public OVObject
{
    // ❌ 위험: SetParameter()와 Render()가 다른 스레드에서 접근
    //    int 읽기/쓰기가 원자적이 아닐 수 있음 (MSVC x86에서도 torn read 가능)
    int m_state;

    void SetParameter(const std::string& key, const GcVariant& value) override
    {
        if (key == "SWEAcqAssist.State")
            m_state = value.AsInt();  // ❌ Render()와 동시 접근
    }

    void Render() override
    {
        if (m_state == Active)  // ❌ 찢긴 값(torn read) 가능
            DrawSWEOverlay();
    }


    // ✅ 안전: atomic 사용 (단순 정수 상태)
    std::atomic<int> m_state;

    void SetParameter(const std::string& key, const GcVariant& value) override
    {
        if (key == "SWEAcqAssist.State")
            m_state.store(value.AsInt(), std::memory_order_release);
    }

    void Render() override
    {
        if (m_state.load(std::memory_order_acquire) == Active)
            DrawSWEOverlay();
    }


    // ✅ 안전: 복잡한 구조체는 ScCommon::Mutex로 보호
    struct Result { float value; bool valid; std::string label; };
    ScCommon::Mutex m_resultMutex;
    Result m_result;

    void SetParameter(const std::string& key, const GcVariant& value) override
    {
        if (key == "SWEAcqAssist.Result")
        {
            ScCommon::LockGuard lock(m_resultMutex);
            m_result = ParseResult(value);  // ✅ Mutex 보호
        }
    }

    void Render() override
    {
        Result snapshot;
        {
            ScCommon::LockGuard lock(m_resultMutex);
            snapshot = m_result;  // ✅ 복사 후 Mutex 해제
        }
        // Mutex 밖에서 렌더링 — lock 보유 시간 최소화
        if (snapshot.valid)
            DrawResult(snapshot);
    }
};
```

### Render()에서 Mutex 보유 시간을 최소화해야 하는 이유

Render 스레드는 16ms 프레임 예산을 갖는다. Mutex를 보유한 채 그리기 작업을 하면:
1. Acquisition 스레드가 해당 Mutex를 기다리며 블로킹된다
2. 프레임 처리가 지연된다
3. 결국 렌더 예산도 초과될 수 있다 (Mutex 경합이 Render에도 역풍)

**패턴:** `lock → 값 복사 → unlock → 복사본으로 렌더링`

## 스레드 간 통신의 유일한 안전한 경로

```
Acquisition Thread ──▶ SetParameterValue("Key", val) ──▶ GC 파라미터 버스
                                                               │
                                                               ▼
                                                    OVObject::SetParameter()
                                                    (호출자 스레드에서 실행)
                                                               │
                                                    atomic.store() 또는
                                                    Mutex 보호 멤버 업데이트
                                                               │
                                                               ▼ (다음 프레임)
                                                    OVObject::Render()
                                                    (GC Render 스레드에서 실행)
                                                    atomic.load() 또는
                                                    Mutex로 스냅샷 획득
```

이 경로 외의 직접 접근(포인터 공유, 전역 변수 등)은 레이스 컨디션으로 이어진다.

## Key Points

- `OVObject::SetParameter()`는 **GC Render 스레드가 아니라** 파라미터를 보낸 쪽(Acquisition/UI) 스레드에서 실행된다. 이 사실을 모르면 공유 멤버에 Mutex 없이 접근하게 된다.
- `OVObject::Render()`는 GC Render 스레드 전용이다. 여기서 힙 할당, 파일 I/O, Mutex 대기를 하면 16ms 프레임 예산이 깨지고 임상 중 프레임 드롭이 발생한다.
- 단순 정수/bool 상태는 `std::atomic<T>`로, 복잡한 구조체는 `ScCommon::Mutex` + 복사 패턴으로 보호한다.
- Render()에서 Mutex를 보유하는 시간은 `값 복사` 만큼만이어야 한다. 렌더링 연산을 Mutex 안에서 하면 Acquisition 스레드가 블로킹된다.
- gipc-app에는 `THREAD_SAFE` 어노테이션이 없다. 스레드 친화도는 주석으로 명시하고, 코드리뷰가 유일한 게이트다 — 자동 검출 도구가 없으므로 신규 공유 상태는 반드시 리뷰어가 확인해야 한다.
