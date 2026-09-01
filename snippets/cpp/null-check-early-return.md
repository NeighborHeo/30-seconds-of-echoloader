---
title: "Null Check + Early Return"
category: cpp
tags: [defensive-coding, null-safety, error-handling]
difficulty: beginner
---

Check every pointer before use and return immediately on null — deep nesting hides failure paths and complicates code review in a regulated codebase.

## Why

There are no PRECONDITION/ENSURES macros and `assert()` is nearly absent (it compiles away in release builds). Defensive null checks with early returns are the only runtime guard between a null dereference and a system crash on a live patient scan.

## Pattern

```cpp
void ImageRenderer::updateOverlay(RegistryKey* regKey, OverlayConfig* config)
{
    // Guard each pointer independently — log before returning
    if (regKey == nullptr)
    {
        ScLogError("IMGRNDR", "regKey is null, cannot update overlay");
        return;
    }
    if (config == nullptr)
    {
        ScLogError("IMGRNDR", "config is null, cannot update overlay");
        return;
    }

    // Safe to dereference — all guards passed
    regKey->SetValue("overlayEnabled", config->m_bEnabled);
    m_pOverlay->apply(*config);
}

// AVOID — nested guards obscure the happy path
void bad(RegistryKey* regKey, OverlayConfig* config)
{
    if (regKey != nullptr) {
        if (config != nullptr) {
            regKey->SetValue("overlayEnabled", config->m_bEnabled);
        }
    }
}
```

## Key Points

- One guard per pointer, one `ScLogError` per guard — never silently swallow a null
- Early return keeps the happy path at the leftmost indentation level
- `assert()` is absent in production paths — a null check that only fires in debug is not a guard
- For `bool`-returning functions, return `false` (not `return;`) and fill the `errorMsg` out-param
