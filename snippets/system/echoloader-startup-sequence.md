---
title: "EchoLoader 시작 시퀀스 다이어그램"
category: system
tags: [startup, initialization, dll-load, echoloader, sequence]
difficulty: intermediate
---

gipc-app 부팅 시 DLL과 패키지가 초기화되는 순서. 순서 위반 = 정적 초기화 순서 문제(SIOF) 또는 크래시.

## 시작 시퀀스

```
OS 프로세스 시작
        │
        ▼
┌───────────────────────────────────────────────────────────┐
│ Phase 1: 정적 초기화 (DLL 로드 순서 의존)                   │
│                                                           │
│  ScCommon.dll 로드  →  ScLogsDatabase.dll 로드            │
│  mcd.dll 로드                                             │
│         │                                                 │
│         ▼                                                 │
│  GcViewer.dll 로드  →  OVObject 팩토리 등록               │
│         │              ("Gc.SWEAcqAssist" 등 태그 등록)   │
│         ▼                                                 │
│  EchoScanner.dll 로드                                     │
│         │                                                 │
│         ▼                                                 │
│  EchoLoader.exe WinMain()                                 │
└───────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────┐
│ Phase 2: EchoRoot 초기화                                   │
│                                                           │
│  EchoRoot::Initialize()                                   │
│    ├── ScCommon 스레드 풀 초기화                           │
│    ├── ScLogsDatabase 연결                                │
│    ├── GrandCentral 프레임워크 시작                        │
│    │     ObjectViewerImpl 인스턴스화                       │
│    │     UDT 그래프 구성                                   │
│    └── EchoScanner ESMain::Initialize()                   │
│          ├── AcqAssist Manager 생성 (SWE + UGAP)         │
│          └── 파라미터 버스 구독 등록                       │
└───────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────┐
│ Phase 3: UI 패키지 초기화                                  │
│                                                           │
│  EchoFrontPanel::Initialize()   ← 버튼/노브 이벤트 연결   │
│  EchoWorksheet::Initialize()    ← 측정 UI                 │
│  EchoConfig::Initialize()       ← 설정/프리셋 로드        │
│  EchoDecisionSupport::Init()    ← AI 지원 기능            │
└───────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────┐
│ Phase 4: 메인 루프                                         │
│                                                           │
│  while (running) {                                        │
│    MFC 메시지 펌프 (Windows 메시지 처리)                   │
│    │                                                      │
│    ├── 사용자 입력 → ESMain 라우팅                         │
│    └── 프레임 수신 → OVObject Render() 루프               │
│  }                                                        │
└───────────────────────────────────────────────────────────┘
        │
        ▼
┌───────────────────────────────────────────────────────────┐
│ Shutdown (역순 소멸)                                       │
│                                                           │
│  UI 패키지 소멸  →  ESMain::Shutdown()                    │
│  GrandCentral 소멸  →  OVObject OnDeactivate() 호출       │
│  ScCommon 스레드 풀 종료  →  EchoRoot 소멸               │
└───────────────────────────────────────────────────────────┘
```

## 정적 초기화 순서 문제 (SIOF) 방지

```cpp
// ❌ 위험: 정적 객체가 다른 DLL의 정적 객체에 의존
// ScCommon.dll
static GlobalLogger g_logger;  // ScLogsDatabase보다 먼저 초기화될 수 있음

// ✅ 안전: Construct-on-first-use idiom
static GlobalLogger& GetLogger()
{
    static GlobalLogger s_logger;  // 첫 호출 시 초기화
    return s_logger;
}
```

## Key Points

- DLL 로드 순서는 `EchoLoader`의 링크 순서와 `LoadLibrary` 호출 순서로 결정
- Layer 1(ScCommon) → Layer 2(GcViewer) → Layer 3(EchoScanner) 순으로 로드해야 의존성 충족
- 셧다운은 **시작 역순** — RAII 소멸자 순서가 곧 셧다운 안전성
- `ESMain::Initialize()` 이전에 `Manager` 사용 = null 역참조 → 부팅 크래시
- 정적 싱글톤은 construct-on-first-use로 SIOF 회피 (`static Local` 패턴)
