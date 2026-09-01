---
title: "External Contracts: Four Locations You Must Not Break"
category: architecture
tags: [contracts, refactoring-safety, factory-tags, manager-api, dynamic-cast]
difficulty: intermediate
---

Four specific locations define the public contract of `AcquisitionAssistant` — changing any of them silently breaks the system; internal implementations are free to evolve.

## Structure

```
1. OVObject.cpp:222,225
   "Gc.SWEAcqAssist"   ← factory tag, ObjectViewer registration
   "Gc.UGAPAcqAssist"  ← factory tag, ObjectViewer registration
   → rename = factory silently returns null, no crash, no warning

2. ObjectViewerImpl.cpp:2063,2098
   dynamic_cast<SWEAcquisitionAssistant*>(...)   ← type-specific dispatch
   dynamic_cast<UGAPAcquisitionAssistant*>(...)
   → change concrete type = cast returns null, behavior silently skipped

3. ESMain.cpp (11 call sites, lines 7066–9954)
   Manager::Instance(HandlerType::SWE)->...
   Manager::Instance(HandlerType::UGAP)->...
   → change Manager API = build breaks across 11 sites

4. AcquisitionAssistant::* namespace public API
   Factory, Manager, IAcqAssistHandler
   → internal impls can change; public signatures cannot
```

## Why It Exists

These four are integration seams between independently-compiled subsystems (`GcViewer`, `EchoScanner`, `ESMain`). The factory tags are pure strings — the compiler cannot catch renames. The `dynamic_cast` sites are the only places that know the concrete type after factory creation. The Manager API is the agreed interface between `ESMain` and the entire `AcquisitionAssistant` layer.

## Key Points

- Factory tag strings (`"Gc.SWEAcqAssist"`, `"Gc.UGAPAcqAssist"`) are the most dangerous — a typo compiles cleanly and breaks at runtime
- Before any refactoring, grep for these four locations first and verify they are untouched after the change
- Internal class members, method bodies, and private helpers can be freely restructured
- Adding new `Manager` methods is safe; removing or renaming existing ones is not
