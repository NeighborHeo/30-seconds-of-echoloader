---
title: "ESMain as System Hub: The Scanner's Central Coordinator"
category: architecture
tags: [esmain, coordinator, mode-gate, gc-params, architecture]
difficulty: intermediate
---

`ESMain.cpp` (~10k+ lines) is the scanner's single coordinator: it owns mode predicates, drives the GC parameter bus, and calls into `AcquisitionAssistant::Manager` at exactly 11 sites — but AA never calls back into ESMain business logic.

## Structure

```cpp
// Mode gates — owned exclusively by ESMain
ESMain::InElasto()        // SWE gate
ESMain::IsInUGAPMode()    // UGAP gate

// GC parameter bus — ESMain is the authority
ESMain::GetParameterValue(key, &value)
ESMain::SetParameterValue(key, value)

// AcquisitionAssistant call sites (lines 7066–9954)
// ESMain.cpp calls Manager; Manager never calls back
AcquisitionAssistant::Manager::Instance(HandlerType::SWE)->OnModeEnter();
//  ... 10 more sites spread across lines 7066–9954
```

## Why It Exists

A 10k-line coordinator is a code smell in most systems, but here it is intentional: the scanner has one physical mode at a time, and ESMain is the single place that knows which mode is active. Scattering mode transitions across packages would require every package to subscribe to mode events — ESMain's central position is load-bearing, not accidental.

## Key Points

- `InElasto()` and `IsInUGAPMode()` are the canonical truth for which modality is active — don't replicate these checks elsewhere
- AcquisitionAssistant is a **consumer** of ESMain, never a driver — the call graph is strictly one-way
- The GC parameter bus (`GetParameterValue` / `SetParameterValue`) is how ESMain communicates state to Layer B without a direct dependency
- Adding a new call site means touching one of the 11 known locations — there is no event subscription mechanism to wire up separately
