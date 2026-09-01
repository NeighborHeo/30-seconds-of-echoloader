---
title: "Acquisition Assistant Phase Lifecycle"
category: domain
tags: [phase, lifecycle, scanstate, measurement, freeze]
difficulty: intermediate
---

`AcquisitionAssistant::Phase` is a higher-level lifecycle concept synthesised from scanner state and measurement axis — it exists because callers need finer distinctions than raw `ScanState` provides.

## Background

`ScanState` tells you whether the scanner is running or frozen. But callers also need to know whether the measure tool is active, and whether acquisition has ever been entered. `Phase` combines all three inputs into one enum that callers can switch on without importing `EchoMeasure` internals.

## What the Code Does

```cpp
enum class Phase {
    NO_ACTIVE,              // SWE/UGAP mode not active
    PRE_MODE,               // mode active, pre-acquisition (ROI placement)
    ACQUISITION,            // actively scanning
    FREEZE,                 // image frozen, no measure tool
    MEASUREMENT,            // frozen + measure tool active
    MEASUREMENT_DEACTIVE    // frozen, measure was active then deactivated
};

// Synthesised from:
Phase ComputePhase(bool hasModeProperty,
                   RunState runState,
                   bool isMeasureActive,
                   bool hasEnteredAcquisition);
```

## Key Points

- `Phase` != `ScanState`: Phase adds measure-tool axis and acquisition-entry history
- `FREEZE` vs `MEASUREMENT` is the key distinction — AutoPos and warning display differ between them
- `MEASUREMENT_DEACTIVE` lets callers detect the "was measuring, stopped" transition without polling
- Synthesised centrally so no caller needs to re-implement the same three-input logic
