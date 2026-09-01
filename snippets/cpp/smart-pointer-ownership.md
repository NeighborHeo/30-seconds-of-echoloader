---
title: "Smart Pointer Ownership"
category: cpp
tags: [memory, ownership, raii, smart-pointers]
difficulty: beginner
---

`std::unique_ptr` covers ~95% of heap allocations in gipc-app; raw `new/delete` appears only in COM/MFC legacy code.

## Why

RAII is enforced throughout the codebase. Explicit `delete` is a code-smell in new code — leaks are silent in a long-running medical imaging process. Domain-specific handle types wrap objects that don't fit standard ownership.

## Pattern

```cpp
// Exclusive ownership — most common
std::unique_ptr<AcquisitionController> m_pAcqController;
m_pAcqController = std::make_unique<AcquisitionController>();

// Shared lifetime (multiple owners)
std::shared_ptr<ImageBuffer> m_pSharedBuffer;

// Domain handle: UDT object reference (not owning)
GcUdtHandle m_udtHandle;

// Graphics subsystem pointer (ref-counted internally)
OVGraphics::Pointer<OVGraphics::Texture> m_pTexture;

// Legacy COM/MFC — raw pointer with explicit release
CDialog* m_pDlg = nullptr;   // MFC manages lifetime
// RBUG123456: must Release() before reassigning COM ptr
IStream* m_pStream = nullptr;
```

## Key Points

- `make_unique<T>()` over `new` — single allocation, exception-safe
- `shared_ptr` only when lifetime genuinely spans multiple owners; prefer `unique_ptr` by default
- `GcUdtHandle` is a non-owning reference to a UDT node — do not `delete` it
- Raw `new/delete` in new code signals a COM or MFC constraint; add a comment explaining why
