---
title: "프리셋 로딩 파이프라인: 선택 → 로딩 → 적용 → AA 초기화"
category: architecture
tags: [preset, EchoConfig, AcquisitionAssistant, initialization, audit]
difficulty: intermediate
---

사용자가 프리셋을 선택하면 EchoConfig가 파일을 파싱하고 GC 파라미터 버스로 배포한 뒤, 각 패키지는 AcquisitionAssistant를 재초기화한다.

## Why

프리셋 로딩은 단순한 파일 읽기가 아니라 시스템 상태 전환이다. EchoConfig → GC 버스 → AA Init 순서가 어긋나면 이전 프리셋 파라미터로 측정이 시작된다. 순서를 알아야 순서 버그를 잡는다.

## Pattern

```cpp
// 1. 사용자 프리셋 선택 — UI 이벤트 진입점
void EchoFrontPanel::OnPresetSelected(const std::string& presetName)
{
    const std::string oldPreset = EchoConfigManager::Instance().GetCurrentPresetName();

    // 2. EchoConfig: 파일 파싱 + 내부 설정 맵 갱신
    if (!EchoConfigManager::Instance().LoadPreset(presetName))
    {
        ScLogError("PRESET", "Failed to load preset: %s", presetName.c_str());
        return;  // 이전 프리셋 유지 — 조용히 폴백하지 않음
    }

    // 3. GC 파라미터 버스로 배포 — 각 패키지가 구독자로 수신
    //    EchoScanner, GcViewer, EchoMeasure 등이 OnParamChanged() 콜백 수신
    EchoConfigManager::Instance().BroadcastPresetParams();

    // 4. AcquisitionAssistant 재초기화 — 상태머신 리셋 포함
    //    Init() 전에 BroadcastPresetParams() 완료 필수
    m_acquisitionAssistant.Init(presetName);

    // 5. 임상 추적성 — 프리셋 변경은 감사 로그 의무
    AuditLogUserEvent(
        AUDIT_LOG,
        "PRESET_CHANGE",
        "Preset changed from '%s' to '%s' by operator",
        oldPreset.c_str(),
        presetName.c_str()
    );
}

// 부팅 시 기본 프리셋 자동 로딩 — EchoLoader 초기화 순서
void EchoLoader::Init()
{
    // EchoConfig가 반드시 먼저 초기화되어야 함
    m_echoConfig.Init();

    const std::string defaultPreset =
        ScCommon::Registry::Read("EchoLoader", "DefaultPreset", "SWE_default");

    m_echoConfig.LoadPreset(defaultPreset);
    m_echoConfig.BroadcastPresetParams();

    // 하위 패키지들은 EchoConfig Init 완료 후 Init
    m_echoScanner.Init();
    m_echoMeasure.Init();
}
```

## 파이프라인 흐름

```
[사용자 프리셋 선택]
        │
        ▼
EchoConfigManager::LoadPreset(presetName)
  → XML/INI 파싱
  → 파라미터 검증 (범위, 필수 키)
  → 내부 설정 맵 갱신
        │
        ▼
EchoConfigManager::BroadcastPresetParams()
  → GC 파라미터 버스 publish
  → EchoScanner::OnParamChanged()   ← 획득 파라미터 수신
  → GcViewer::OnParamChanged()      ← 렌더링 파라미터 수신
  → EchoMeasure::OnParamChanged()   ← 측정 단위/포맷 수신
        │
        ▼
AcquisitionAssistant::Init(presetName)
  → 상태머신 리셋 (Idle 상태로)
  → 알고리즘 파라미터 설정
  → 내부 버퍼 클리어
        │
        ▼
AuditLogUserEvent("PRESET_CHANGE", ...)
```

## Key Points

- `BroadcastPresetParams()` 완료 전에 `AA::Init()` 호출 시 이전 프리셋 파라미터로 알고리즘이 초기화됨 — 순서가 계약이다.
- `LoadPreset()` 실패 시 이전 프리셋을 유지하고 명시적으로 로그를 남겨야 한다; 조용한 폴백은 임상 위험이다.
- 부팅 시 기본 프리셋 이름은 `ScCommon::Registry::Read()`로 읽는다 — 코드에 하드코딩하지 않는다.
- `AA::Init()`은 상태머신을 Idle로 리셋한다 — 진행 중인 측정이 있으면 먼저 중단해야 한다.
- 프리셋 변경은 임상 설정 변경이므로 `AuditLogUserEvent("PRESET_CHANGE", ...)` 없이 완료되어서는 안 된다.
