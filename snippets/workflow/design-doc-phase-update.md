---
title: "리팩토링 Phase 설계문서 갱신"
category: workflow
tags: [refactoring, phase, SWE, UGAP, documentation]
difficulty: advanced
---

Phase 문서 → 코드 주석 `§` 참조 → PR 커밋 접두사 순서로 추적성을 확보한다.

## 절차

1. **문서 추가**: `analysis/SWEUGAPAcqAssist/` 하위에 새 파일
   - 파일명 예: `15_phaseB4_b6_plan.md`
   - 내용: 목표, 변경 범위, RBUG/FBUG/CR 티켓 번호 목록

2. **코드 주석에 `§` 참조 삽입**
   ```cpp
   // Phase B.4, see analysis/SWEUGAPAcqAssist/15_phaseB4_b6_plan.md §3
   ```

3. **Phase 마커 형식** (변경 지점 상단)
   ```cpp
   // Phase A.3 — SWE 핸들러 분리, Manager 직접 호출 제거
   ```

4. **티켓 번호 기록**: Phase 문서 내 `## Tickets` 섹션에 RBUG/FBUG/CR 번호 나열

5. **PR 커밋 메시지 접두사**
   ```
   [B.4-001] SWE: Manager 호출 경로 단순화
   ```

## 예시

```markdown
<!-- analysis/SWEUGAPAcqAssist/15_phaseB4_b6_plan.md -->
# Phase B.4 / B.6 계획

## 목표
SWE/UGAP Manager 인터페이스 통합

## §3 변경 범위
- ESMain.cpp: Handler 분기 제거
- OVObject 파생: GetTag() 반환값 통일

## Tickets
- RBUG-4521
- CR-8801
```

## 체크리스트

- [ ] `analysis/SWEUGAPAcqAssist/` 에 Phase 문서 추가
- [ ] 코드 변경 지점에 `// Phase X.Y — 설명` 마커
- [ ] `§` 번호로 문서 섹션 참조
- [ ] Phase 문서에 RBUG/FBUG/CR 티켓 번호 기록
- [ ] PR 커밋 메시지에 `[B.4-xxx]` 접두사
