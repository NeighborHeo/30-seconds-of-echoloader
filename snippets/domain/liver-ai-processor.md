---
title: "LiverAIProcessor"
category: domain
tags: [ai, processor, liver, neural-network, anatomy]
difficulty: advanced
---

`LiverAIProcessor` (namespace `LiverAI`) runs a neural-network model over B-mode frames to detect anatomy and feed all three liver quality warnings simultaneously.

## Background

Rather than heuristic image processing, gipc-app delegates liver geometry detection to an AI model. The processor ingests B-mode frames and outputs capsule angle, skin-to-capsule distance, and contact quality — the three inputs that drive the entire warning triad and AutoPos pipeline.

## What the Code Does

```cpp
#include "LiverAIProcessor.h"
using namespace LiverAI;

// Single processor, multiple consumers
LiverAIProcessor m_liverAI;

void OnNewFrame(const BFrame& frame) {
    m_liverAI.Process(frame);

    // All three warnings read from the same processor output:
    auto scd   = m_liverAI.GetSkinToCapsuleResult();   // → LargeSCD warning
    auto angle = m_liverAI.GetCapsuleAngleResult();     // → ObliqueCapsule warning
    // ProbeContactQualityIndicator also calls into m_liverAI internally
}
```

## Key Points

- Namespace `LiverAI`, include `"LiverAIProcessor.h"` — no other header needed for consumers
- One instance feeds `AcquisitionAssistantBase`, `LiverWarning3Panel`, and `ProbeContactQualityIndicator`
- No consumer outside `ObjectViewer` — fully encapsulated; do not expose it across that boundary
- All three warning conditions are downstream of this single processor; fixing AI output fixes all three warnings
