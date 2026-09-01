---
title: "Bool + Out-Param Error Handling"
category: cpp
tags: [error-handling, conventions, api-design]
difficulty: beginner
---

`bool FunctionName(std::string& errorMsg)` is the universal error contract in gipc-app — returns `true` on success, fills `errorMsg` on failure.

## Why

C++ exceptions are not used in application code. HRESULT is reserved for COM interfaces. A plain `bool` return with an out-param string gives callers a consistent, zero-overhead way to check and surface errors without exception overhead or COM ceremony.

## Pattern

```cpp
// Declaration
bool AuthenticateRemoteStreamingUser(std::string& errorMsg);

// Implementation
bool AuthenticateRemoteStreamingUser(std::string& errorMsg)
{
    if (!m_pConnection)
    {
        errorMsg = "Connection not initialized";
        return false;
    }
    if (!m_pConnection->Authenticate())
    {
        errorMsg = "Authentication failed for remote streaming user";
        return false;
    }
    return true;
}

// Call site
std::string errMsg;
if (!AuthenticateRemoteStreamingUser(errMsg))
{
    ScLogError("STREAMING", "Auth failed: %s", errMsg.c_str());
    return false;  // propagate up
}
```

## Key Points

- `true` = success, `false` = failure; `errorMsg` is only meaningful on `false`
- Propagate errors upward by returning `false` from the caller too — no exception unwinding
- `try`/`catch` is nearly absent; COM boundaries use `HRESULT` instead
- Log with `ScLogError` before returning `false` so the failure is always captured
