---
title: "Composition over Inheritance: LiverWarning3Panel"
category: architecture
tags: [composition, refactoring, liverwaring3panel, swe, ugap, ovobject]
difficulty: advanced
---

`LiverWarning3Panel` is being extracted from `AcquisitionAssistantBase` as a composed member, so warning logic can be reused without inheriting the full base class (which drags in LiverAI, ROI, AutoPos, and everything else).

## Structure

```cpp
// BEFORE (inheritance — callers must subclass Base to get warnings)
class AcquisitionAssistantBase : public OVObject {
    // warning subsystem tangled into base alongside ROI, LiverAI, AutoPos...
};

// AFTER (composition — warnings are a separate member)
class LiverWarning3Panel { ... };  // standalone, composable

class SWEAcquisitionAssistant : public OVObject {
    LiverWarning3Panel m_warningPanel;  // composed in, not inherited
};
class UGAPAcquisitionAssistant : public OVObject {
    LiverWarning3Panel m_warningPanel;  // same, independent instance
};
```

## Why It Exists

The existing design forces any feature that needs 3-warning display to inherit from `AcquisitionAssistantBase`, which pulls in the entire base class surface area. Composition breaks that coupling: `LiverWarning3Panel` becomes a leaf object that any `OVObject` subclass can hold as a member.

## Key Points

- Refactoring phases A.2–B.3 are complete (skeleton + mirror); phases B.4–B.6 (authority flip) are pending
- The authority flip is the critical step — until it lands, both the old base and the new panel co-own warning state
- Once complete, SWE and UGAP can each be `OVObject + LiverWarning3Panel` with no shared base class at all
- Do not add new logic to `AcquisitionAssistantBase` during this transition — the base is shrinking, not growing
