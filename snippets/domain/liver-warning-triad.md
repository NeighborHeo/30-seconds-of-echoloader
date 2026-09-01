---
title: "Liver Warning Triad: LargeSCD, PoorProbeContact, ObliqueCapsule"
category: domain
tags: [warnings, liver, quality, scd, capsule, probe]
difficulty: intermediate
---

Three quality gates that must all pass before a SWE or UGAP measurement is considered reliable.

## Background

Liver elastography and attenuation measurements are sensitive to probe placement. The AI model continuously evaluates geometry and coupling; if any of the three conditions fails, the system flags it so the sonographer can correct before acquiring.

## What the Code Does

```cpp
// Three boolean warning flags in AcquisitionAssistantBase
bool m_bLargeSCDWarning;         // skin-to-capsule distance too large
bool m_bPoorProbeContactWarning; // acoustic coupling insufficient
bool m_bObliqueCapsuleWarning;   // capsule not perpendicular to beam

// LargeSCD: set when AI returns AboveThreshold
if (scdResult == SkintoCapsuleDistanceResult::AboveThreshold)
    m_bLargeSCDWarning = true;

// ObliqueCapsule: set on either extreme angle
if (angleResult == CapsuleAngleResult::AboveThreshold ||
    angleResult == CapsuleAngleResult::BelowNegativeThreshold)
    m_bObliqueCapsuleWarning = true;
```

## Key Points

- All three warnings are boolean flags; any `true` blocks a valid measurement
- LargeSCD fires on `SkintoCapsuleDistanceResult::AboveThreshold` — probe too far from liver surface
- ObliqueCapsule fires on either `AboveThreshold` or `BelowNegativeThreshold` — capsule angled either way
- These three flags are being extracted from `AcquisitionAssistantBase` into a dedicated `LiverWarning3Panel`
