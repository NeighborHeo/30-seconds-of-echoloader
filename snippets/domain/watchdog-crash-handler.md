---
title: "Watchdog & Fatal Error Handler"
category: domain
tags: [safety, watchdog, crash, fault-tolerance, medical-device]
difficulty: advanced
---

`FatalErrorHandler()` 와 `WatchdogCB()` 는 치명적 시스템 오류를 처리하는 마지막 방어선이다. 부팅 시 이전 크래시를 감지하고 복구 경로를 결정한다.

## Pattern

```cpp
// EchoRootPck.h (추정 시그니처)
void FatalErrorHandler(const char* moduleName, const char* reason);
void WatchdogCB();       // 워치독 타이머 콜백 — 응답 없으면 재시작

// 부팅 시 이전 크래시 확인
void UpdatePreviousBootCrashDump()
{
    // OS 이벤트 로그에서 BSOD 정보 읽기
    // "Previous Unclean Shutdown: BugcheckCode=0x..." 패턴 파싱
    std::string crashInfo = ReadBsodEventLog();
    if (!crashInfo.empty())
    {
        ScLogError("BOOTCHECK",
            "Previous Unclean Shutdown detected: %s", crashInfo.c_str());

        // RBUG159787: 하드웨어 부팅 후 재부팅 카운터 클리어
        // "If we booted up with HW, make sure we clear the reboot retry counter"
        if (IsHardwarePresent())
        {
            ClearRebootRetryCounter();
        }
    }
}
```

## 에스컬레이션 계층

```
정상 에러:   bool + errorMsg → ScLogError → 호출자에게 전파
심각한 에러: FatalErrorHandler() → 시스템 리셋 또는 안전 상태 진입
워치독 타임아웃: WatchdogCB() → 강제 재시작 (하드웨어 watchdog 연동)
크래시 덤프:  UpdatePreviousBootCrashDump() → 다음 부팅에서 분석
```

## IEC 62304 관련성

```
IEC 62304 §6.2.5 (Software Integration Testing):
  시스템이 비정상 종료 후 재기동 시 안전 상태를 복원해야 함

→ UpdatePreviousBootCrashDump() 가 이 요구사항의 구현체
→ 재부팅 카운터(RBUG159787)는 무한 재부팅 루프 방지용
```

## Key Points

- `FatalErrorHandler` 는 마지막 방어선 — 여기서 예외를 던지거나 로직을 추가하면 안 됨
- 워치독은 하드웨어 레벨에서도 동작 — 소프트웨어 워치독만 믿으면 안 됨
- 부팅 경로 변경 시 `UpdatePreviousBootCrashDump` 영향 반드시 검토
- BSOD 이벤트 로그 파싱은 OS 버전 종속 — Windows 업그레이드 시 회귀 테스트 필수
- 재부팅 카운터 클리어(RBUG159787)는 HW 존재 여부 조건부 — 시뮬레이터 모드에서 오동작 방지
