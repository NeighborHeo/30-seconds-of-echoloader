---
title: "analysis/ Directory and Design Doc §References"
category: architecture
tags: [documentation, traceability, design-docs, phase-markers, governance]
difficulty: intermediate
---

`analysis/` 디렉토리는 리팩토링 설계 문서의 저장소다. 코드 주석의 `§참조` 가 이 문서의 특정 섹션을 가리킨다.

## 디렉토리 구조

```
gipc-app/
└── analysis/
    └── SWEUGAPAcqAssist/
        ├── 14_poorprobe_reuse_design.md     ← 기존 설계 문서
        │    §2: PoorProbeContact 재사용 설계
        │    §6: 추출 단계별 계획
        └── 15_phaseB4_b6_plan.md           ← 신규 작성 예정
             §4: UGAP preset 문자열 검증 계획
             §?: P2+P4 atomic PR 4-commit 분할
```

## §참조 패턴 (코드 ↔ 문서 연결)

```cpp
// Phase 마커 + §참조 — 코드에서 설계 문서로 이동 가능
// Phase A.3, see analysis/SWEUGAPAcqAssist/14_poorprobe_reuse_design.md §2 / §6
void LiverWarning3Panel::Init(const std::string& presetName)
{
    // §2: preset 기반 초기화 로직
    // §6: Base와의 라이프타임 동기화
}

// B.1 단계 구현 주석
// Phase B.1 — mirror 3-warning flags into LiverWarning3Panel.
// see §6 for mirror consistency requirements
bool m_bLargeSCDWarning         = false;
bool m_bPoorProbeContactWarning = false;
bool m_bObliqueCapsuleWarning   = false;
```

## 문서 작성 규칙

```markdown
<!-- analysis/SWEUGAPAcqAssist/15_phaseB4_b6_plan.md -->

# Phase B.4~B.6 실행 계획

## §1. 배경
...

## §2. P-INIT: Init() 파라미터화
...

## §3. P2+P4: Authority Flip
...

## §4. UGAP Preset 문자열 검증  ← Q2 Director 확인 항목
현재 하드코딩: "ShearTrackAlg"
확인 담당: UGAP 팀
검증 방법: ...
```

## Key Points

- **리팩토링은 Phase 연속성 + 문서 §참조 갱신 의무** — 코드만 바꾸고 문서 미갱신 = 추적성 위반
- 새 Phase 진입 시 `analysis/` 에 신규 문서 생성 또는 기존 §추가
- 코드 주석의 `§N` 은 문서의 `## §N` 섹션과 1:1 대응 — 번호 불일치 금지
- `14_` 접두어는 순서 번호 — 새 문서는 `15_`, `16_` 순으로 추가
- 이 문서들은 IEC 62304 §5.5 (소프트웨어 유닛 설계) 아티팩트 역할
