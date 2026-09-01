---
title: "ProbeContactQualityIndicator"
category: domain
tags: [probe, contact, quality, indicator, warning]
difficulty: intermediate
---

`ProbeContactQualityIndicator` is the algorithm component that measures acoustic coupling — distinct from the boolean warning flag it ultimately sets.

## Background

Good acoustic contact between probe face and skin is required for reliable liver measurements. The indicator continuously evaluates coupling quality from B-mode data; when it drops below threshold, the `PoorProbeContact` warning becomes true and a yellow overlay appears in the UI.

## What the Code Does

```cpp
// Indicator = the algorithm (reusable component)
ProbeContactQualityIndicator m_contactIndicator;

// Warning = the UI state (boolean in AcquisitionAssistantBase)
bool m_bPoorProbeContactWarning = false;

// Update path:
void AcquisitionAssistantBase::UpdateWarnings() {
    m_bPoorProbeContactWarning = !m_contactIndicator.IsGoodContact();
}

// LiverAIProcessor also consumes the indicator directly
// for its own AI pipeline — it does not read the warning boolean
```

## Key Points

- Indicator and warning are separate: indicator is the algorithm object; warning is a derived boolean for UI
- Three consumers: `AcquisitionAssistantBase`, `LiverWarning3Panel`, and `LiverAIProcessor`
- `LiverAIProcessor` reads the indicator directly — bypasses the warning flag entirely
- During the refactor to `LiverWarning3Panel`, the indicator instance must be shared, not duplicated
