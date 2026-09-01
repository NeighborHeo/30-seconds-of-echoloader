---
title: "Phase Marker Traceability"
category: cpp
tags: [traceability, refactoring, comments, swe-ugap]
difficulty: intermediate
---

Refactoring work in the AcquisitionAssistant subsystem uses Phase markers tied to design document sections — they are the handshake between code and the living design doc.

## Why

The SWE/UGAP refactoring spans dozens of files across multiple sprints. Without explicit phase anchors, reviewers cannot verify that a code change matches the approved design, and incomplete phases become invisible technical debt in a regulated system.

## Pattern

```cpp
// Full reference — cross-file refactoring step
// Phase A.3, see analysis/SWEUGAPAcqAssist/14_poorprobe_reuse_design.md §2 / §6
void AcquisitionAssistant::rebuildProbeContext()
{
    // ...
}

// Inline step marker — single responsibility within a phase
// Phase B.1 — mirror 3-warning flags into LiverWarning3Panel.
m_pLiverWarning3Panel->setFlags(m_bDepthWarning, m_bGainWarning, m_bFreqWarning);

// Phase completion marker in commit message (mirrors code comment)
// [B.3-fix] Correct warning flag propagation after probe swap

// When a phase is complete, update the design doc §reference:
// Phase C.2 ✓ — analysis/SWEUGAPAcqAssist/14_poorprobe_reuse_design.md §4 updated
```

## Key Points

- Phase label format: `Phase <Letter>.<Number>` — letter = major milestone, number = sub-step
- Always include the `§` section number from the design doc — the doc section and the code must stay in sync
- Commit message prefix mirrors the phase: `[B.3-fix]`, `[A.3]`, `[C.2-complete]`
- Only required for AcquisitionAssistant refactoring work; other subsystems use ticket IDs only
