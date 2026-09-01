---
title: "Push-down vs Hoisting: The Refactoring Direction Choice"
category: architecture
tags: [refactoring, push-down, hoisting, swe, ugap, base-class]
difficulty: advanced
---

Two directions were considered for cleaning up `AcquisitionAssistantBase` — hoisting duplication up into Base was rejected; push-down into SWE/UGAP was chosen so Base can eventually become a pure "warning shell."

## Structure

```
HOISTING (rejected)
  SWEAcqAssistant ─┐
                   ├─ shared logic pulled UP into Base
  UGAPAcqAssist  ─┘
  Result: Base grows; SWE/UGAP stay coupled through a fat base

PUSH-DOWN (chosen)
  Base (warning shell only)
    ├── SWEAcqAssistant  ← non-warning SWE logic pushed DOWN here
    └── UGAPAcqAssist    ← non-warning UGAP logic pushed DOWN here
  Result: Base shrinks; LiverWarning3Panel extraction becomes clean
```

## Why It Exists

Hoisting looks like deduplication but increases coupling — every change to shared logic risks breaking both modes simultaneously. Push-down accepts short-term duplication between SWE and UGAP in exchange for independent evolvability, which matches the domain reality: SWE and UGAP are fundamentally different modalities.

## Key Points

- Push-down is safe only **after** the Warning authority flip (phases P2 + P4)
- D5-stub (7 trivial stubs, ~21 LoC) and D4 (enum extraction) can be pushed down immediately without waiting
- Deliberate duplication between SWE and UGAP is acceptable and intentional — don't merge it
- The long-term target: `AcquisitionAssistantBase` contains only warning-panel logic; everything else lives in the concrete subclasses
