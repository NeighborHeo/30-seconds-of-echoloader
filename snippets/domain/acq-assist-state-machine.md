---
title: "Acquisition Assistant State Machine"
category: domain
tags: [state-machine, acquisition, swe, ugap, gc-parameter]
difficulty: advanced
---

`AcqAssistState` is a 5-value enum that encodes the current ROI automation level and scan mode combination transmitted across the GC parameter boundary.

## Background

The acquisition assistant spans two architectural layers (Layer A and Layer B). The integer value of the enum is the payload that crosses that boundary, so both sides must agree on the numeric mapping. This is why the enum is defined in two header files that must stay in sync.

## What the Code Does

```cpp
enum AcqAssistState {
    NOT_STARTED             = 0,
    AUTO_ROI_ON_DEMAND_OFF  = 1, // OnDemand mode, warnings only, ROI off
    AUTO_ROI_ON_DEMAND_ON   = 2, // OnDemand mode, warnings + ROI once
    AUTO_ROI_CONTINUOUS_OFF = 3, // Continuous mode, warnings only
    AUTO_ROI_CONTINUOUS_ON  = 4, // Continuous mode, warnings + live ROI
    COMPLETED               = STATE_COMPLETED
};
// WARNING: defined in BOTH AcquisitionAssistantBase.h AND AcquisitionAssistant.h
// Numeric values must stay identical — GC parameter crosses layer boundary
```

## Key Points

- The integer value is transmitted as a GC parameter between Layer A and Layer B — reordering breaks the protocol
- Two definitions exist (Base + subclass headers); they must stay numerically identical
- States 1–2 cover OnDemand mode (user-triggered); states 3–4 cover Continuous mode (live tracking)
- `COMPLETED` maps to `STATE_COMPLETED`, a separate sentinel distinct from the numeric sequence
