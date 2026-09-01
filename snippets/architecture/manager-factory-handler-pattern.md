---
title: "Manager–Factory–Handler: The AcquisitionAssistant Triad"
category: architecture
tags: [singleton, factory, interface, swe, ugap, design-pattern]
difficulty: beginner
---

`AcquisitionAssistant` uses a three-part pattern — Manager (singleton dispatch), Factory (object creation), Handler interface (modality impl) — so `ESMain.cpp` never needs to know which ultrasound modality is active.

## Structure

```cpp
// Manager: single entry point for all callers
AcquisitionAssistant::Manager::Instance(HandlerType::SWE)->DoSomething();
AcquisitionAssistant::Manager::Instance(HandlerType::UGAP)->DoSomething();

// Factory: registers with ObjectViewer
//   tag "Gc.SWEAcqAssist"  → OVObject.cpp:222
//   tag "Gc.UGAPAcqAssist" → OVObject.cpp:225

// Handler interface
class IAcqAssistHandler {
    virtual void OnModeEnter() = 0;
    virtual void OnModeExit()  = 0;
    // ...
};
class SWEAcqAssistHandler  : public IAcqAssistHandler { ... };
class UGAPAcqAssistHandler : public IAcqAssistHandler { ... };
```

## Why It Exists

`ESMain.cpp` coordinates the scanner at a high level — it must not contain `if (SWE) ... else if (UGAP) ...` branches scattered across 10k lines. The Manager abstraction collapses all modality-specific routing to one place; the Factory ensures ObjectViewer can instantiate the right rendering object without knowing the C++ type.

## Key Points

- `Manager::Instance()` is the only way `ESMain.cpp` touches AcquisitionAssistant logic
- Factory tags `"Gc.SWEAcqAssist"` / `"Gc.UGAPAcqAssist"` in `OVObject.cpp:222,225` are **load-bearing strings** — rename them and registration silently breaks
- `IAcqAssistHandler` is the only interface — SWE and UGAP each have exactly one implementation
- `ObjectViewerImpl.cpp:2063,2098` uses `dynamic_cast` to dispatch to the correct concrete type after factory creation
