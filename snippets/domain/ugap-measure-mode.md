---
title: "UGAP: Measure Mode and Analysis Key"
category: domain
tags: [ugap, measure-mode, autopos, analysis, attenuation]
difficulty: intermediate
---

UGAP는 Measure 모드에서 `"UGAPAcqAssistAnalyze"` 키를 AutoPos에 보내 감쇠 계수(Attenuation Parameter) 계산을 트리거한다.

## Behavior

```cpp
// UGAPAcqAssistHandler::ResolveAutoPosKey()
std::string UGAPAcqAssistHandler::ResolveAutoPosKey(AcqAssistState state)
{
    if (state == AcqAssistState::Measure)
    {
        // Measure 모드 진입 시 즉시 분석 키 발행
        return "UGAPAcqAssistAnalyze";
    }
    return "";
}
```

## 측정 흐름

```
[Scanning / B-mode]
    │
    ▼ 사용자가 ROI 배치 확인
[Measure 모드 활성화]
    │  ─ AutoPos 키 = "UGAPAcqAssistAnalyze"
    │  ─ m_currentFrame_BS + m_currentFrame_BS_UGAP 스냅샷
    │
    ▼ ACResults 계산
[감쇠 계수 표시 (dB/cm/MHz)]
    │
    │  측정값 예시: 0.62 dB/cm/MHz → 정상 지방 수준
    │              > 0.8 dB/cm/MHz → 지방간 의심
```

## SWE와 비교

```
SWE 모드 게이팅:   ESMain::InElasto()
UGAP 모드 게이팅:  ESMain::IsInUGAPMode()

SWE AutoPos 키:   "" (Freeze) / "SWEAcqAssistToggleContinuous" (Continuous)
UGAP AutoPos 키:  "UGAPAcqAssistAnalyze" (Measure 시)

SWE 결과 타입:    SWEROIProcessor::ProcessResult  (m/s, kPa)
UGAP 결과 타입:   ACResults                       (dB/cm/MHz)
```

## 주의사항

```cpp
// UGAP preset 문자열 하드코딩 위험 — Director 확인 필요
// RBUG-???: "ShearTrackAlg" 가 실제 UGAP preset 이름인지 검증 필요
// → analysis/SWEUGAPAcqAssist/15_phaseB4_b6_plan.md §4 참조
```

## Key Points

- `"UGAPAcqAssistAnalyze"` 는 레이어 A → 레이어 B 전달 키 — 문자열 typo는 묵음 실패
- Measure 모드는 `ESMain::IsInUGAPMode()` 로 게이팅 — SWE와 완전 분리
- 두 채널(`m_currentFrame_BS` + `m_currentFrame_BS_UGAP`) 동기화가 측정 정확도의 핵심
- UGAP preset 이름(`"ShearTrackAlg"` 등)은 외부 계약 — 변경 전 Director sign-off 필수
