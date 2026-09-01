---
title: "D5-stub 7개 핸들러: 즉시 가능한 푸시다운 작업"
category: architecture
tags: [refactoring, pushdown, stub, acquisition-assistant, swe, ugap]
difficulty: intermediate
---

AcquisitionAssistantBase에 있는 7개 빈 stub 핸들러는 Warning 로직을 전혀 참조하지 않으므로, P2/P4 완료를 기다리지 않고 즉시 서브클래스로 푸시다운할 수 있다.

## Why

AcquisitionAssistantBase는 926줄에 Warning 로직, ROI 연산, UI 업데이트, 기하학 계산이 뒤섞여 있다. 이 중 7개 핸들러는 완전히 빈 구현(body가 `{}`)으로, `m_bLargeSCDWarning` 같은 Warning 플래그를 한 군데도 참조하지 않는다. Base에서 이 7개를 제거하면 Base에 Warning 관련 코드만 남게 되어 이후 LiverWarning3Panel 추출 단계가 단순해진다. 저위험, 즉시 시작 가능한 첫 번째 리팩토링 단계다.

## Pattern

```cpp
// Before: AcquisitionAssistantBase (현재 상태, 총 ~21줄)
class CAcquisitionAssistantBase : public IAcqAssistHandler
{
public:
    // D5-stub: 빈 구현, Warning 플래그 참조 없음 → 즉시 제거 가능
    virtual void HandleSelectDn(int nX, int nY)       override {}
    virtual void HandleSelectUp(int nX, int nY)       override {}
    virtual void HandleTrbMove(int nDx, int nDy)      override {}
    virtual void HandleSelectMoved(int nX, int nY)    override {}
    virtual void HandleDoubleSelectDn(int nX, int nY) override {}
    virtual void HandleLongSelectDn(int nX, int nY)   override {}
    virtual void HandleSelectCancel()                  override {}

    // Warning 관련 멤버 (D5-stub과 완전 독립)
    bool m_bLargeSCDWarning;
    bool m_bPoorProbeContactWarning;
    void UpdateWarningState();   // ← D5-stub은 이 함수를 호출하지 않음
};

// After: Base에서 stub 7개 제거, 서브클래스에 각자 구현
class CSWEAcquisitionAssistant : public CAcquisitionAssistantBase
{
public:
    // SWE 의미: SelectDn = 측정 시작 트리거
    void HandleSelectDn(int nX, int nY) override
    {
        if (m_eState == EAcqState::ROIPlaced)
            StartMeasurement(nX, nY);
    }

    void HandleTrbMove(int nDx, int nDy) override
    {
        MoveROICenter(nDx, nDy);
        UpdateROICursorPosition();
    }

    // SWE에서 사용하지 않는 이벤트는 명시적 빈 구현
    void HandleSelectUp(int nX, int nY)       override {}
    void HandleSelectMoved(int nX, int nY)    override {}
    void HandleDoubleSelectDn(int nX, int nY) override {}
    void HandleLongSelectDn(int nX, int nY)   override {}
    void HandleSelectCancel()                  override {}
};

class CUGAPAcquisitionAssistant : public CAcquisitionAssistantBase
{
public:
    // UGAP 의미: SelectDn = ROI 위치 확정
    void HandleSelectDn(int nX, int nY) override
    {
        if (m_eState == EAcqState::Positioning)
            ConfirmROIPosition(nX, nY);
    }

    void HandleTrbMove(int nDx, int nDy) override
    {
        MoveROICenter(nDx, nDy);  // UGAP은 즉시 처리 (스로틀링 없음)
    }

    void HandleSelectUp(int nX, int nY)       override {}
    void HandleSelectMoved(int nX, int nY)    override {}
    void HandleDoubleSelectDn(int nX, int nY) override {}
    void HandleLongSelectDn(int nX, int nY)   override {}
    void HandleSelectCancel()                  override {}
};
```

## Key Points

- **7개 stub = ~21줄 제거**: Base에서 빈 구현 7개를 삭제하는 것만으로 Base 크기가 줄고 Warning 로직이 시각적으로 분리됨
- **Warning 플래그 의존성 없음이 핵심 근거**: `m_bLargeSCDWarning`, `m_bPoorProbeContactWarning` 등 Warning 관련 상태를 한 곳도 읽거나 쓰지 않으므로 분리 시 동작 변화 없음
- **SWE와 UGAP의 SelectDn 의미가 다름**: 푸시다운 후 각 서브클래스가 도메인에 맞는 구현을 갖게 됨 — Base에 빈 구현이 있을 때보다 의도가 명확해짐
- **P2/P4 완료 전에 먼저 가능**: Warning 추출(P2)이나 ROI 분리(P4) 작업과 독립적 — 이 작업이 완료되면 오히려 후속 작업의 Base 코드 범위가 좁아져 P2 추출이 쉬워짐
- **남은 Base에 Warning 코드만 집중**: 7개 stub 제거 후 Base에는 `UpdateWarningState()`, `m_bLargeROIWarning`, LiverWarning3Panel 관련 코드가 명확하게 드러남 → LiverWarning3Panel 추출(P2 목표)이 단순해짐
