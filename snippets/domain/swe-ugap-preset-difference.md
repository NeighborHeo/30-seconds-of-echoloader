---
title: "SWE vs UGAP 프리셋 차이: 왜 같은 파라미터를 쓸 수 없나"
category: domain
tags: [SWE, UGAP, preset, elastography, attenuation, clinical]
difficulty: intermediate
---

SWE와 UGAP는 물리적 측정 원리가 달라 프리셋 파라미터를 공유할 수 없다. 같은 환자에게 연속 수행할 때 EchoConfig 재로딩이 필수다.

## Why

두 측정법이 비슷해 보여도 핵심 파라미터 집합이 완전히 다르다. SWE 프리셋으로 UGAP를 수행하면 주파수 쌍이 없어 감쇠 계산 자체가 불가능하다. `swe-vs-ugap-cannot-merge.md`가 구조적 분리를 다룬다면, 이 글은 프리셋 레벨에서 왜 분리가 필요한지 다룬다.

## Pattern

```cpp
// SWE 프리셋 파라미터 집합 (SWE_default.xml)
// 목적: 빠른 push → 전단파 추적 → stiffness map
// 핵심 축: 시간 해상도
struct SWEPresetParams
{
    float pushPulseAmplitude;    // push pulse 세기 (%) — 조직 변형 유발
    float pushPulseDuration;     // push pulse 지속시간 (us)
    float trackingFrequency;     // 전단파 추적 주파수 (MHz) — 4~5 MHz 전형적
    float roiDepthMax;           // ROI 최대 깊이 (cm) — SWE: 최대 8 cm
    float scanLineDensity;       // 스캔 라인 밀도 — 시간 해상도와 트레이드오프
    // 결과 단위: m/s (속도), kPa (탄성률)
};

// UGAP 프리셋 파라미터 집합 (UGAP_default.xml)
// 목적: 두 주파수 비교 → 주파수 의존 감쇠율
// 핵심 축: 주파수 정확도
struct UGAPPresetParams
{
    float baseFrequency1;        // 기준 주파수 f1 (MHz) — 예: 3.0 MHz
    float baseFrequency2;        // 기준 주파수 f2 (MHz) — 예: 5.0 MHz
    float frequencyBandwidth;    // 각 주파수의 대역폭 (MHz)
    float attenROIWidth;         // 감쇠 계산 ROI 폭 (cm)
    float attenROIDepth;         // 감쇠 계산 ROI 깊이 (cm) — UGAP: 최대 6 cm
    int   averagingCount;        // 평균화 횟수 — 노이즈 감소
    // 결과 단위: dB/cm/MHz (주파수 의존 감쇠율)
};

// 같은 환자에게 SWE → UGAP 연속 수행 시 필수 절차
void EchoScanner::SwitchFromSWEtoUGAP()
{
    // 1. SWE 측정 완료 및 결과 저장
    m_sweProcessor.FinalizeResults();

    // 2. 프리셋 교체 — EchoConfig 재로딩 필수
    //    SWE 프리셋에는 f1/f2가 없어 UGAP 계산 불가
    const std::string prevPreset =
        EchoConfigManager::Instance().GetCurrentPresetName();

    if (!EchoConfigManager::Instance().LoadPreset("UGAP_default"))
    {
        ScLogError("SCANNER", "Failed to switch to UGAP preset");
        return;
    }

    EchoConfigManager::Instance().BroadcastPresetParams();

    // 3. AA 재초기화 — UGAP 알고리즘 파라미터로
    m_ugapAcqAssist.Init(EchoConfig::PresetKeys::UGAP_ALGORITHM);

    // 4. 감사 로그 — 임상 측정 컨텍스트 전환
    AuditLogUserEvent(
        AUDIT_LOG,
        "PRESET_CHANGE",
        "Measurement context switched from '%s' to UGAP_default",
        prevPreset.c_str()
    );
}

// 파라미터 불일치 시 조용한 실패 예시 (하지 말 것)
void UGAPAcqAssist::ComputeAttenuation_WRONG()
{
    // SWE 프리셋 로드 상태에서 UGAP 파라미터 읽기 시도
    const float f1 = EchoConfigManager::Instance().GetFloat("BaseFrequency1", 0.0f);
    const float f2 = EchoConfigManager::Instance().GetFloat("BaseFrequency2", 0.0f);

    if (f1 == 0.0f || f2 == 0.0f)
    {
        // 폴백값 0.0으로 계산 → 감쇠율 = 0 또는 divide-by-zero
        // 반환값이 0.0 dB/cm/MHz → 임상적으로 "정상"으로 해석될 수 있음
    }
}
```

## SWE vs UGAP 프리셋 비교

```
파라미터          SWE 프리셋          UGAP 프리셋
─────────────────────────────────────────────────
push pulse        필수 (세기/시간)    없음
추적 주파수       필수 (4~5 MHz)      없음
기준 주파수 쌍    없음                필수 (f1, f2)
ROI 최대 깊이     8 cm               6 cm
평균화 횟수       없음                필수 (노이즈 감소)
결과 단위         m/s, kPa           dB/cm/MHz

임상 질문         간 섬유화 정도?     간 지방증 정도?
물리적 원리       전단파 전파 속도    초음파 주파수 의존 감쇠
측정 관심사       시간 해상도         주파수 정확도
```

## Key Points

- SWE 프리셋으로 UGAP을 수행하면 f1/f2가 없어 감쇠율 계산 자체가 실패하거나 0으로 폴백된다 — 조용한 임상 오류다.
- ROI 깊이도 다르다: SWE 최대 8 cm, UGAP 최대 6 cm — 같은 ROI 설정을 그대로 쓰면 UGAP ROI가 범위를 벗어날 수 있다.
- 같은 환자에게 SWE 후 UGAP 수행 시 반드시 `EchoConfigManager::LoadPreset("UGAP_default")` → `BroadcastPresetParams()` → `AA::Init()` 순서를 밟아야 한다.
- SWE와 UGAP은 임상적으로 다른 질문에 답한다: SWE = 섬유화(조직 강성), UGAP = 지방증(감쇠율) — 하나로 합칠 수 없는 이유가 물리에 있다.
- 컨텍스트 전환은 `AuditLogUserEvent("PRESET_CHANGE", ...)`로 기록해야 한다 — 같은 세션에서 두 측정을 수행한 임상 기록이 추적 가능해야 한다.
