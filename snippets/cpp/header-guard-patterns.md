---
title: "Header Guard Patterns"
category: cpp
tags: [headers, include-guards, conventions, build]
difficulty: beginner
---

68% of headers use `#pragma once`; 32% use classic `#ifndef` guards — both are valid, never mix them in the same file.

## Why

The codebase grew across years and teams. New files default to `#pragma once` (simpler, faster). Legacy files keep `#ifndef` to avoid diffs that touch nothing functional. Include order is standardized to keep PCH hits reliable under MSVC.

## Pattern

```cpp
// ── New file style (preferred) ──────────────────────
// GE CONFIDENTIAL — DO NOT DISTRIBUTE
#pragma once

#include "StdAfx.h"              // 1. Precompiled header — always first
#include "AcquisitionController.h"  // 2. Project headers
#include "ScCommon/ScLogInfo.h"  // 3. ScCommon / mcd platform headers
#include <string>                // 4. STL last
#include <vector>


// ── Legacy file style (leave as-is) ─────────────────
// GE CONFIDENTIAL — DO NOT DISTRIBUTE
#ifndef _ACQUISITION_CONTROLLER_H_
#define _ACQUISITION_CONTROLLER_H_

// ... header body ...

#endif // _ACQUISITION_CONTROLLER_H_
```

## Key Points

- GE CONFIDENTIAL block goes above the guard — mandatory in every header
- `StdAfx.h` must be first include; violating this breaks the PCH and slows all builds
- Do not convert `#ifndef` guards to `#pragma once` unless you own the file — pointless churn
- STL headers last; they are already included transitively through project headers in most cases
