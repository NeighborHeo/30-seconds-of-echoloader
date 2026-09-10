---
title: "방송국 비유로 이해하는 gipc-app 전체 구조"
category: system
tags: [architecture, mental-model, gc-parameter-bus, echoScanner, ovobject, layer-a, layer-b, broadcast]
difficulty: beginner
---

gipc-app의 Layer A / Layer B / GC 버스 관계를 5분 만에 이해하는 비유. 이 모델이 머릿속에 잡히면 "이 기능이 어느 파일에?" 질문에 즉시 답할 수 있다.

## Why / 왜 알아야 하나

40,000개 C++ 파일, 26년 히스토리. 어디서부터 봐야 할지 모르면 하루 종일 grep만 하게 된다. 이 비유는 거기서 탈출하는 첫 번째 열쇠다.

gipc-app을 처음 보는 사람이 가장 많이 하는 실수:
- Layer A 코드(EchoScanner)에서 화면 그리기를 시도함
- Layer B 코드(OVObject)에서 하드웨어 상태를 직접 수정하려 함
- "두 레이어가 왜 직접 함수를 호출 안 하나?" 하고 인터페이스를 추가함

이 비유가 그 실수를 사전에 막는다.

## 방송국 비유

```
방송국 세계                    gipc-app 세계
──────────────────────────     ────────────────────────────────────────

카메라/스튜디오           ↔    EchoScanner (Layer A)
  └ 촬영, 음향 처리              └ 초음파 취득, 신호 처리, 상태 관리
  └ 방송 내용의 원천              └ SWEAcqAssistHandler, UGAPAcqAssistHandler
                                  AcquisitionAssistant::Manager (싱글톤)
                                  ESMain (11개 진입점)

방송 신호 (RF / 위성)     ↔    GC 파라미터 버스
  └ 단방향, 타입 없음            └ string 키 / GcVariant 값, fire-and-forget
  └ 수신자가 누구든 상관없음       └ SetParameterValue("Key", val) 발행
  └ 신호 손실이 있어도 방송 계속   └ 구독자 없어도 발행 측은 계속 동작

TV 수신기                 ↔    ObjectViewer / OVObject (Layer B)
  └ 신호를 받아 화면 표시         └ OVObject::SetParameter() → Render()
  └ 여러 채널(OVObject)           └ SWEAcquisitionAssistant, UGAPAcquisitionAssistant
  └ 채널마다 독립 표시            └ 각 OVObject가 독립적으로 파라미터 구독

리모컨                    ↔    EchoFrontPanel
  └ 채널 변경, 볼륨 조절          └ 버튼/노브/터치 이벤트 → ESMain으로 전달
  └ TV를 직접 제어하지 않음       └ OVObject를 직접 호출하지 않음

TV 가이드 / 편성표         ↔    UDT 객체 그래프 + 팩토리 태그
  └ 어떤 채널이 있는지 등록        └ "Gc.SWEAcqAssist", "Gc.UGAPAcqAssist" 태그
  └ 수신기가 채널을 찾는 기준      └ OVObject.cpp에서 팩토리 등록, ObjectViewerImpl이 매핑

방송국 규정 (전파법)       ↔    GE 코딩 표준 + IEC 62304
  └ 신호 규격 준수 의무            └ 레이어 경계 위반 불가
  └ 규정 위반 = 면허 취소           └ 위반 = 인증 실패, 시장 회수 가능성
```

## 아키텍처를 방송국으로 시각화

```
                    ┌─────────────────────────────────┐
                    │      EchoFrontPanel (리모컨)      │
                    │  버튼 / 노브 / 터치 이벤트         │
                    └──────────────┬──────────────────┘
                                   │ 이벤트 전달
                                   ▼
┌──────────────────────────────────────────────────────┐
│                 EchoScanner / ESMain (카메라팀)        │
│                                                      │
│  프로브 ─▶ 신호처리 ─▶ 프레임 생성                     │
│                  │                                   │
│  Manager::Instance(SWE)->HandleInput() / OnNewFrame() │
│                  │                                   │
│         계산 완료, 상태 결정                           │
└──────────────────┬───────────────────────────────────┘
                   │
                   │  SetParameterValue("SWEAcqAssist.State", 2)
                   │  SetParameterValue("SWEAcqAssist.Result", ...)
                   │
                   ▼
        ┌──────────────────┐
        │   GC 파라미터 버스  │  ← 방송 신호
        │   (fire-and-     │     누가 듣든 상관없이 발행
        │    forget)       │
        └──────┬───────────┘
               │  라우팅
               ▼
┌──────────────────────────────────────────────────────┐
│              ObjectViewer / OVObject (TV팀)           │
│                                                      │
│  SWEAcquisitionAssistant::SetParameter()              │
│    m_state.store(2)   // Acquisition 스레드           │
│                                                      │
│  SWEAcquisitionAssistant::Render()                   │
│    DrawSWEOverlay()   // GC Render 스레드             │
│                                                      │
│  UGAPAcquisitionAssistant (별도 채널, 독립 동작)       │
└──────────────────────────────────────────────────────┘
               │
               ▼
        ┌──────────────┐
        │   모니터 출력  │
        └──────────────┘
```

## 비유가 깨지는 곳 (그리고 왜 중요한가)

