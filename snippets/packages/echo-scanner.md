---
title: "EchoScanner — 초음파 스캐닝 엔진"
category: packages
tags: [acquisition, SWE, UGAP, state-machine, factory]
difficulty: advanced
---

초음파 신호 획득(Acquisition)의 핵심 엔진. AcquisitionAssistant 패턴의 Manager/Factory/Handler 인터페이스가 여기에 정의되고, ESMain.cpp가 11개 호출 사이트를 가진다.

## 역할

- 초음파 모드별(SWE, UGAP 등) 획득 파라미터 관리
- `AcquisitionAssistant` 패턴: Manager가 Factory로 Handler 인스턴스를 생성·교체
- `ESMain.cpp` — 스캐닝 루프 메인, AA 호출 사이트 집중

## 위치

```
src/packages/EchoScanner/
├── AcquisitionAssistant/
│   ├── IAcqAssistManager.h      ← Manager 인터페이스
│   ├── IAcqAssistFactory.h      ← Factory 인터페이스
│   └── IAcqAssistHandler.h      ← Handler 인터페이스
├── SWEAcqAssistHandler.h/.cpp   ← SWE 구현체
├── UGAPAcqAssistHandler.h/.cpp  ← UGAP 구현체
└── ESMain.cpp                   ← 스캐닝 루프 (11개 AA 호출)
```

## 핵심 클래스

| 클래스 | 역할 |
|---|---|
| `IAcqAssistManager` | AA 라이프사이클(Init/Start/Stop) 제어 |
| `IAcqAssistFactory` | 모드에 따라 Handler 인스턴스 생성 |
| `IAcqAssistHandler` | 획득 이벤트 처리 인터페이스 |
| `SWEAcqAssistHandler` | Shear Wave Elastography 획득 구현 |
| `UGAPAcqAssistHandler` | Ultrasound Guided Attenuation Parameter 획득 구현 |

## 의존 관계

```
EchoScanner
  → EchoConfig    (프리셋 파라미터)
  → EchoRoot      (FatalError 훅)
  ← GcViewer      (AA 구현체 상속·확장)
  ← EchoLoader    (패키지 초기화)
```

## 주의사항

- `ESMain.cpp` 11개 호출 사이트는 AA 교체 시 모두 영향을 받음 — 리팩토링 전 전체 사이트 확인 필수
- Handler 추가 시 Factory에 등록 누락되면 런타임에만 발견됨 — 단위 테스트로 커버할 것
- SWE/UGAP Handler는 동일 인터페이스를 구현하지만 획득 파라미터 도메인이 다름 (kPa vs dB/cm/MHz)
