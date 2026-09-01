---
title: "UGAP: Ultrasound-Guided Attenuation Parameter"
category: domain
tags: [ugap, steatosis, liver, attenuation, acquisition]
difficulty: beginner
---

Quantifies hepatic fat content by measuring how much the ultrasound signal attenuates as it travels through liver tissue.

## Background

Fat-laden liver cells scatter and absorb sound differently than healthy tissue. The attenuation coefficient (dB/cm/MHz) rises with fat infiltration, enabling non-invasive detection and grading of NAFLD/MASLD without biopsy.

## What the Code Does

```cpp
// Gate on UGAP mode
if (ESMain::IsInUGAPMode()) {
    m_ugapAcqAssistant->HandleEvent(event);
}

// UGAP maintains two frame channels (unlike SWE's one)
m_currentFrame_BS;       // standard B-mode frame
m_currentFrame_BS_UGAP;  // UGAP-processed frame

// AutoPos in Measure mode fires analysis
// returns "UGAPAcqAssistAnalyze" — triggers attenuation calculation
```

## Key Points

- `UGAPAcquisitionAssistant` owns UGAP logic; `ESMain::IsInUGAPMode()` gates entry
- Two frame channels (`m_currentFrame_BS` + `m_currentFrame_BS_UGAP`) vs SWE's single channel — do not conflate them
- Results surface as `ACResults` (attenuation coefficient results), distinct from SWE's `ProcessResult`
- AutoPos key is `"UGAPAcqAssistAnalyze"` in Measure mode — triggered on measurement entry, not Freeze toggle
