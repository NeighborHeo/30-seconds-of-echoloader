---
title: "ScLog Module-Key Logging"
category: cpp
tags: [logging, debugging, conventions]
difficulty: beginner
---

Always pass a module key as the first argument to every log call — it is the only mechanism for filtering logs by subsystem in a system with three coexisting log backends.

## Why

gipc-app runs three logging systems (`ILogSimple.h`, `DBLogger.h`, `LogShim.h`). `printf`/`cout` output goes nowhere useful on deployed hardware. The module key is what triage engineers use to isolate a subsystem's log stream from millions of lines of output.

## Pattern

```cpp
#include "ScCommon/ScLogInfo.h"

// Info — normal flow
ScLogInfo("SHAPEMI", "Shape migration started, count=%d", m_shapeCount);

// Error — unexpected failure
ScLogError("STREAMING", "Failed to connect to remote host: %s", host.c_str());

// Warning — degraded but continuing
ScLogWarning("ACQASST", "Probe not recognized, using default gain table");

// Debug (stripped in release builds)
ScLogDebug("IMGBUF", "Frame %d buffered, size=%zu", frameId, bufSize);

// WRONG — not captured on deployed system
printf("shape count: %d\n", m_shapeCount);
std::cout << "streaming error" << std::endl;
```

## Key Points

- Module key is ALL_CAPS, 4–10 chars, matches the subsystem abbreviation used in JIRA/tickets
- `%s` on `std::string` requires `.c_str()` — these are printf-style format strings
- `ScLogError` auto-increments an error counter visible in the system health dashboard
- One module key per subsystem — do not invent new keys; check existing headers first