비유가 완벽하지 않은 세 지점을 알아야 실제 코드에서 혼동하지 않는다.

### 1. 단방향 방송이 아닌 역방향 파라미터 존재

TV는 방송국에 신호를 보내지 않는다. 하지만 gipc-app의 Layer B(OVObject)는 **제한적으로 Layer A에 파라미터를 역방향 발행**할 수 있다.

```
예: AutoPos (자동 ROI 위치 조정)

사용자가 마우스로 ROI 드래그
        │
        ▼
SWEAcquisitionAssistant::OnMouseDrag()   ← OVObject (Layer B)
        │
        │ 역방향 발행
        ▼
SetParameterValue("SWEAcqAssist.AutoPos", newPosition)
        │
        ▼
ESMain::OnAutoPos()   ← Layer A가 수신하여 취득 파라미터 조정
```

이 역방향은 **극히 제한적**이고, 반드시 파라미터 버스를 통해서만 한다. Layer B 코드가 Layer A의 함수를 직접 호출하는 것은 금지다.

### 2. TV 채널은 정적이지만 OVObject는 동적으로 켜지고 꺼진다

방송 채널 목록은 편성표에 고정되어 있다. 하지만 OVObject는 모드 진입/퇴장에 따라 동적으로 활성화/비활성화된다.

```
SWE 모드 진입: OVObject::OnActivate()   → GcUdtHandle.IsValid() == true
SWE 모드 퇴장: OVObject::OnDeactivate() → GcUdtHandle.IsValid() == false
```

`SetParameter()`나 `Render()`는 **Active 상태일 때만** 호출된다. 비활성 OVObject는 파라미터를 수신하지 않는다. "파라미터 발행했는데 왜 반응 없지?"의 절반은 OVObject가 꺼져 있기 때문이다.

### 3. 수신 확인(ACK) 없는 발행 — 하지만 일부는 응답 기대

방송은 수신 확인이 없다. 하지만 일부 GC 파라미터는 Layer A가 Layer B로 "질의"하고 역방향 파라미터로 "응답"을 받는 패턴을 사용한다.

```
Layer A 발행: "SWEAcqAssist.RequestSnapshot" = true
Layer B 수신: SetParameter()에서 처리
Layer B 역발행: "SWEAcqAssist.Snapshot" = <계산된 결과>
Layer A 수신: 다음 프레임 처리에 반영
```

이것은 요청-응답처럼 보이지만 **동기 함수 호출이 아니다**. 두 번의 비동기 버스 발행이다.

## 비유를 실무에 적용하는 매핑 테이블

```
질문                                    | 방송국에서 찾아라
----------------------------------------|----------------------------------------------
"이 기능 코드가 어디에 있나?"             | 카메라팀(EchoScanner/ESMain/AcqAssist)?
                                        | TV팀(GcViewer/ObjectViewer/OVObject)?
                                        | 취득 로직 → A, 표시 로직 → B

"두 기능이 왜 직접 함수 호출 안 해?"      | 방송 규정: 레이어 간 직접 호출 금지
                                        | SetParameterValue()가 유일한 통신 경로

"이 파라미터가 왜 OVObject에 반응 없어?" | 수신기(OVObject)가 OnActivate 됐나?
                                        | 파라미터 키 이름 오타?
                                        | SetParameter() 구현이 해당 키를 처리하나?

"새 기능 어디에 추가해야 하나?"           | 취득/계산 → Layer A (EchoScanner 패키지)
                                        | 화면 표시 → Layer B (GcViewer 패키지)
                                        | 둘 다 필요 → A에서 계산 후 버스로 B에 전달

"OVObject에서 ESMain 함수 직접 호출 가능?" | TV가 카메라를 제어할 수 없다
                                        | 역방향은 버스 발행으로만 (극히 제한적)

"파라미터 발행 후 즉시 화면에 반영되나?"  | 아니다. 다음 방송 수신 타이밍(≤16ms)까지 대기
                                        | 이건 버그가 아니라 채널 전환 지연
```

## Key Points

- "취득이냐 표시냐"가 코드 위치 판단의 첫 번째 질문이다. 이 답이 Layer A(EchoScanner)냐 Layer B(GcViewer)냐를 결정한다. 둘을 섞는 순간 레이어 경계가 무너진다.
- GC 파라미터 버스는 fire-and-forget이다. 발행 측은 수신자가 있는지, 처리됐는지 알 수 없다. 처리 확인이 필요하면 역방향 파라미터 패턴(Layer B → A 발행)을 사용해야 한다. 동기 반환값은 없다.
- OVObject가 반응 없으면 세 가지를 먼저 확인한다: (1) OnActivate 됐나, (2) 파라미터 키 이름이 정확한가 ("Gc." 접두사 등), (3) SetParameter() 구현이 해당 키를 처리하는 분기가 있나.
- Layer B가 Layer A를 직접 호출하고 싶은 충동이 들면 방송국 비유를 떠올려라. TV가 카메라를 직접 제어하지 않는 이유가 있다 — 의존성이 역전되면 Layer A 없이 Layer B를 테스트할 수 없고, 레이어 경계 붕괴는 26년 레거시 코드베이스에서 되돌리기 극히 어렵다.
