---
title: "EchoSysmon — 런타임 시스템 모니터링"
category: packages
tags: [monitoring, BSOD, crash-detection, health, OS-event-log]
difficulty: intermediate
---

런타임 중 시스템 상태를 지속 감시. OS 이벤트 로그에서 BSOD(BugcheckCode)를 감지하고 이전 비정상 종료 여부를 보고한다. EchoRoot(부트스트랩)와 역할이 명확히 분리된다.

## 역할

- Windows 이벤트 로그에서 BSOD 이벤트 감지
  - 로그 메시지 예: `"Previous Unclean Shutdown: BugcheckCode=0x0000007E"`
- 이전 세션 비정상 종료 여부를 애플리케이션에 보고
- 런타임 중 주기적 헬스 체크 수행

## 위치

```
src/packages/EchoSysmon/
├── EchoSysmonPck.h/.cpp     ← 패키지 초기화, 헬스 체크 루프
├── OSEventLogReader.h/.cpp  ← Windows Event Log API 래퍼
└── UncleanShutdownDetector.h/.cpp  ← BSOD/비정상 종료 판정 로직
```

## 핵심 클래스

| 클래스 | 역할 |
|---|---|
| `EchoSysmonPck` | 패키지 초기화 및 모니터링 루프 관리 |
| `OSEventLogReader` | Windows Event Log에서 BSOD 이벤트 추출 |
| `UncleanShutdownDetector` | BugcheckCode 파싱 및 비정상 종료 판정 |

## 의존 관계

```
EchoSysmon
  → OS (Windows Event Log API)
  → EchoRoot  (크래시 덤프 경로 공유)
  ← EchoLoader (런타임 초기화 후 시작)
```

## EchoRoot vs EchoSysmon 역할 구분

| | EchoRoot | EchoSysmon |
|---|---|---|
| 타이밍 | 부팅 시 1회 | 런타임 내내 |
| 역할 | 워치독 피드, 치명 오류 처리 | BSOD 감지, 헬스 체크 |
| 종류 | 반응형(fatal 발생 시) | 능동형(주기적 폴링) |

## 주의사항

- `BugcheckCode` 파싱은 Windows 버전별 이벤트 ID가 다를 수 있음 — OS 버전 확인 필수
- 헬스 체크 폴링 주기가 너무 짧으면 이벤트 로그 I/O 부하 발생 — 기본값 검토
- EchoRoot의 `UpdatePreviousBootCrashDump()`와 데이터 중복 방지: 각자 담당 영역 명확히 유지
