---
title: "AutoPos Key Resolution"
category: domain
tags: [autopos, roi, swe, ugap, key-dispatch]
difficulty: intermediate
---

`ResolveAutoPosKey()` returns the string parameter that triggers automatic ROI positioning — and SWE vs UGAP return different values by design.

## Background

AutoPos fires when a mode-specific string key is dispatched to the system. The key must be empty to suppress the trigger, or a specific string to activate it. Because SWE and UGAP have different triggering conditions (Freeze toggle vs Measure entry), each subclass resolves the key independently.

## What the Code Does

```cpp
// SWE subclass: suppress when frozen
std::string SWEAcquisitionAssistant::ResolveAutoPosKey() const {
    if (m_phase == Phase::FREEZE || m_phase == Phase::MEASUREMENT)
        return "";          // frozen — don't re-trigger
    return m_activeSweAutoPosKey;
}

// UGAP subclass: fire on Measure mode entry
std::string UGAPAcquisitionAssistant::ResolveAutoPosKey() const {
    if (m_phase == Phase::MEASUREMENT)
        return "UGAPAcqAssistAnalyze";   // triggers attenuation analysis
    return "";
}
```

## Key Points

- Empty string = no AutoPos trigger; non-empty string = dispatch that key to the system
- SWE returns empty in any frozen `Phase` — prevents ROI drift while image is held
- UGAP returns `"UGAPAcqAssistAnalyze"` specifically in `MEASUREMENT` phase, not in plain `FREEZE`
- Do not merge these into a base-class implementation — the triggering logic is intentionally different per mode
