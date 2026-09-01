---
title: "Skin-to-Capsule Distance (SCD)"
category: domain
tags: [scd, capsule, distance, warning, angle]
difficulty: beginner
---

SCD is the physical distance from the skin surface to the liver capsule; if too large, shear waves attenuate before reaching the liver and SWE results become unreliable.

## Background

The liver capsule must be close enough to the probe (typically within 2–3 cm) for the acoustic push pulse to generate detectable shear waves inside the parenchyma. The LiverAI model measures this distance in mm from each B-mode frame in real time.

## What the Code Does

```cpp
enum class SkintoCapsuleDistanceResult {
    Error,           // model could not measure
    BelowThreshold,  // distance OK — proceed
    AboveThreshold   // too far — LargeSCD warning fires
};

enum class CapsuleAngleResult {
    Error,
    BelowThreshold,           // angle OK
    BelowNegativeThreshold,   // capsule tilted the other way — ObliqueCapsule fires
    AboveThreshold            // capsule tilted — ObliqueCapsule fires
};

// Warning update
m_bLargeSCDWarning = (scdResult == SkintoCapsuleDistanceResult::AboveThreshold);
```

## Key Points

- `SkintoCapsuleDistanceResult` has three values: `Error`, `BelowThreshold`, `AboveThreshold`
- LargeSCD warning fires only on `AboveThreshold` — `Error` does not set the warning (treated as no-data)
- `CapsuleAngleResult` is a separate enum with four values; both extreme values trigger `ObliqueCapsuleWarning`
- Both enums come from `LiverAIProcessor` output — same processing frame, two distinct result types
