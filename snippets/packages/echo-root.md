---
title: "EchoRoot — 시스템 부트스트랩 & 워치독"
category: packages
tags: [bootstrap, watchdog, crash-dump, fatal-error]
difficulty: intermediate
---

애플리케이션 최초 부팅 단계를 담당. 워치독 콜백과 치명 오류 핸들러, 이전 크래시 덤프 처리가 모두 여기에 있다.

## 역할

- `FatalErrorHandler()` — 복구 불가능한 오류 발생 시 안전 종료 수행
- `WatchdogCB()` — OS 레벨 워치독 타이머 피드; 타임아웃 시 강제 재시작
- `UpdatePreviousBootCrashDump()` — 이전 세션 크래시 덤프를 수집·저장

## 위치

```
src/packages/EchoRoot/
├── EchoRootPck.h        ← FatalErrorHandler, WatchdogCB 선언
├── EchoRootPck.cpp      ← 위 함수 구현
└── CrashDumpHelper.cpp  ← UpdatePreviousBootCrashDump 구현
```

## 핵심 클래스 / 함수

| 심볼 | 역할 |
|---|---|
| `FatalErrorHandler()` | SEH/terminate 훅; 덤프 저장 후 프로세스 종료 |
| `WatchdogCB()` | 하드웨어 워치독 피드 콜백 |
| `UpdatePreviousBootCrashDump()` | 부팅 시 이전 비정상 종료 기록 갱신 |

## 의존 관계

```
EchoRoot
  → (OS API, MSVC SEH)
  ← EchoLoader  (최초 로드)
  ← EchoSysmon  (런타임 헬스 데이터 전달)
```

## 주의사항

- `FatalErrorHandler`는 스택이 오염된 상태에서 호출될 수 있음 — 내부에서 동적 할당 금지
- `WatchdogCB` 주기 변경 시 하드웨어 타이머 설정과 반드시 동기화
- EchoRoot = **부트스트랩** 전용; 런타임 헬스 모니터링은 EchoSysmon이 담당 (역할 혼동 주의)
