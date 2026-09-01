---
title: "Tilt Angle Validation (PoorProbe Geometry)"
category: domain
tags: [ugap, poor-probe, tilt-angle, geometry, validation]
difficulty: intermediate
---

`m_leftTiltAngle` 등의 기울기 각도는 `PoorProbeContact` 경고가 직접 읽는 기하학적 상태다. UGAP에서 `updateROICursorPosition()` 이 이 값을 검증한다.

## Structure

```cpp
class AcquisitionAssistantBase : public OVObject
{
    // 기하학적 프로브 상태 — PoorProbeContact 경고가 읽음
    float m_leftTiltAngle  = 0.0f;   // 좌측 기울기 (도, degrees)
    float m_rightTiltAngle = 0.0f;   // 우측 기울기
    float m_tipTiltAngle   = 0.0f;   // 팁 기울기
};

// UGAP updateROICursorPosition 에서 틸트 검증
void UGAPAcquisitionAssistant::updateROICursorPosition(const CursorEvent& ev)
{
    // 2D/CM 뷰어 메타데이터 이전
    TransferViewerMetadata();

    // 틸트 각도 검증 — ObliqueCapsule 경고 조건과 다름
    float tiltMagnitude = std::abs(m_leftTiltAngle) + std::abs(m_rightTiltAngle);
    if (tiltMagnitude > POOR_PROBE_TILT_THRESHOLD_DEG)
    {
        m_bPoorProbeContactWarning = true;
        ScLogInfo("UGAP_GEOM", "Tilt exceeded threshold: %.1f deg", tiltMagnitude);
    }
    else
    {
        m_bPoorProbeContactWarning = false;
    }
}
```

## SWE vs UGAP 처리 차이

```
SWE::updateROICursorPosition:
  → CursorMoved 이벤트 스로틀링 (빠른 커서 이동 필터링)
  → 틸트 계산 없음 (SWE는 ObliqueCapsule만 적용)

UGAP::updateROICursorPosition:
  → 2D/CM 뷰어 메타데이터 이전 (추가 단계)
  → 틸트 각도 검증 (PoorProbeContact 조건)
  → 같은 함수 이름, 완전히 다른 의미
```

## 경고 조건 정리

```
PoorProbeContact: 틸트 과다 또는 접촉 불량 (m_leftTiltAngle 사용)
ObliqueCapsule:   캡슐 면과 초음파 빔의 비스듬한 각도 (별도 계산)
LargeSCD:         피부-캡슐 거리가 기준치 초과 (m_leftTiltAngle 미사용)
```

## Key Points

- `m_leftTiltAngle` 은 Base 외부 사용 0건 — 푸시다운 대상이지만 Warning과 결합되어 있음
- SWE와 UGAP의 `updateROICursorPosition` 은 이름만 같고 로직이 다름 — 절대 합치지 말 것
- 틸트 임계값(`POOR_PROBE_TILT_THRESHOLD_DEG`) 변경은 임상 검증 필요
- 기하학적 상태(`m_*TiltAngle`)는 D2 Geometry 묶음으로 P2+P4 후 푸시다운 예정
