---
title: "AuditLog 의무 vs ScLog 선택: 판단 기준"
category: domain
tags: [auditlog, sclog, fda, iec62304, compliance, logging]
difficulty: intermediate
---

새 사용자 액션 핸들러를 작성할 때 `AuditLogUserEvent()`를 써야 하는지 `ScLogInfo()`로 충분한지 판단하는 기준.

## 언제 이 아티클을 보나

- 새 기능의 사용자 액션 핸들러를 작성하면서 로깅 방식을 결정해야 한다
- 코드 리뷰에서 "AuditLog 써야 하는 거 아닌가요?" 질문을 받았다
- FDA 21 CFR Part 11 또는 IEC 62304 감사 대비 로그 정책을 점검한다

## AuditLog가 의무인 경우 (FDA 21 CFR Part 11 / IEC 62304)

```
다음에 해당하면 AuditLogUserEvent(AUDIT_LOG, ...) 필수:

[ ] 임상 측정값을 생성, 수정, 삭제하는 사용자 행위
    예: 워크시트 저장, 측정값 수동 입력, 보고서 출력

[ ] 환자 데이터에 접근하거나 변경하는 행위
    예: 환자 선택, DICOM 전송, 기록 열기

[ ] 시스템 설정을 변경하는 행위
    예: 프리셋 저장, 네트워크 설정 변경, 보정(calibration) 수행

[ ] 보안 관련 행위
    예: 로그인, 로그아웃, 권한 변경 → AuditLogUserEvent(SECURITY_ALERT, ...)
```

## ScLog만 써도 되는 경우

```
다음에 해당하면 ScLogInfo/Warn/Error로 충분:

[ ] 개발자/진단 목적 정보 (환자나 임상 결과와 무관)
    예: "SWE 모드 진입", "프레임 처리 시간 18ms", "렌더 루프 시작"

[ ] 내부 상태 변화 (사용자가 직접 트리거하지 않은)
    예: "GC 파라미터 수신: State=2", "OnActivate 호출"

[ ] 오류/경고 진단
    예: "SetParameter 타입 불일치", "프레임 오버런 감지"
```

## 빠른 판단 질문

```
Q: 이 이벤트가 나중에 "언제, 누가, 무엇을 했는가"를
   FDA 감사관에게 설명해야 할 때 필요한가?
   → YES: AuditLog
   → NO: ScLog

Q: 환자 기록이나 임상 데이터와 연결되는가?
   → YES: AuditLog

Q: 코드 디버깅 목적으로만 필요한가?
   → YES: ScLog (운영 빌드에서는 억제될 수 있음)
```

## 잘못된 사용 예

```cpp
// ❌ 임상 이벤트에 ScLog만 사용
void EchoWorksheet::SaveMeasurement(const Measurement& m)
{
    ScLogInfo("WORKSHEET", "측정값 저장: %.2f %s", m.value, m.unit);
    // → 감사 추적 없음 → FDA 감사 시 문제
}

// ✅ 올바른 사용
void EchoWorksheet::SaveMeasurement(const Measurement& m)
{
    AuditLogUserEvent(AUDIT_LOG, "WORKSHEET",
                      "측정값 저장: patient=%s value=%.2f unit=%s",
                      m.patientId.c_str(), m.value, m.unit);
    ScLogInfo("WORKSHEET", "측정값 저장 완료 (내부 진단)");
}
```

두 로그는 목적이 다르다. `AuditLogUserEvent`는 규제 추적성, `ScLogInfo`는 개발자 진단. 동시에 쓰는 것이 정상이다.

## Key Points

- `AuditLogUserEvent()`의 의무 기준은 "사용자가 직접 트리거했고, 임상/환자/보안 데이터에 영향을 미치는가"다 — 내부 상태 변화나 시스템 이벤트는 해당하지 않는다
- 보안 행위(로그인, 로그아웃, 권한 변경)는 `AUDIT_LOG`가 아닌 `SECURITY_ALERT` 카테고리를 사용한다 — 카테고리 혼용은 감사 필터링을 깨뜨린다
- `AuditLogUserEvent`와 `ScLogInfo`는 대체 관계가 아니다 — 임상 이벤트에서는 두 가지를 함께 쓴다: AuditLog는 규제 추적, ScLog는 진단 목적
- `ScLogInfo`는 운영 빌드에서 억제(suppress)될 수 있다 — 임상 이벤트를 ScLog로만 기록하면 운영 환경에서 추적 자체가 사라진다
- 판단이 애매할 때의 기본값은 AuditLog다 — 과도한 AuditLog는 용량 문제, 누락된 AuditLog는 규정 위반이며 후자가 더 심각하다
