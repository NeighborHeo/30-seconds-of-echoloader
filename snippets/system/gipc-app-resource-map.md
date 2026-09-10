---
title: "gipc-app 전체 리소스 구조도"
category: system
tags: [resource-map, packages, layers, overview, architecture]
difficulty: intermediate
---

gipc-app의 ~50개 패키지가 어떻게 배치되고 연결되는지 한눈에 보여주는 전체 구조도.

## 패키지 레이어 구조

```
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 4: Application / UI                                           │
│                                                                      │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌────────────┐ │
│  │ EchoLoader   │ │EchoFrontPanel│ │EchoWorksheet │ │EchoConfig  │ │
│  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ └─────┬──────┘ │
│         │                │                │               │         │
│  ┌──────┴───────────────────────────────────────────────┐ │         │
│  │           EchoDecisionSupport                        │ │         │
│  └──────────────────────────────────────────────────────┘ │         │
└──────────────────────┬──────────────────────────────────────────────┘
                       │ depends on ↓
┌──────────────────────▼──────────────────────────────────────────────┐
│  Layer 3: Domain Logic                                               │
│                                                                      │
│  ┌──────────────────────────┐   ┌──────────────┐  ┌─────────────┐  │
│  │ EchoScanner (Layer A)    │   │ EchoMeasure  │  │ EchoSysMon  │  │
│  │                          │   └──────────────┘  └─────────────┘  │
│  │  ESMain.cpp (~10k lines) │                                        │
│  │  ├── AcquisitionAssistant│                                        │
│  │  │   Manager (SWE/UGAP)  │                                        │
│  │  └── GC 파라미터 발행     │                                        │
│  └──────────┬───────────────┘                                        │
│             │ GC param bus only                                       │
└─────────────│───────────────────────────────────────────────────────┘
              │ depends on ↓
┌─────────────▼───────────────────────────────────────────────────────┐
│  Layer 2: Display / Rendering (Layer B)                              │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  GcViewer / ObjectViewer                                        │ │
│  │                                                                 │ │
│  │  ┌─────────────────────────────────────────────────────────┐   │ │
│  │  │  GrandCentral (Gc) 프레임워크                             │   │ │
│  │  │                                                           │   │ │
│  │  │  UDT 객체 그래프 (~30개 OVObject)                         │   │ │
│  │  │  ├── [Gc.SWEAcqAssist]  → SWEAcquisitionAssistant       │   │ │
│  │  │  ├── [Gc.UGAPAcqAssist] → UGAPAcquisitionAssistant      │   │ │
│  │  │  ├── [Gc.LiverAI]       → LiverAIProcessor               │   │ │
│  │  │  └── ... (기타 OVObject들)                                │   │ │
│  │  │                                                           │   │ │
│  │  │  GC 파라미터 버스 (문자열 키/값)                            │   │ │
│  │  │  SetParameterValue("SWEAcqAssist.State", value)          │   │ │
│  │  └─────────────────────────────────────────────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌──────────────┐                                                    │
│  │   EchoRoot   │  ← 시스템 루트, 전체 생명주기 관리                  │
│  └──────────────┘                                                    │
└──────────────────────────────────────────────────────────────────────┘
              │ depends on ↓
┌─────────────▼───────────────────────────────────────────────────────┐
│  Layer 1: Infrastructure                                             │
│                                                                      │
│  ┌────────────┐ ┌────────────────┐ ┌─────┐ ┌────────────────────┐  │
│  │  ScCommon  │ │ ScLogsDatabase │ │ mcd │ │ target/include/    │  │
│  │            │ │                │ │     │ │ (Eigen, 3rd-party) │  │
│  │ Thread.h   │ │ 진단/감사 로그  │ │     │ └────────────────────┘  │
│  │ Mutex.h    │ │ DB 저장        │ └─────┘                          │
│  │ Timer.h    │ └────────────────┘                                   │
│  │ Path.h     │                                                       │
│  │ AuditLog.h │                                                       │
│  └────────────┘                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

## 데이터 흐름 다이어그램

```
초음파 프로브
     │
     │ RF 신호
     ▼
┌────────────┐     ┌──────────────┐     ┌───────────────────┐
│  Hardware  │────▶│  EchoScanner │────▶│  GC 파라미터 버스  │
│  (Probe,   │     │  (Layer A)   │     │  (문자열 키/값)    │
│   Scanner) │     │              │     └─────────┬─────────┘
└────────────┘     │  AcqAssist   │               │ SetParameter()
                   │  Manager     │               ▼
                   └──────────────┘     ┌───────────────────┐
                                        │  ObjectViewer     │
┌────────────┐                          │  (Layer B)        │
│Front Panel │────▶ ESMain              │                   │
│  (버튼,    │     (게이팅+라우팅)       │  OVObject 그래프  │
│   노브)    │                          │  Render() 루프    │
└────────────┘                          └─────────┬─────────┘
                                                  │
                                                  ▼
                                        ┌───────────────────┐
                                        │  Display (모니터)  │
                                        └───────────────────┘
```

## 주요 파일/디렉토리 위치

```
src/
├── EchoLoader/              ← 앱 진입점, DLL 로드 순서
├── EchoScanner/             ← Layer A: 스캔/취득 로직
│   ├── ESMain.cpp           ← ~10k줄 중심축 (AA 호출 7066~9954)
│   ├── AcquisitionAssistant/
│   │   ├── Manager.h/.cpp   ← SWE/UGAP 핸들러 팩토리+싱글톤
│   │   └── IAcqAssistHandler.h ← 인터페이스 (외부 컨트랙트)
│   └── ...
├── GcViewer/                ← Layer B: 렌더링
│   └── ObjectViewer/
│       ├── OVObject.cpp     ← UDT 팩토리 등록 (태그 → 타입)
│       ├── ObjectViewerImpl.cpp ← 런타임 인스턴스화
│       └── SWEAcquisitionAssistant.h/.cpp  ← Gc.SWEAcqAssist
├── ScCommon/                ← Layer 1: 공통 인프라
└── buildconfig/
    └── CppProjectBase.props ← Additional Include Directories (레이어 경계)
```

## Key Points

- 화살표 방향 = 의존 방향. **역방향 #include = 빌드 오류 또는 런타임 크래시**
- Layer A ↔ Layer B 통신은 **GC 파라미터 버스만** 허용 — 직접 `#include` 금지
- `EchoRoot`가 전체 패키지 수명주기를 관리 — 셧다운 순서가 RAII 소멸자 순서
- `ESMain.cpp`는 게이팅(InElasto/IsInUGAPMode) 후 위임 — 직접 로직 없음
- `buildconfig/CppProjectBase.props`의 Include Directories가 레이어 경계를 물리적으로 강제
