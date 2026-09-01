---
title: "OVObject 렌더 루프: Render()에서 무엇을 그리나"
category: architecture
tags: [OVObject, Render, OVGraphics, thread-safety, GcViewer]
difficulty: intermediate
---

GcViewer 스케줄러가 매 프레임 Render()를 호출한다 — 계산은 ProcessFrame()에서, 표시만 Render()에서.

## Why

Render()에서 무거운 계산을 수행하면 프레임 드롭과 UI 블로킹이 발생한다. 렌더 루프와 데이터 처리 스레드가 분리되어 있으므로 공유 상태 접근 시 동기화가 필요하다.

## Pattern

```cpp
// GcViewer 스케줄러가 매 프레임 Render() 호출
class SWEAcquisitionAssistant : public AcquisitionAssistantBase
{
public:
    void Render(OVGraphics::Context& ctx) override;
    void ProcessFrame(const SWEFrame& frame);  // 별도 스레드에서 호출

private:
    std::mutex                              m_renderMutex;
    OVGraphics::Pointer<OVGraphics::Overlay> m_pRoiOverlay;
    OVGraphics::Pointer<OVGraphics::Overlay> m_pStiffnessColorMap;
    OVGraphics::Pointer<OVGraphics::Overlay> m_pWarningIcon;
    SWEFrameResult                          m_currentFrameResult;  // ProcessFrame이 갱신
};

// ProcessFrame: 알고리즘 계산 → 결과를 멤버에 저장
void SWEAcquisitionAssistant::ProcessFrame(const SWEFrame& frame)
{
    SWEFrameResult result = m_algorithm.Compute(frame);  // 무거운 계산은 여기서

    std::lock_guard<std::mutex> lock(m_renderMutex);
    m_currentFrameResult = result;  // Render()가 읽을 스냅샷 갱신
}

// Render: 저장된 결과만 표시 — 계산 금지
void SWEAcquisitionAssistant::Render(OVGraphics::Context& ctx)
{
    std::lock_guard<std::mutex> lock(m_renderMutex);  // 공유 상태 보호
    SWEFrameResult snapshot = m_currentFrameResult;
    // lock 해제 후 렌더링 (lock 구간 최소화)

    // ROI 박스 그리기
    m_pRoiOverlay->SetRect(snapshot.m_roiRect);
    ctx.Draw(m_pRoiOverlay);

    // Stiffness color map 오버레이
    m_pStiffnessColorMap->SetData(snapshot.m_colorMapData);
    ctx.Draw(m_pStiffnessColorMap);

    // 경고 아이콘 (조건부)
    if (snapshot.m_bWarningActive)
    {
        ctx.Draw(m_pWarningIcon);
    }
}

// UGAP: ROI 박스 + 감쇠 계수 텍스트 + 틸트 인디케이터
void UGAPAcquisitionAssistant::Render(OVGraphics::Context& ctx)
{
    std::lock_guard<std::mutex> lock(m_renderMutex);
    UGAPFrameResult snapshot = m_currentFrameResult;

    ctx.Draw(m_pRoiOverlay);
    m_pAttenuationText->SetText(FormatAttenuation(snapshot.m_attenuationCoeff));
    ctx.Draw(m_pAttenuationText);
    m_pTiltIndicator->SetAngle(snapshot.m_tiltAngleDeg);
    ctx.Draw(m_pTiltIndicator);
}

// LiverWarning3Panel: 경고 아이콘 3개
void LiverWarning3Panel::Render(OVGraphics::Context& ctx)
{
    if (m_bLargeSCD)      ctx.Draw(m_pLargeSCDIcon);
    if (m_bPoorProbe)     ctx.Draw(m_pPoorProbeIcon);
    if (m_bObliqueCapsule) ctx.Draw(m_pObliqueCapsuleIcon);
}
```

## Key Points

- Render(OVGraphics::Context& ctx)는 GcViewer 스케줄러가 매 프레임 호출하며, 무거운 계산을 수행해서는 안 된다.
- 알고리즘 계산은 ProcessFrame()에서 수행하고 결과를 멤버 변수에 저장한 뒤, Render()는 그 스냅샷을 표시만 한다.
- 렌더 루프와 ProcessFrame() 스레드가 분리되어 있으므로 공유 상태(m_currentFrameResult 등) 접근 시 뮤텍스로 보호해야 한다.
- OVGraphics::Pointer\<OVGraphics::Overlay\>로 렌더링 객체를 관리하며, Render() 내에서 생성·업데이트한다.
- SWE는 ROI 박스·stiffness color map·경고 아이콘, UGAP는 ROI 박스·감쇠 계수 텍스트·틸트 인디케이터, LiverWarning3Panel은 LargeSCD·PoorProbe·ObliqueCapsule 아이콘 3개를 그린다.
