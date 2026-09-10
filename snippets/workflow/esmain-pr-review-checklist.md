---
title: "ESMain.cpp PR 리뷰 체크리스트"
category: workflow
tags: [esmain, pr-review, gating, aa, include, iec62304, traceability]
difficulty: intermediate
---

ESMain.cpp를 건드리는 PR을 리뷰할 때 순서대로 확인해야 할 항목 목록.

## 언제 이 아티클을 보나

- ESMain.cpp 변경이 포함된 PR의 리뷰어로 지정됐을 때
- Approve 전에 놓친 항목이 없는지 최종 점검할 때
- 주니어 개발자가 처음 ESMain을 건드리는 PR을 올렸을 때

## 리뷰 체크리스트

```
ESMain.cpp PR 리뷰 시 확인 항목

[ ] 1. 새 AA 호출 사이트에 게이팅 술어가 있는가?
    찾는 방법: diff에서 Manager::Instance() 검색
    → InElasto() 또는 IsInUGAPMode() 없으면 즉시 리젝
    → 공통 기능이라면 "두 개 별도 게이팅" 패턴인지 확인

[ ] 2. ESMain이 직접 로직을 갖게 됐는가?
    찾는 방법: diff에서 새 멤버 변수, 새 계산 로직 검색
    → "if (InElasto()) { [로직] }" 형태는 경고
    → "if (InElasto()) Manager::Instance(SWE)->[함수]()" 형태만 허용
    → 로직은 Manager/Handler 안으로

[ ] 3. 새 전역 변수나 정적 변수가 추가됐는가?
    → ESMain에 전역 상태 추가 = 테스트 불가, 재초기화 위험
    → Manager 내부 상태로 이동 요청

[ ] 4. 새 #include가 추가됐는가?
    → GcViewer/ 방향 include = 즉시 리젝 (역방향 의존)
    → 새 ScCommon/ include = 레이어 1이므로 허용

[ ] 5. 11개 기존 사이트 패턴이 일관되게 유지됐는가?
    찾는 방법: git diff | grep -A5 "Manager::Instance"
    → 새 사이트가 기존 패턴(gate → instance → delegate)과 다르면 통일 요청

[ ] 6. 기존 게이팅 로직이 수정됐는가?
    → InElasto() / IsInUGAPMode() 내부 수정은 매우 고위험
    → 두 함수가 배타적 조건을 유지하는지 확인
    → 모드 전환 타이밍 변경 시 전환 구간 분석 필요

[ ] 7. 티켓 ID 주석이 있는가?
    → // RBUG-XXXXX 또는 // FBUG-XXXXX 없으면 추적성 위반
    → IEC 62304 Class C: 모든 변경에 요구사항 추적 필수
```

## 즉시 리젝 사유

```
즉시 리젝             | 이유
---------------------|---------------------------
게이팅 없는 AA 호출   | 모드 오염 → 임상 데이터 오류
역방향 #include       | 빌드 순환 → 팀 전체 빌드 블로킹
새 ESMain 전역 상태   | 재초기화 불가, 테스트 불가
게이팅 조건 내부 수정  | 모든 모드 동작 검증 필요
```

## 리뷰 시 자주 쓰는 grep

```bash
# diff에서 게이팅 없는 Instance() 호출 빠르게 찾기
git diff HEAD~1 -- ESMain.cpp | grep "Manager::Instance"

# 새 #include 라인만 추출
git diff HEAD~1 -- ESMain.cpp | grep "^+#include"

# 기존 사이트 수 확인 (리뷰 전 baseline)
grep -c "Manager::Instance" ESMain.cpp

# 티켓 주석 확인
git diff HEAD~1 -- ESMain.cpp | grep "RBUG\|FBUG"
```

## 리뷰 코멘트 템플릿

게이팅 누락:
```
이 AA 호출에 게이팅 술어(InElasto() / IsInUGAPMode())가 없습니다.
게이팅 없이 Manager::Instance()를 호출하면 B모드 등 다른 모드에서도 실행됩니다.
[swe-ugap-dual-gating.md] 패턴 1 참고 부탁드립니다.
```

역방향 include:
```
GcViewer/ 헤더를 ESMain에서 직접 include하면 레이어 의존 방향이 역전됩니다.
필요한 타입은 ScCommon/ 또는 AcquisitionAssistant/ 경계 인터페이스를 통해 전달해야 합니다.
```

## Key Points

- 리뷰의 1번 항목(게이팅)이 가장 중요하다 — 임상 데이터 오류로 직결된다.
- ESMain은 게이트·위임 전용이므로 로직이 추가되면 그 자체가 구조 위반이다.
- 역방향 `#include`는 팀 전체 빌드를 블로킹하므로 즉시 리젝이 원칙이다.
- `InElasto()` / `IsInUGAPMode()` 내부를 수정하는 PR은 모든 모드 동작을 재검증해야 한다.
- IEC 62304 Class C 제품에서는 티켓 ID 주석 없는 변경은 추적성 위반이다.
