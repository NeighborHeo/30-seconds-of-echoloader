---
title: "IEC 62304 소프트웨어 안전 등급과 실무 영향"
category: domain
tags: [iec-62304, safety-class, compliance, traceability, unit-test]
difficulty: beginner
---

소프트웨어 안전 등급(A/B/C)이 코딩 규칙, 테스트 요건, 빌드 설정까지 바꾼다.

## Why

gipc-app의 초음파 진단 기능은 IEC 62304 Class C(심각한 상해/사망 가능)에 해당한다.
등급을 모르고 코딩하면 `assert()` 제거, 로그 생략, 임의 코드 삭제가 모두 규제 위반이 된다.

## Pattern

```
IEC 62304 소프트웨어 안전 등급

┌─────────┬──────────────────────────────┬──────────────────────────────────────┐
│  등급   │  정의                        │  예시                                │
├─────────┼──────────────────────────────┼──────────────────────────────────────┤
│ Class A │  상해 없음                   │  사용자 인터페이스 테마, 도움말 텍스트│
│ Class B │  비심각 상해                 │  PDF 내보내기, 원격 서비스 연결       │
│ Class C │  심각 상해 또는 사망 가능    │  B-mode 영상 측정, 진단 수치 계산    │
└─────────┴──────────────────────────────┴──────────────────────────────────────┘

gipc-app 진단 측정 기능 → Class C
```

```
Class C 요구사항이 실무에서 의미하는 것:

1. 추적 가능성 (Traceability)
   요구사항 → 설계 → 코드 → 테스트 가 문서로 연결되어야 함
   코드 한 줄을 삭제해도 "왜?"를 설명하는 변경 이력이 필요

2. 변경 관리 (Change Control)
   모든 코드 수정 = 공식 변경 요청(CR) + 영향 분석 + 재검증
   "작은 버그픽스"도 예외 없음

3. 단위 테스트 요건
   Class A: 불필요
   Class B: 소프트웨어 단위 시험 (선택적)
   Class C: 소프트웨어 단위 시험 필수
             → 모든 측정 알고리즘 함수에 단위 테스트 존재해야 함

4. assert() 와 릴리스 빌드
   #ifdef NDEBUG로 assert()가 제거되면
   = 런타임 안전 검사 제거
   = Class C 제품에서 감사(audit) 시 지적 대상
   → Release 빌드에서도 유지되는 별도 런타임 검사 필요
```

```cpp
// 잘못된 패턴 — Release 빌드에서 검사 사라짐
void ComputeEF(float fEDV, float fESV)
{
    assert(fEDV > 0.0f);  // NDEBUG 시 제거됨!
    float fEF = (fEDV - fESV) / fEDV * 100.f;
    // ...
}

// 올바른 패턴 — 등급 C 코드에서 런타임 검사 유지
void ComputeEF(float fEDV, float fESV)
{
    if (fEDV <= 0.0f) {
        // 입력 오류 → 계산 중단, 오류 코드 반환
        GcLog::Error("ComputeEF: invalid EDV=%.2f", fEDV);
        return;  // 또는 예외/오류 코드 반환
    }
    float fEF = (fEDV - fESV) / fEDV * 100.f;
    // ...
}
```

## Key Points

- **등급은 기능 단위로 결정됨**: 한 실행 파일 안에서도 기능마다 등급이 다를 수 있음. 진단 측정 = Class C, 사용자 설정 저장 = Class A
- **Class C 코드 삭제는 변경 이력 필요**: `git blame`이 규제 감사의 증거가 됨 — 커밋 메시지에 이유를 항상 기재
- `assert()` 대신 방어적 `if` + 로그 + 안전 반환값 패턴을 Class C 경로에 일관 적용
- 단위 테스트 파일이 없는 측정 함수 = 불완전한 Class C 구현 → FDA/CE 감사 시 Non-Conformance(NC) 발견
- IEC 62304 § 5.5 (소프트웨어 단위 구현)와 § 5.6 (소프트웨어 단위 검증)이 직접 적용되는 조항
