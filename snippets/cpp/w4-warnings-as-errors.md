---
title: "/W4 Warnings as Errors"
category: cpp
tags: [build, msvc, warnings, quality-gate]
difficulty: intermediate
---

MSVC `/W4` with 23 specific warnings promoted to errors is the sole automated quality gate — there is no CI, no clang-tidy, and no clang-format.

## Why

In the absence of a CI pipeline, the compiler warning configuration in `CppProjectBase.props` is the only systematic check that runs on every build. A warning-as-error that ships means a broken build for every developer — that friction is intentional.

## Pattern

```cpp
// These trigger promoted-to-error warnings — fix, don't suppress

// C4700: uninitialized local variable (always initialize)
int frameCount = 0;          // not: int frameCount;

// C4706: assignment within conditional
bool bReady = getReady();
if (bReady) { /* ... */ }    // not: if (bReady = getReady())

// C4715: not all control paths return a value
bool isValid(int x)
{
    if (x > 0) return true;
    return false;            // explicit — no implicit fall-off
}

// C4150: deletion of pointer to incomplete type
// → include the full header, not just a forward declaration, before delete

// C4553: '==' may have been intended (typo guard)
// → use parentheses or split the expression

// GCC fallback (.vscode / WSL builds)
// -Wall -Wextra -Wpedantic -Wshadow -Wformat=2
// -Wcast-align -Wconversion -Wsign-conversion -Wnull-dereference
```

## Key Points

- Promoted errors live in `CppProjectBase.props` — do not suppress them with `#pragma warning(disable)` without a ticket comment
- The full promoted list: 4047, 4101, 4130, 4142, 4146, 4150, 4218, 4269, 4305, 4308–4313, 4390, 4532, 4553, 4554, 4603, 4661, 4700, 4706, 4715
- Debug builds add `EnableFastChecks` (stack corruption, uninitialized variables at runtime)
- Fix the code, not the warning — suppression is a last resort and requires a `// RBUG` comment explaining why
