---
title: "새 코드는 어디에 짜나: Layer A vs Layer B 결정 기준"
category: architecture
tags: [layer-a, layer-b, echoscanner, gcviewer, ovobject, architecture]
difficulty: beginner
---

새 기능이나 버그 수정을 시작할 때, 코드를 어느 레이어 어느 파일에 넣을지 결정하기 위한 아티클.

## 언제 이 아티클을 보나

새 기능을 받았는데 EchoScanner에 짜야 할지 GcViewer에 짜야 할지 모를 때. 또는 기존 코드를 수정하면서 "이 로직이 여기 있는 게 맞나?" 의심이 들 때.

## Layer A vs Layer B 결정 트리

```
Q1: 이 기능이 초음파 신호를 처리하거나, 스캔 모드를 제어하거나, 하드웨어와 상호작용하나?
  → YES: EchoScanner (Layer A)로 간다
  → NO:  다음 질문

Q2: 이 기능이 화면에 무언가를 그리거나, 사용자에게 시각적으로 표시되나?
  → YES: GcViewer / ObjectViewer (Layer B)로 간다
  → NO:  다음 질문

Q3: 이 기능이 기존 측정값, AI 결과, 경고를 표시하는 건가?
  → YES: Layer B — 관련 OVObject에 로직 추가 (또는 새 OVObject 생성)
  → NO:  계속...

Q4: 이 기능이 스캔 파라미터(게인, 깊이, 프레임레이트)를 읽거나 쓰나?
  → YES: Layer A
  → NO:  계속...

Q5: 이 기능이 타이밍, 트리거, 상태 머신을 다루나?
  → YES: Layer A
  → NO:  Layer B (표시/UI 로직일 가능성이 높음)
```

## Layer A 내부 — 어디에 짜나?

```
Q: ESMain에서 모드 게이팅(InElasto/IsInUGAPMode)이 필요한 새 진입점인가?
  → YES: ESMain.cpp에 게이팅 추가 + Manager::Instance()에 위임
  → NO:  Manager/Handler 내부에만 변경

Q: SWE와 UGAP 모두에 적용되나, 하나에만 적용되나?
  → 둘 다: IAcqAssistHandler에 새 가상 함수 추가 → SWEAcqAssistHandler + UGAPAcqAssistHandler 양쪽 구현
  → SWE만: SWEAcqAssistHandler에만
  → UGAP만: UGAPAcqAssistHandler에만

Q: 복수 핸들러에 공통 로직인가?
  → YES: AcquisitionAssistantBase에 추가
  → NO:  각 핸들러에만
```

## Layer B 내부 — 어디에 짜나?

```
Q: 기존 OVObject(SWEAcquisitionAssistant 등)의 Render()에 추가할 수 있나?
  → YES (해당 객체의 책임 범위 내): 기존 OVObject에 추가
  → NO  (다른 책임, 독립적 수명주기): 새 OVObject 생성

Q: Layer A의 상태를 화면에 반영하기만 하나?
  → YES:         SetParameter() 수신 + Render() 표시 패턴
  → YES + 계산:  SetParameter()로 입력 받음 → 내부 계산 → Render()
  → NO:          OVObject 신규 생성 여부를 재검토

Q: 새 OVObject를 만든다면, 어디에 등록하나?
  → OVObject.cpp 약 222번째 줄 — 기존 등록 패턴 옆에 추가
```

## 경계에서 헷갈리는 케이스

**경고 표시 로직**
- 경고 조건 판단 (어떤 상태일 때 경고인가) → Layer A
- 경고 아이콘/텍스트를 화면에 그리는 것 → Layer B OVObject

**측정값 계산**
- 계산 자체 (수식, 알고리즘) → Layer A
- 계산 결과를 화면에 숫자로 표시 → Layer B

**AI 추론**
- 입력 데이터 준비, 파라미터 전달 → Layer A가 SetParameterValue로 전달
- 추론 실행 + 결과 표시 → Layer B OVObject 내부

**모드 전환 시 UI 업데이트**
- 모드 전환 결정 → Layer A (ESMain, 게이팅 술어)
- 전환 후 오버레이 변경 → Layer B (SetParameter 수신 후 Render 업데이트)

## 한 줄 판단 규칙

> 기능 설명에 **"계산", "판단", "모드", "하드웨어"** 가 들어가면 **Layer A**.
> **"표시", "그리기", "오버레이", "UI"** 가 들어가면 **Layer B**.

## Key Points

- Layer A(EchoScanner)는 신호 처리·모드 제어·하드웨어 상호작용 전담. Layer B(GcViewer)는 표시·오버레이·UI 전담.
- ESMain.cpp는 진입점만 갖는다. 로직은 반드시 Manager/Handler 내부로 위임.
- SWE/UGAP 공통 로직 → IAcqAssistHandler 가상 함수. 모드별 로직 → 각 Handler 구현체.
- OVObject 신규 생성 기준: 독립적 수명주기 또는 다른 책임일 때만. 기존 OVObject 범위 내면 Render()에 추가.
- Layer A → Layer B 데이터 전달은 GC 파라미터(SetParameterValue/SetParameter) 경유. 직접 호출 금지.
