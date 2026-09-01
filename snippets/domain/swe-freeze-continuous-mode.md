---
title: "SWE: Freeze vs Continuous Mode"
category: domain
tags: [swe, freeze, continuous, autopos, state-machine]
difficulty: intermediate
---

SWE는 Freeze와 Continuous 두 가지 작동 모드를 갖는다. AutoPos 키 동작이 모드에 따라 정반대로 동작한다.

## Mode Behavior

```cpp
// SWEAcqAssistHandler::ResolveAutoPosKey()
std::string SWEAcqAssistHandler::ResolveAutoPosKey(AcqAssistState state)
{
    switch (state)
    {
    case AcqAssistState::Freeze:
        // Freeze 중에는 재트리거 억제 — 빈 문자열 반환
        return "";

    case AcqAssistState::Continuous:
        // Continuous 토글 — 다음 측정 사이클 시작
        return "SWEAcqAssistToggleContinuous";

    default:
        return "";
    }
}
```

## 측정 흐름

```
[Scanning]
    │
    ▼ 사용자가 Freeze 버튼 누름
[Freeze]
    │  ─ AutoPos 키 = ""  (억제)
    │  ─ ROI 고정, 재측정 불가
    │  ─ m_currentFrame_BS 잠김
    │
    ▼ 사용자가 다시 Freeze 해제
[Continuous]
    │  ─ AutoPos 키 = "SWEAcqAssistToggleContinuous"
    │  ─ 다음 push pulse 준비
    │
    ▼ 자동으로 다음 측정 사이클
[Scanning]
```

## UGAP와 비교

```
SWE Freeze:    AutoPos → ""         (억제)
SWE Continuous: AutoPos → "SWEAcqAssistToggleContinuous"

UGAP Measure:  AutoPos → "UGAPAcqAssistAnalyze"
               (Measure 모드 진입 시 즉시 분석 키 발행)
```

## Key Points

- Freeze 상태에서 AutoPos `""` 반환은 의도적 — 빈 문자열이 "아무것도 하지 않음"을 의미
- ROI는 Freeze에서 고정됨 — 재배치하려면 반드시 Unfreeze 필요
- `AcqAssistState::Freeze` → `Continuous` 전환이 토글 키의 트리거
- UGAP는 별도의 Measure 상태에서 `UGAPAcqAssistAnalyze` 를 발행 — SWE의 Freeze/Continuous 개념과 다름
