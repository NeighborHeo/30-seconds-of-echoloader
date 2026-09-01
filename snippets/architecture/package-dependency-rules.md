---
title: "패키지 의존 방향 규칙"
category: architecture
tags: [dependencies, packages, layering, coupling, architecture]
difficulty: intermediate
---

gipc-app의 ~50개 패키지는 허용된 의존 방향이 있다. 역방향 의존은 빌드 순환 또는 런타임 크래시로 이어진다.

## 허용된 의존 방향

```
허용 방향: 위 → 아래 (상위 레이어 → 하위 레이어)

┌──────────────────────────────────────────────────────────┐
│ Layer 4: Application / UI                                 │
│   EchoLoader, EchoFrontPanel, EchoWorksheet              │
│   EchoDecisionSupport, EchoConfig                        │
└──────────────────────────┬───────────────────────────────┘
                           │ 의존 ↓
┌──────────────────────────▼───────────────────────────────┐
│ Layer 3: Domain Logic                                     │
│   EchoScanner (→ AcquisitionAssistant)                   │
│   EchoMeasure, EchoSysMon                                │
└──────────────────────────┬───────────────────────────────┘
                           │ 의존 ↓
┌──────────────────────────▼───────────────────────────────┐
│ Layer 2: Display / Rendering                              │
│   GcViewer / ObjectViewer (OVObject 계층)                │
│   EchoRoot (시스템 루트)                                  │
└──────────────────────────┬───────────────────────────────┘
                           │ 의존 ↓
┌──────────────────────────▼───────────────────────────────┐
│ Layer 1: Infrastructure                                   │
│   ScCommon, ScLogsDatabase, mcd                          │
│   target/include/ (Eigen, 3rd-party)                     │
└──────────────────────────────────────────────────────────┘
```

## 금지된 역방향 의존

```cpp
// ❌ Layer 1(ScCommon)이 Layer 3(EchoScanner)을 알면 안 됨
// ScCommon/Thread.h:
#include "EchoScanner/AcquisitionAssistant.h"  // ❌ 역방향 의존

// ❌ Layer 2(GcViewer)가 Layer 3(EchoScanner)를 직접 포함하면 안 됨
// GcViewer/ObjectViewer/SWEAcquisitionAssistant.h:
#include "EchoScanner/ESMain.h"  // ❌ 역방향 의존
// → GC 파라미터 버스를 통해 간접 통신해야 함 (외부 컨트랙트)
```

## 실제 확인된 예외 (허용)

```cpp
// 검증된 발견: AA 클래스는 ObjectViewer 내부에 캡슐화
// EchoMeasure, EchoWorksheet, EchoScanner가 GcViewer를 직접 import하지 않음
// → Manager facade가 디커플링 역할

// 허용 패턴: ESMain → Manager → IAcqAssistHandler (Layer A 내부)
// 허용 패턴: GcViewer → ScCommon (Layer 2 → Layer 1)
// 금지 패턴: GcViewer → ESMain (Layer 2 → Layer 3)
```

## 의존 추가 전 체크리스트

```
새 #include를 추가하려 할 때:
[ ] 포함하는 파일의 레이어는?
[ ] 포함되는 헤더의 레이어는?
[ ] 방향이 위 → 아래인가? (하위 레이어 포함)
[ ] 같은 레이어 내 포함인가? (허용)
[ ] 역방향이라면: GC 파라미터 버스나 인터페이스로 해결 가능한가?

빌드 순환 의심 시:
  grep -r "include.*EchoScanner" src/GcViewer/
  → 결과 있으면 역방향 의존 존재 → 즉시 제거
```

## Key Points

- **AA 클래스가 ObjectViewer에 캡슐화** 됨 — EchoMeasure/EchoWorksheet/EchoScanner가 GcViewer를 직접 알지 않음 (실측 확인)
- GC 파라미터 버스가 역방향 의존을 끊는 firewall — 레이어 분리의 핵심 이유
- `.vscode/settings.json` 의 `/W4 /permissive-` 가 일부 역방향 의존을 빌드 오류로 잡아줌
- CI 없음 → 빌드 에러가 유일한 자동 방어선 — 코드리뷰에서 include 방향 수동 확인 필수
- `src/buildconfig/CppProjectBase.props` 의 Additional Include Directories가 레이어 경계의 실제 구현
