---
title: "AcqAssistState Enum Duplication: The Cross-Boundary Integer"
category: architecture
tags: [enum, duplication, gc-params, sync-risk, refactoring]
difficulty: intermediate
---

`AcqAssistState` is defined in two separate files — one per layer — because the integer value must cross the GC parameter string boundary; the two definitions must stay numerically in sync or silent state corruption occurs.

## Structure

```cpp
// Layer B — GcViewer/ObjectViewer/AcquisitionAssistantBase.h  (lines 67–74)
enum AcqAssistState {
    eIdle = 0,
    eActive,
    eFreeze,
    // ...
};

// Layer A — EchoScanner/AcquisitionAssistant/AcquisitionAssistant.h  (lines 14–20)
enum AcqAssistState {   // MUST match Layer B numerically
    eIdle = 0,
    eActive,
    eFreeze,
    // ...
};
// [SI-04] Phase axis mirror... MUST stay numerically in sync

// The integer crosses the boundary as a GC param string value:
// Layer A writes: SetParameterValue("AcqAssistState", (int)eActive)
// Layer B reads:  (AcqAssistState)GetParameterValue("AcqAssistState")
```

## Why It Exists

The GC parameter bus carries variants/strings — it has no type system. The integer representation of the enum is the only shared contract. Sharing a single header across the Layer A/B boundary would introduce a direct C++ include dependency that violates the decoupling intent.

## Key Points

- If you add, remove, or reorder an enum value in one file, you **must** update the other immediately — there is no compiler guard
- The comment `// [SI-04]` marks this as a known synchronization risk in the refactoring roadmap
- Planned fix: extract both to `AcqAssistTypes.h` (single source of truth) — not yet implemented
- Until the fix lands: always edit both files together and verify values match before committing
