---
title: "UpdateDebugParameters: 내부 상태를 GC 파라미터로 노출하는 패턴"
category: cpp-patterns
tags: [debug, GC-parameter, UpdateDebugParameters, DebugParams, QA]
difficulty: intermediate
---

AA 내부 상태를 GC 파라미터 버스에 노출해 개발·임상 QA 중 외부에서 실시간 관찰 가능하게 만드는 패턴이다.

## Why

AA 알고리즘 상태(ROI 좌표, warning 플래그, 프레임 번호)는 UI에 직접 표시되지 않는다. 필드 엔지니어와 QA가 실시간으로 관찰하려면 GC 파라미터 버스를 통한 노출이 필요하다. 릴리스 빌드에서도 활성화되므로 임상 현장 지원에도 쓰인다.

## Pattern

```cpp
// AA 내부 상태를 담는 구조체
struct DebugParams
{
    bool   m_bWarningActive;        // warning 관련 필드 (D3 Lifecycle에서 분리 예정)
    float  m_roiX;
    float  m_roiY;
    float  m_roiWidth;
    float  m_roiHeight;
    int    m_algorithmParam;
    int    m_frameNumber;
};

class SWEAcquisitionAssistant : public AcquisitionAssistantBase
{
public:
    void UpdateDebugParameters();

private:
    DebugParams m_debugParams;
    // GC 파라미터 키: "AA_Debug_WarningActive", "AA_Debug_RoiX", ...
};

// P0a 작업: 스텁을 채워 실제 값 노출 (동작 변화 없음)
void SWEAcquisitionAssistant::UpdateDebugParameters()
{
    // DebugParams를 최신 내부 상태로 갱신
    m_debugParams.m_bWarningActive = m_bCurrentWarningActive;
    m_debugParams.m_roiX           = m_roiProcessor.GetRoiX();
    m_debugParams.m_roiY           = m_roiProcessor.GetRoiY();
    m_debugParams.m_frameNumber    = m_currentFrameIndex;

    // GC 파라미터 버스에 노출
    bool bOk = false;
    m_pGcParam->SetParameterValue("AA_Debug_WarningActive", m_debugParams.m_bWarningActive, bOk);
    m_pGcParam->SetParameterValue("AA_Debug_RoiX",          m_debugParams.m_roiX,           bOk);
    m_pGcParam->SetParameterValue("AA_Debug_FrameNumber",   m_debugParams.m_frameNumber,    bOk);

    ScLogInfo("SWEAcquisitionAssistant::UpdateDebugParameters — frame=%d warning=%d",
              m_debugParams.m_frameNumber, static_cast<int>(m_debugParams.m_bWarningActive));
}

// 호출 시점: ProcessFrame() 말미 또는 상태 변경 직후
void SWEAcquisitionAssistant::ProcessFrame(const SWEFrame& frame)
{
    // ... 알고리즘 처리 ...
    UpdateDebugParameters();  // 매 프레임 갱신
}
```

## Key Points

- DebugParams 구조체는 AA 내부 상태의 스냅샷이며, SetParameterValue로 GC 파라미터 버스에 노출된다.
- 릴리스 빌드에서도 활성화될 수 있어 임상 QA와 필드 엔지니어가 실시간 관찰에 사용한다.
- 현재 DebugParams에 warning 관련 필드가 혼재되어 있으며, D3 Lifecycle 묶음 작업에서 분리 예정이다.
- P0a 작업 범위: 스텁으로 비어 있던 UpdateDebugParameters를 실제 값을 노출하도록 채우되 동작 변화는 없다.
- DebugParams 멤버 수 증가 = AA 내부가 외부에 더 많이 노출됨을 의미하므로 최소한으로 유지해야 한다.
