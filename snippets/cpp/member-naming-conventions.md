---
title: "Member Naming Conventions"
category: cpp
tags: [naming, conventions, style]
difficulty: beginner
---

gipc-app follows a strict naming scheme: PascalCase classes, camelCase methods, `m_` member vars, `s_` statics — deviating breaks grep patterns used across 40k files.

## Why

Consistent prefixes let developers instantly distinguish scope and type when reading unfamiliar code. The `m_b*` bool prefix and `s_` static prefix are particularly load-bearing: they appear in thousands of search patterns and code-review comments.

## Pattern

```cpp
// Class: PascalCase. MFC dialogs prefix with C.
class ApplicationTouchPanel { /* ... */ };
class CApplicationTouchPanel : public CDialog { /* ... */ };

// Methods: camelCase
void AuthenticateRemoteStreamingUser();
bool getNeedDrawAgain() const;

// Member variables: m_ prefix
int  m_frameCount;
bool m_bNeedDrawAgain;          // bool: m_b*
bool m_bIsStreamingActive;

// Static members: s_ prefix
static ApplicationTouchPanel* s_applicationTouchPanel;

// Constants: constexpr PascalCase
constexpr int MaxRetryCount = 3;

// Files: match class name exactly
// ApplicationTouchPanel.h / ApplicationTouchPanel.cpp
```

## Key Points

- `m_b` prefix on every `bool` member — no exceptions in new code
- MFC dialog classes keep the legacy `C` prefix (`CMainFrame`, `CApplicationTouchPanel`)
- Static members use `s_` so they're immediately distinguishable from instance state
- File names must match the primary class name exactly — one class per file pair
