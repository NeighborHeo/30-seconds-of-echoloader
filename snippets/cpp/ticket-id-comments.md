---
title: "Ticket ID Comments"
category: cpp
tags: [traceability, comments, conventions, regulated]
difficulty: beginner
---

Every non-trivial code change must cite its ticket ID in an inline comment — traceability to requirements and bug reports is a regulatory requirement.

## Why

gipc-app is medical device software. Auditors and reviewers must be able to trace every behavioral change to a documented work item. A fix with no ticket reference is untraceable — it can block a release audit.

## Pattern

```cpp
// Single bug fix
// RBUG159787: If we booted up with HW, make sure we clear the reboot retry counter
m_rebootRetryCounter = 0;

// Feature bug + change request together
// FBUG 87799/CR 27138: Suppress gain warning during calibration phase
if (m_bInCalibration)
    return;

// Change request only
// CR 31045: Remote streaming user must re-authenticate after 30-minute idle
AuthenticateRemoteStreamingUser(errMsg);

// Multi-line explanation is fine
// RBUG201144: AcqAssistant sent a stale probe handle after config reload.
// Null-check added here because the handle is rebuilt asynchronously.
if (m_probeHandle == nullptr)
{
    ScLogError("ACQASST", "Stale probe handle after config reload");
    return false;
}
```

## Key Points

- Prefixes: `RBUG` = release bug, `FBUG` = feature bug, `CR` = change request
- Place the comment on the line immediately before the affected code — not in the function header
- No file-level revision history blocks; Git log is the authoritative history
- When a single change touches multiple tickets, combine: `// FBUG 87799/CR 27138`
