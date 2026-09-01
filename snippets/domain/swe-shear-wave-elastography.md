---
title: "SWE: Shear Wave Elastography"
category: domain
tags: [swe, elastography, liver, fibrosis, acquisition]
difficulty: beginner
---

Measures liver stiffness non-invasively by tracking how fast shear waves travel through tissue.

## Background

Shear waves radiate perpendicular to the ultrasound push pulse. Stiffer tissue — like fibrotic liver — propagates them faster. Speed (m/s) maps directly to stiffness (kPa), giving a non-invasive alternative to liver biopsy for fibrosis staging.

## What the Code Does

```cpp
// Entry point: gate on SWE mode, then delegate to assistant
if (ESMain::InElasto()) {
    m_sweAcqAssistant->HandleEvent(event);
}

// After processing:
SWEROIProcessor::ProcessResult result = m_roiProcessor->GetResult();
// result contains acoustic radiation force + stiffness map data
```

## Key Points

- `SWEAcquisitionAssistant` owns the SWE-specific acquisition logic; `ESMain::InElasto()` gates entry
- Results surface as `SWEROIProcessor::ProcessResult` (acoustic + stiffness measurements)
- In Freeze+Continuous mode, `AutoPos` key returns `""` — suppresses re-trigger while frozen
- ROI is fixed once frozen; the user must unfreeze to reposition
