---
title: "OVObject Inheritance: How Display Objects Are Structured"
category: architecture
tags: [ovobject, gcviewer, objectviewer, rendering, inheritance]
difficulty: beginner
---

`OVObject` is the base class for everything rendered in `GcViewer` — `AcquisitionAssistantBase` inherits from it to get the rendering lifecycle, parameter bus, and factory registration for free.

## Structure

```
OVObject                          (GcViewer/ObjectViewer — base)
  └── AcquisitionAssistantBase    (926 + 277 lines)
        ├── SWEAcquisitionAssistant
        └── UGAPAcquisitionAssistant

Key capabilities inherited from OVObject:
  SetParameter()     — receive GC param updates from Layer A
  ChangeState()      — state machine transitions
  OVRendering        — draw primitives
  OVGraphics::Pointer — handle to graphic objects in the scene
```

## Why It Exists

The `ObjectViewer` framework owns the rendering loop; any object that needs to draw on screen must register via the factory and implement the `OVObject` lifecycle. Inheriting rather than wrapping means `AcquisitionAssistantBase` gets `SetParameter()` calls automatically whenever the GC param bus fires — no polling, no glue code.

## Key Points

- `AcquisitionAssistantBase` **is** an `OVObject` — it lives entirely inside `GcViewer`, isolated from `EchoScanner`, `EchoMeasure`, and `EchoWorksheet`
- `OVRendering` and `OVGraphics` provide all drawing primitives; `OVGraphics::Pointer` is the handle type for scene objects
- State transitions flow through `ChangeState()` — Layer A triggers them by writing GC params, not by calling C++ methods directly
- Factory registration (via tag strings) is the only link between the ObjectViewer framework and this class hierarchy
