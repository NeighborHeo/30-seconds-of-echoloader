---
title: "프리셋 문자열 하드코딩 위험: \"ShearTrackAlg\" 문제"
category: domain
tags: [preset, hardcoding, UGAP, risk, director-q2, silent-failure]
difficulty: intermediate
---

`"ShearTrackAlg"` 같은 프리셋 이름이 코드에 하드코딩되면, 이름이 바뀔 때 Init()이 조용히 성공하면서 잘못된 알고리즘 파라미터로 측정이 수행된다.

## Why

프리셋 이름은 외부 계약이다. XML 설정 파일이나 장비 팀이 관리하는 이름인데 코드에 박아두면, 이름 변경 시 컴파일 오류도 없고 런타임 예외도 없다. `Init()`은 true를 반환하되 기본값(또는 이전 프리셋)으로 조용히 폴백할 수 있다. 감쇠 계수 측정이 잘못된 알고리즘 파라미터로 수행되고, 그 결과가 임상 보고서에 기록된다. Director Q2: UGAP 팀에게 실제 프리셋 이름 확인 필수 (P-INIT spike 블로킹).

## Pattern

```cpp
// --- 위험한 패턴 ---
void UGAPAcqAssist::Init()
{
    // 하드코딩: 이름이 바뀌면 조용히 실패
    if (!m_acquisitionAssistant.Init("ShearTrackAlg"))
    {
        ScLogError("UGAP", "AcqAssist Init failed");
        return;
    }
    // Init()이 true를 반환해도 잘못된 프리셋일 수 있음
}

// --- 안전한 패턴 1: constexpr 단일 관리 ---
// ConfigKeys.h
namespace EchoConfig {
namespace PresetKeys {
    // ponytail: 상수 하나짜리 헤더, UGAP 팀 확인 후 값 확정 필요
    constexpr const char* UGAP_ALGORITHM = "ShearTrackAlg";  // TODO: Director Q2
}}

void UGAPAcqAssist::Init()
{
    if (!m_acquisitionAssistant.Init(EchoConfig::PresetKeys::UGAP_ALGORITHM))
    {
        ScLogError("UGAP", "AcqAssist Init failed with preset: %s",
                   EchoConfig::PresetKeys::UGAP_ALGORITHM);
        return;
    }
}

// --- 안전한 패턴 2: 레지스트리/XML 외부화 ---
void UGAPAcqAssist::Init()
{
    const std::string presetName =
        ScCommon::Registry::Read("UGAP", "AlgorithmPreset", "ShearTrackAlg");
    //                                                        ^ 폴백값은 남겨둠

    if (!m_acquisitionAssistant.Init(presetName))
    {
        ScLogError("UGAP", "AcqAssist Init failed with preset: %s",
                   presetName.c_str());
        return;
    }

    // Init 성공 후에도 실제 로드된 프리셋 이름을 검증
    const std::string loaded = m_acquisitionAssistant.GetLoadedPresetName();
    if (loaded != presetName)
    {
        ScLogError("UGAP", "Preset mismatch: requested '%s', loaded '%s'",
                   presetName.c_str(), loaded.c_str());
        // 임상 위험 — 측정 시작 전에 차단
    }
}
```

## 조용한 실패 시나리오

```
[UGAP Init 호출]
        │
        ▼
AcquisitionAssistant::Init("ShearTrackAlg")
  → 프리셋 파일 탐색
  → 파일 없음 또는 이름 불일치
  → 기본값(SWE_default)으로 폴백  ← 반환값: true (!)
        │
        ▼
UGAP 측정 시작
  → SWE 알고리즘 파라미터로 감쇠 계수 계산
  → ACResults: 의미 없는 값 (단위: dB/cm/MHz)
        │
        ▼
임상 보고서에 잘못된 감쇠 계수 기록
(지방간 위음성/위양성 가능)
```

## Key Points

- `Init()` 반환값 `true`는 프리셋이 올바르게 로드됐다는 보장이 아니다 — 로드된 프리셋 이름을 별도로 검증해야 한다.
- 프리셋 이름이 바뀌면 컴파일러가 잡아주지 않는다 — 문자열 계약은 코드 외부에서 깨진다.
- 단기 수정: `constexpr`로 단일 위치에서 관리하고 `TODO: Director Q2` 주석으로 추적한다.
- 장기 수정: `ScCommon::Registry::Read()`로 외부화하고 Init 후 로드된 이름을 검증한다.
- 임상 위험 등급: 잘못된 감쇠 계수는 지방간 진단 위음성/위양성으로 이어질 수 있어 환자 안전 이슈다.
