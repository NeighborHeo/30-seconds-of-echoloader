---
title: "Layer A vs Layer B: The Two AcquisitionAssistant Layers"
category: architecture
tags: [acquisition-assistant, layering, swe, ugap, gc-params]
difficulty: intermediate
---

AcquisitionAssistant is split across two physically separate layers — Layer A in `EchoScanner` (logic) and Layer B in `GcViewer` (display) — connected only through GC parameter strings.

## Structure

```
EchoScanner/AcquisitionAssistant/   ← Layer A (logic)
├── AcquisitionAssistant.h          (~1,324 lines)
├── AcquisitionAssistant.cpp
│   ├── Manager (singleton)
│   ├── Factory
│   ├── IAcqAssistHandler (interface)
│   ├── SWEAcqAssistHandler
│   └── UGAPAcqAssistHandler

GcViewer/ObjectViewer/              ← Layer B (display)
├── AcquisitionAssistantBase.h/cpp  (926 + 277 lines)
├── SWEAcquisitionAssistant.*
└── UGAPAcquisitionAssistant.*

         Layer A ──[GC param strings]──▶ Layer B
         (no direct C++ calls across the boundary)
```

## Why It Exists

The GC parameter bus (string/variant pairs exchanged via `ESMain::GetParameterValue()` / `SetParameterValue()`) is the intentional firewall: it lets the rendering layer evolve independently of acquisition logic, and it matches how the broader GcViewer framework communicates with all ObjectViewer objects.

## Key Points

- Layer A entry: `ESMain.cpp` — 11 call sites at lines 7066–9954
- Layer B entry: ObjectViewer factory tags `"Gc.SWEAcqAssist"` / `"Gc.UGAPAcqAssist"` (`OVObject.cpp:222,225`)
- The boundary is **GC parameter strings only** — Layer B never calls back into `EchoScanner` C++ directly
- Layer B also handles: rendering, UI overlays, ROI, LiverAI — none of that logic belongs in Layer A
