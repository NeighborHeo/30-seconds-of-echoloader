---
title: "ROI Processor Concept"
category: domain
tags: [roi, processor, swe, ugap, cursor]
difficulty: intermediate
---

Each acquisition mode owns its own ROI processor with a distinct result type — the base class deliberately does not unify them.

## Background

The Region of Interest (ROI) is the box the sonographer positions over the liver. SWE and UGAP compute fundamentally different quantities from the same spatial region (shear wave speed vs attenuation coefficient), so their processors and result types are kept separate in each subclass.

## What the Code Does

```cpp
// SWE processor — lives in SWEAcquisitionAssistant
SWEROIProcessor m_roiProcessor;
SWEROIProcessor::ProcessResult result = m_roiProcessor.GetResult();
// ProcessResult: acoustic radiation force data + stiffness map

// UGAP processor — lives in UGAPAcquisitionAssistant
// Result type: ACResults (attenuation coefficient results)
ACResults acResult = m_ugapProcessor.GetACResults();

// updateROICursorPosition() exists in both — semantics differ:
// SWE:  throttles cursor-moved events (rate limiting)
// UGAP: transfers 2D/CM viewer metadata + validates tilt angle
```

## Key Points

- `SWEROIProcessor::ProcessResult` and `ACResults` are distinct types — do not cast between them
- The base class `AcquisitionAssistantBase` does not own or expose a processor; processors are subclass-private
- `updateROICursorPosition()` shares a name but does different work in each subclass — read both before touching either
- Tilt angle validation in UGAP's cursor update can reject repositioning; SWE's only throttles the event rate
