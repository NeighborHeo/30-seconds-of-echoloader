---
title: "새 스캔 모드 추가 시 7개 수정 지점 체크리스트"
category: workflow
tags: [scan-mode, acqassist, grandcentral, ovobject, dicom, iec62304]
difficulty: advanced
---

완전히 새로운 획득 모드(예: cSWE, 새 UGAP 변형)를 추가할 때 수정해야 할 7개 지점. 하나라도 빠지면 무음 실패(silent failure)가 발생한다.

## 언제 이 아티클을 보나

- 기존 모드를 복사하는 게 아니라 완전히 새로운 스캔 모드를 추가한다
- add-new-ovobject.md만으로는 부족하다는 것을 알고 있다
- PR 전 누락 지점 없는지 교차 검증이 필요하다

## 새 스캔 모드 추가 체크리스트 (예: NewMode)

```
[ ] 1. AcqAssistState enum 양쪽 동기화
    파일: src/EchoScanner/AcquisitionAssistant/AcqAssistHandler.h
          src/GcViewer/ObjectViewer/AcquisitionAssistantBase.h (또는 동기화된 Layer B 헤더)
    작업: enum AcqAssistState { ..., NewModeActive } 추가
    주의: 두 파일의 값이 동일해야 함 (GC 버스로 int 전달)

[ ] 2. 게이팅 술어 추가 또는 확장
    파일: src/EchoScanner/ESMain.cpp
    작업: bool IsInNewMode() 함수 추가 또는 기존 술어 조건 확장
    패턴: InElasto()와 IsInUGAPMode()처럼 배타적 조건

[ ] 3. ESMain AA 호출 사이트 추가
    파일: src/EchoScanner/ESMain.cpp (7066~9954 라인 범위)
    작업: if (IsInNewMode()) Manager::Instance(HandlerType::NewMode)->OnNewFrame(frame);
    주의: 11개 기존 사이트 패턴 그대로 따를 것

[ ] 4. HandlerType enum + Handler 구현
    파일: src/EchoScanner/AcquisitionAssistant/Manager.h
          새 파일: NewModeAcqAssistHandler.h/.cpp
    작업: HandlerType::NewMode 추가, IAcqAssistHandler 구현체 작성

[ ] 5. GC 파라미터 키 네임스페이스 추가
    파일: src/EchoScanner/AcquisitionAssistant/GcParamKeys.h (또는 신규 생성)
    작업: namespace GcParamKeys { constexpr const char* NewModeState = "NewMode.State"; }
    패턴: "PackageName.Feature" 형식

[ ] 6. Layer B OVObject 추가 (Gc.NewMode 태그)
    파일: src/GcViewer/ObjectViewer/OVObject.cpp (팩토리 등록)
          src/GcViewer/ObjectViewer/ObjectViewerImpl.cpp (dynamic_cast)
          새 파일: NewModeAcquisitionAssistant.h/.cpp
    참고: add-new-ovobject.md의 3단계 절차

[ ] 7. DICOM SR 코드 매핑 추가 (측정값이 있는 경우)
    파일: EchoWorksheet 관련 DICOM SR 생성 코드
    작업: 새 모드의 측정값을 위한 DICOM 태그/코드 추가
    주의: UCUM 단위 코드 + SNOMED 코드 쌍으로 추가

[ ] 8. 단위 테스트 추가 (IEC 62304 Class C 요건)
    작업: NewModeAcqAssistHandlerTest 픽스처
          AcqAssistState::NewModeActive를 포함한 상태 전환 테스트
```

## 누락 항목별 증상

```
누락 항목      | 증상
--------------|------------------------------------
1 (enum)      | Layer B가 Unknown 상태 수신 → 묵음 무시
2 (게이팅)     | 잘못된 모드에서 NewMode 로직 실행
3 (ESMain)    | NewMode 진입해도 AA 콜백 없음
4 (Handler)   | Manager::Instance(NewMode) 예외
5 (GC 키)     | 오타로 파라미터 드롭
6 (OVObject)  | Layer B에 아무것도 표시 안 됨
7 (DICOM)     | 측정값 저장 시 규정 위반
```

## Key Points

- `AcqAssistState` enum은 Layer A(EchoScanner)와 Layer B(GcViewer) 헤더가 별도로 존재한다 — int로 GC 버스를 통해 전달되므로 두 파일의 값이 불일치하면 Layer B가 잘못된 상태로 해석한다
- 게이팅 술어(`IsInNewMode()`)를 먼저 추가하지 않으면 ESMain 호출 사이트가 기존 모드와 NewMode를 동시에 실행하는 조건을 만들 수 있다
- GC 파라미터 키(`GcParamKeys::NewModeState`)는 반드시 `"PackageName.Feature"` 형식을 따른다 — 오타는 파라미터 드롭으로 무음 실패하며 컴파일 오류가 없다
- DICOM SR 매핑(항목 7)은 기능 동작과 무관하게 보이지만 IEC 62304 Class C 소프트웨어에서는 측정값 저장 경로가 규정 위반이 되므로 동시에 처리한다
- 단위 테스트(항목 8)는 상태 전환 검증이 핵심이다 — `AcqAssistState::NewModeActive`를 포함한 진입/이탈 시나리오를 커버해야 IEC 62304 추적성 요건을 만족한다
