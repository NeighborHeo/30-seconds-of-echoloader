---
title: "Build Macros: _DEBUG vs NDEBUG"
category: cpp
tags: [build, debug, macros, msvc, safety]
difficulty: beginner
---

gipc-app은 `_DEBUG` / `NDEBUG` 표준 매크로만 사용한다. Debug 빌드에는 MFC 디버그 할당자(`DEBUG_NEW`)가 활성화된다.

## Pattern

```cpp
// StdAfx.h 또는 precompiled header 에서 설정
#ifdef _DEBUG
    #define new DEBUG_NEW   // MFC 디버그 할당자 — 메모리 누수 추적
#endif

// 코드 내 조건부 컴파일
#ifdef _DEBUG
    ScLogInfo("ACQASSIST", "Debug: state=%d frame=%d", m_state, m_frameCount);
    // 디버그 전용 검증 코드
    ValidateInternalConsistency();
#endif

// NDEBUG = Release (assert 비활성화)
// production 코드에 assert() 거의 없는 이유:
// 1. assert()는 NDEBUG에서 완전히 사라짐 — 릴리스 행동이 달라짐
// 2. 의료기기 코드에서 크래시 유발 가능 → 로깅으로 대체
#ifndef NDEBUG
    assert(ptr != nullptr);  // 새 코드에서는 회피
#endif
```

## RuntimeChecks (Debug 빌드)

```cpp
// CppProjectBase.props 설정:
// <RuntimeLibrary>MultiThreadedDebugDLL</RuntimeLibrary>
// <BasicRuntimeChecks>EnableFastChecks</BasicRuntimeChecks>
//
// EnableFastChecks = /RTC1:
//   - 스택 포인터 검증
//   - 로컬 변수 초기화 검사
//   - 버퍼 오버런 감지 (릴리스에서 성능 영향으로 제거됨)
```

## Key Points

- `_DEBUG` 는 MSVC가 설정; `NDEBUG` 는 C 표준 (`assert.h` 가 읽음) — 둘 다 사용됨
- `DEBUG_NEW` 는 `delete` 없이 종료 시 메모리 누수 위치를 리포트 → 디버깅 필수 도구
- **production에 `assert()` 추가 금지** — NDEBUG 시 사라지므로 릴리스 행동이 달라지고, 의료기기에서 예기치 않은 crach 유발
- 대신 `ScLogError` + `return false` 패턴으로 에러 상황을 처리
- `/RTC1` (EnableFastChecks)은 Debug 전용 — 잘못된 코드를 빠르게 잡아주지만 릴리스 성능에는 영향 없음
