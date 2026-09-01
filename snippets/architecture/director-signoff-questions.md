---
title: "5 Director Sign-Off Questions Before P2+P4"
category: architecture
tags: [process, refactoring, governance, p2-p4, decision]
difficulty: advanced
---

Warning authority flip(P2+P4)에 진입하기 전 Director(또는 담당 책임자)의 확인이 필요한 5가지 열린 질문이다.

## 5가지 질문

```
Q1. GE 측 또는 다운스트림이 ShouldShowGuidelinesGray()를 호출하는가?
    → 이 메서드가 외부에서 호출된다면 고아 오버라이드가 아님
    → 확인 방법: grep 전체 솔루션, 외부 COM 인터페이스 확인
    → 미확인 시 리스크: override 제거 후 다운스트림 silent failure

Q2. UGAP의 실제 preset 문자열은 무엇인가?
    → 코드에 "ShearTrackAlg" 하드코딩 발견 — 검증 필요
    → 확인 방법: UGAP 담당 팀 + 설정 파일 검토
    → 미확인 시 리스크: Init(presetName)에 잘못된 문자열 → 무음 실패

Q3. s_*LastState의 SWE/UGAP 의미가 동일한가?
    → 두 핸들러가 같은 정적 변수를 공유하는지, 독립 변수인지
    → 확인 방법: AcquisitionAssistant.cpp 정적 변수 추적
    → 미확인 시 리스크: 모드 전환 시 상태 오염

Q4. 빌드타임 #ifdef 롤백이 릴리스 관리자에게 수용되는가?
    → P2+P4 atomic PR에 롤백 플래그를 넣을 계획
    → 확인 방법: 릴리스 브랜치 정책 + QA 프로세스 검토
    → 미확인 시 리스크: 롤백 불가 상태로 불안정한 빌드 릴리스

Q5. 임상 루프 녹화본 확보가 가능한가?
    → P2+P4 전후 동일 임상 시나리오 재생으로 회귀 검증 필요
    → 확인 방법: QA 팀 + 임상 녹화 인프라 확인
    → 미확인 시 리스크: 렌더링 회귀를 자동 테스트가 잡지 못함
```

## 의사결정 매트릭스

```
Q1 미확인 → P1-spike 먼저 (≤1일)
Q2 미확인 → UGAP 담당에게 티켓 발급, P-INIT spike 블로킹
Q3 미확인 → AcquisitionAssistant.cpp 정적 변수 grep, 1~2시간
Q4 미확인 → 릴리스 관리자 미팅, 빌드 정책 문서 검토
Q5 미확인 → QA 팀 미팅, 녹화 환경 구성 계획 수립
```

## Key Points

- 5개 질문 중 하나라도 열려 있으면 P2+P4 진입 금지 — "닫아야 할 질문"
- Q1(ShouldShowGuidelinesGray)은 P1-spike로 ≤1일 안에 답 가능
- Q2(preset 문자열)는 외부 팀 의존 — 가장 오래 걸릴 수 있음
- 각 질문은 별도 FBUG/RBUG 티켓을 발급해 추적할 것
- Director sign-off = 구두 확인 아님 — 티켓 코멘트 또는 설계문서 §업데이트로 기록
