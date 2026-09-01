---
title: "ROI 좌표계: 커서 위치 → ROI 배치 → GC 파라미터 전달"
category: domain
tags: [roi, coordinate-system, gc-parameter, trackball, swe, ugap]
difficulty: intermediate
---

트랙볼 이벤트로 이동한 ROI 중심 좌표는 픽셀 좌표계에서 GC 파라미터 버스를 통해 Layer B(GcViewer)로 전달되어 렌더링된다.

## Why

ROI(Region of Interest)는 SWE와 UGAP 모두에서 측정 대상 영역을 정의하는 핵심 개념이다. 커서 위치(픽셀)에서 GC 파라미터 문자열로의 변환 흐름을 이해하지 못하면 ROI가 화면에 잘못 표시되거나 측정값이 엉뚱한 위치에서 계산된다. Freeze 상태, 스로틀링 유무, 틸트 각도 등 도메인 조건이 이 흐름에 개입하기 때문에 좌표 파이프라인 전체를 한눈에 파악해야 한다.

## Pattern

```cpp
// 트랙볼 이벤트 → ROI 중심 이동 → GC 파라미터 전달 전체 흐름

// Step 1: TrbMove 이벤트 수신 (델타값, 픽셀 단위)
void CSWEAcquisitionAssistant::HandleTrbMove(int nDx, int nDy)
{
    if (m_bFrozen)
        return;  // Freeze 시 ROI 잠김 — 이벤트 무시

    m_nRoiCenterX += nDx;
    m_nRoiCenterY += nDy;

    // 화면 경계 클램핑 (0,0 = 좌상단)
    m_nRoiCenterX = std::max(0, std::min(m_nRoiCenterX, m_nImageWidth));
    m_nRoiCenterY = std::max(0, std::min(m_nRoiCenterY, m_nImageHeight));

    UpdateROICursorPosition();
}

// Step 2: ROI 좌표를 GC 파라미터로 변환하여 전달
void CSWEAcquisitionAssistant::UpdateROICursorPosition()
{
    // SWE: CursorMoved가 빠를 때 스로틀링 적용
    // ponytail: 단순 카운터 스로틀링, 정밀 타이머가 필요하면 교체
    if (++m_nCursorMoveCount % SWE_ROI_THROTTLE_RATE != 0)
        return;

    // 픽셀 좌표 → GC 파라미터 버스 전달 (Layer A → Layer B)
    SetParameterValue("SWEAcqAssist.RoiX", m_nRoiCenterX);
    SetParameterValue("SWEAcqAssist.RoiY", m_nRoiCenterY);

    // Layer B(GcViewer)가 파라미터 변경을 구독하여 점선 박스 리렌더링
}

// UGAP: 스로틀링 없이 즉시 처리
void CUGAPAcquisitionAssistant::HandleTrbMove(int nDx, int nDy)
{
    if (m_bFrozen)
        return;

    m_nRoiCenterX += nDx;
    m_nRoiCenterY += nDy;

    // UGAP은 틸트 각도(m_leftTiltAngle)도 함께 전달
    SetParameterValue("UGAPAcqAssist.RoiX",         m_nRoiCenterX);
    SetParameterValue("UGAPAcqAssist.RoiY",         m_nRoiCenterY);
    SetParameterValue("UGAPAcqAssist.TiltAngle",    m_leftTiltAngle);

    // 틸트 각도 임계값 초과 시 PoorProbe 경고 발생
    if (std::abs(m_leftTiltAngle) > UGAP_TILT_WARN_THRESHOLD_DEG)
        TriggerPoorProbeWarning();
}

// GC 파라미터 setter (Layer A → Layer B 브리지)
// 실제 구현은 gc-parameter-bus.md 참조
void CAcquisitionAssistantBase::SetParameterValue(const char* pszKey, int nValue)
{
    // GC 프레임워크 파라미터 버스에 기록 → OVObject 서브클래스가 구독
    m_pGCParamBus->Set(pszKey, nValue);
}
```

## Key Points

- **좌표계: 픽셀, (0,0) = 좌상단**: TrbMove는 절대 좌표가 아닌 델타값 — 핸들러에서 누적하여 중심 좌표 갱신 후 경계 클램핑 필수
- **GC 파라미터 키가 SWE/UGAP 네임스페이스로 분리**: `SWEAcqAssist.RoiX` vs `UGAPAcqAssist.RoiX` — 동시에 두 레이어가 활성화되어도 파라미터 충돌 없음
- **SWE는 스로틀링, UGAP은 즉시 처리**: 트랙볼을 빠르게 움직일 때 SWE는 GC 파라미터 업데이트 빈도를 제한하고, UGAP은 즉시 반영 — 도메인 응답성 요구사항이 다름
- **Freeze 시 TrbMove 무시**: `m_bFrozen` 플래그 확인이 두 핸들러 모두의 첫 번째 가드 — 누락 시 Freeze 중에도 ROI가 이동하는 버그
- **UGAP ROI는 틸트 각도 포함**: `m_leftTiltAngle`이 GC 파라미터로 함께 전달되며, 임계값 초과 시 `TriggerPoorProbeWarning()` 호출 — tilt-angle-validation.md 참조
