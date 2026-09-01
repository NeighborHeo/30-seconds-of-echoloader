---
title: "보안 이벤트 감사 로그 패턴"
category: workflow
tags: [security, audit-log, FDA, IEC62304, compliance]
difficulty: intermediate
---

보안 관련 변경(인증, 권한, 사용자 이벤트)은 반드시 `AuditLogUserEvent()`로 별도 채널에 기록한다.

## 절차

1. **일반 로깅**(`ScLogInfo` / `ScLogError`)과 **감사 로그**(`AuditLogUserEvent`)는 채널이 다름
2. 아래 상황에서 `AuditLogUserEvent()` 호출 **의무**:
   - 사용자 인증 성공/실패
   - 접근 권한 변경
   - 민감한 설정 변경
   - 사용자 트리거 임상 이벤트
3. 매크로: `AUDIT_LOG(event)`, `SECURITY_ALERT(event)` — 내부적으로 `AuditLogUserEvent` 래핑
4. FDA 21 CFR Part 11 / IEC 62304 감사 추적 요건 충족 목적

## 예시

```cpp
// 인증 이벤트 — 감사 로그 필수
bool AuthManager::Login(const UserId& id, const Password& pw)
{
    const bool ok = ValidateCredentials(id, pw);
    if (ok)
        AUDIT_LOG("USER_LOGIN_SUCCESS: " + id.ToString());
    else
        SECURITY_ALERT("USER_LOGIN_FAILURE: " + id.ToString());

    return ok;
}

// 권한 변경 — 감사 로그 필수
void AccessControl::GrantRole(const UserId& id, Role role)
{
    m_roles[id] = role;
    AUDIT_LOG("ROLE_GRANTED: user=" + id.ToString()
              + " role=" + RoleToString(role));
}
```

## 체크리스트

- [ ] 인증/권한/설정 변경 코드 경로에 `AUDIT_LOG` 또는 `SECURITY_ALERT` 추가
- [ ] 일반 `ScLogInfo`/`ScLogError` 로만 처리하지 않음
- [ ] 감사 이벤트 메시지에 UserId / 변경 내용 포함
- [ ] FDA / IEC 62304 감사 추적 요건 검토
- [ ] 감사 로그 저장소 경로 및 보존 기간 정책 확인
