---
title: "GcViewer / ObjectViewer — 렌더링 레이어"
category: packages
tags: [rendering, OVObject, dynamic-cast, SWE, UGAP, LiverWarning]
difficulty: advanced
---

화면에 초음파 영상과 오버레이를 그리는 렌더링 레이어. `OVObject` 상속 계층의 기반이며, AA(AcquisitionAssistant) 구현체와 경고 패널이 여기에 산다.

## 역할

- `OVObject` 계층을 통한 화면 객체(overlay, panel, annotation) 관리
- SWE / UGAP AcquisitionAssistant의 **구현체**를 호스팅 (EchoScanner의 인터페이스 구현)
- `LiverWarning3Panel` — 간 경직도 측정 경고 UI 패널
- `ObjectViewerImpl.cpp` — `dynamic_cast` 기반 타입 분기 처리

## 위치

```
src/packages/GcViewer/ObjectViewer/
├── OVObject.h                         ← 렌더링 객체 기반 클래스
├── AcquisitionAssistantBase.h/.cpp    ← 공통 AA 기반 (926+277 lines)
├── SWEAcquisitionAssistant.h/.cpp     ← SWE AA 구현 (469+125 lines)
├── UGAPAcquisitionAssistant.h/.cpp    ← UGAP AA 구현 (781+139 lines)
├── LiverWarning3Panel.h/.cpp          ← 경고 UI 패널
└── ObjectViewerImpl.cpp               ← dynamic_cast 분기 허브
```

## 핵심 클래스

| 클래스 | 역할 |
|---|---|
| `OVObject` | 화면 객체 기반; 모든 패널·오버레이가 상속 |
| `AcquisitionAssistantBase` | SWE/UGAP 공통 획득 로직 (926줄) |
| `SWEAcquisitionAssistant` | SWE 렌더링 + 획득 구현 |
| `UGAPAcquisitionAssistant` | UGAP 렌더링 + 획득 구현 |
| `LiverWarning3Panel` | 간 경직도 3단계 경고 패널 |

## 의존 관계

```
GcViewer/ObjectViewer
  → EchoScanner  (IAcqAssist* 인터페이스 구현)
  → EchoConfig   (프리셋)
  ← EchoDecisionSupport  (LiverWarning3Panel 소비)
  ← EchoLoader   (렌더링 레이어 초기화)
```

## 주의사항

- `ObjectViewerImpl.cpp`의 `dynamic_cast` 분기: 새 AA 타입 추가 시 분기 누락 주의
- `AcquisitionAssistantBase` 926줄 — 변경 시 SWE/UGAP 양쪽 회귀 테스트 필수
- `LiverWarning3Panel`은 EchoDecisionSupport가 소비하므로 패널 API 변경 시 양쪽 확인
