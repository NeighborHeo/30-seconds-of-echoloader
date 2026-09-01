---
title: "Audit Log User Event"
category: domain
tags: [security, audit-log, compliance, fda, traceability]
difficulty: intermediate
---

보안/임상적으로 중요한 사용자 행동은 `AuditLogUserEvent()` 로 감사 로그에 기록한다. FDA 규제 추적성 요건의 핵심 구현체다.

## Pattern

```cpp
#include <ScCommon/AuditLog.h>   // 추정

// 사용자 로그인 이벤트
AuditLogUserEvent(
    AUDIT_LOG,           // 이벤트 심각도: AUDIT_LOG, SECURITY_ALERT
    "USER_AUTH",         // 모듈 키
    "User '%s' logged in from %s",
    userName.c_str(),
    ipAddress.c_str()
);

// 보안 경고 — 무단 접근 시도
AuditLogUserEvent(
    SECURITY_ALERT,
    "USER_AUTH",
    "Unauthorized remote streaming attempt blocked for user '%s'",
    userName.c_str()
);

// 임상 설정 변경
AuditLogUserEvent(
    AUDIT_LOG,
    "PRESET_CHANGE",
    "Preset changed from '%s' to '%s' by operator",
    oldPreset.c_str(),
    newPreset.c_str()
);
```

## 일반 로그와 감사 로그의 차이

```
ScLogInfo("KEY", "...")    → 진단/디버그 목적, 필요 시 덮어쓰기 가능
ScLogError("KEY", "...")   → 오류 추적, 순환 버퍼

AuditLogUserEvent(...)     → 규제 목적, 삭제 불가, 외부 감사 가능
                             FDA 21 CFR Part 11: 전자 서명/기록 규정 대응
```

## Key Points

- `AUDIT_LOG` vs `SECURITY_ALERT` 는 심각도 레벨 — 후자는 규제 보고 대상
- 새 코드에서 보안 관련 로직(인증, 권한 변경, 원격 접속)에는 반드시 `AuditLogUserEvent` 추가
- 감사 로그는 덮어쓰거나 삭제할 수 없음 — 스토리지 용량 계획 시 고려
- 일반 `ScLogInfo`/`ScLogError` 로 감사 로그를 대체하면 FDA 규정 위반
- 코드 리뷰 시 신규 인증/권한 로직에 `AuditLogUserEvent` 누락이 없는지 확인
