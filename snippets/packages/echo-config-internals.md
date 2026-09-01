---
title: "EchoConfig 패키지 내부: 프리셋 관리자의 책임 범위"
category: packages
tags: [EchoConfig, preset, XML, validation, GC-parameter-bus, internals]
difficulty: intermediate
---

EchoConfig는 프리셋 파일 파싱부터 파라미터 검증, GC 버스 배포까지 담당한다. `echo-config.md`의 개요를 넘어 내부 동작과 경계를 다룬다.

## Why

EchoConfig 패키지의 "어디까지가 책임인가"를 모르면 기능을 엉뚱한 패키지에 넣거나 같은 검증 로직을 여러 곳에 중복 구현한다. 파라미터 검증은 EchoConfig 한 곳에서 끝내야 한다.

## Pattern

```cpp
// PresetLoader.h — XML 파싱 책임
class PresetLoader
{
public:
    // 프리셋 파일 → 내부 파라미터 맵
    // 반환 false: 파일 없음, XML 형식 오류, 필수 키 누락
    bool Load(const std::string& presetFilePath, PresetParamMap& outParams);

private:
    // XML 계층 구조:
    // <Preset name="SWE_default">
    //   <Algorithm>
    //     <PushPulseAmplitude unit="percent">80</PushPulseAmplitude>
    //     <TrackingFrequency unit="MHz">4.5</TrackingFrequency>
    //   </Algorithm>
    //   <UI>
    //     <ROIDepthMax unit="cm">8.0</ROIDepthMax>
    //   </UI>
    //   <Device readonly="true">
    //     <ProbeModel>9L</ProbeModel>
    //   </Device>
    // </Preset>
    bool ParseAlgorithmSection(XmlNode& node, PresetParamMap& params);
    bool ParseUISection(XmlNode& node, PresetParamMap& params);
    // Device 섹션: readonly — 파싱은 하되 덮어쓰기 불가
};

// EchoConfigManager.h — 검증 + 배포 책임
class EchoConfigManager
{
public:
    bool LoadPreset(const std::string& presetName);
    void BroadcastPresetParams();
    std::string GetCurrentPresetName() const;

private:
    // 범위 검증 — EchoConfig가 단일 검증 지점
    bool ValidateParams(const PresetParamMap& params);

    // 커스텀 프리셋 vs 공장 프리셋 구분
    // Factory: <Device readonly="true"> 섹션 변경 불가
    // Custom:  Algorithm/UI 섹션만 사용자 수정 허용
    bool IsFactoryPreset(const std::string& presetName) const;

    // GC 파라미터 버스로 배포
    // 구독자: EchoScanner, GcViewer, EchoMeasure
    void PublishToGcBus(const PresetParamMap& params);

    PresetParamMap m_currentParams;
    std::string m_currentPresetName;
    PresetLoader m_loader;
};

// 파라미터 범위 검증 예시
bool EchoConfigManager::ValidateParams(const PresetParamMap& params)
{
    // 공통 파라미터
    const float depth = params.GetFloat("ROIDepthMax");
    if (depth < 1.0f || depth > 25.0f)
    {
        ScLogError("ECHOCONFIG", "ROIDepthMax out of range: %.1f cm", depth);
        return false;
    }

    const float gain = params.GetFloat("Gain");
    if (gain < 0.0f || gain > 100.0f)
    {
        ScLogError("ECHOCONFIG", "Gain out of range: %.1f", gain);
        return false;
    }

    // SWE 전용 파라미터 (UGAP 프리셋에 없으면 GetFloat 기본값 사용)
    if (params.Contains("PushPulseAmplitude"))
    {
        const float amp = params.GetFloat("PushPulseAmplitude");
        if (amp < 0.0f || amp > 100.0f)
        {
            ScLogError("ECHOCONFIG", "PushPulseAmplitude out of range: %.1f", amp);
            return false;
        }
    }

    return true;
}

// 변경 전 스냅샷 → 배포 → 감사 로그 패턴
bool EchoConfigManager::LoadPreset(const std::string& presetName)
{
    // 변경 전 상태 보존 (롤백/감사 용도)
    const PresetParamMap snapshot = m_currentParams;
    const std::string prevName = m_currentPresetName;

    PresetParamMap newParams;
    const std::string filePath = ResolvePresetFilePath(presetName);

    if (!m_loader.Load(filePath, newParams))
    {
        ScLogError("ECHOCONFIG", "PresetLoader failed: %s", filePath.c_str());
        return false;  // snapshot 그대로 유지
    }

    if (!ValidateParams(newParams))
    {
        ScLogError("ECHOCONFIG", "Preset validation failed: %s", presetName.c_str());
        return false;  // snapshot 그대로 유지
    }

    m_currentParams = newParams;
    m_currentPresetName = presetName;
    return true;
}
```

## 책임 경계

```
EchoConfig 담당:
  ✓ 프리셋 파일 경로 해석 (레지스트리 또는 설정 파일에서)
  ✓ XML 파싱 및 내부 맵 변환
  ✓ 파라미터 범위 검증 (단일 검증 지점)
  ✓ 커스텀/공장 프리셋 구분
  ✓ GC 버스로 파라미터 배포
  ✓ 로딩 실패 시 이전 상태 보존

EchoConfig 비담당:
  ✗ AA 상태머신 리셋 — EchoScanner/UGAPAcqAssist 담당
  ✗ 감사 로그 기록 — 호출자(EchoFrontPanel 등) 담당
  ✗ UI 갱신 — GcViewer 구독자 콜백이 처리
  ✗ 프리셋 선택 UI — EchoFrontPanel 담당
```

## Key Points

- 파라미터 범위 검증은 `EchoConfigManager::ValidateParams()` 한 곳에서만 한다 — 각 패키지에서 중복 검증하지 않는다.
- 커스텀 프리셋은 Algorithm/UI 섹션만 수정 가능; `readonly="true"` Device 섹션은 파싱만 하고 변경을 거부한다.
- `LoadPreset()` 실패 시 이전 `m_currentParams` 스냅샷이 유지된다 — 부분 적용 상태가 없다.
- `Init(presetName)` 호출 전에 `BroadcastPresetParams()` 완료가 선행되어야 한다 — GC 버스 구독자가 먼저 파라미터를 수신해야 AA가 올바른 컨텍스트로 Init된다.
- 의존 방향: `EchoLoader → EchoConfig → (GC 버스) → EchoScanner, GcViewer, EchoMeasure` — EchoConfig는 하위 패키지를 직접 알지 않는다.
