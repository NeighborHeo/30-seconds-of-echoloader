---
title: "C++11/14 Language Ceiling"
category: cpp
tags: [language-version, msvc, cpp11, cpp14, build]
difficulty: beginner
---

The codebase is hard-locked to C++11/14. C++17 and later features are prohibited — using them silently compiles on some toolchains and breaks others.

## Why

Medical device software undergoes long validation cycles. Introducing a new language revision requires re-validation of the toolchain. The team standardized on C++11/14 with MSVC; that contract is held until an explicit migration project is approved.

## Pattern

```cpp
// OK — C++11/14 features in active use
auto pCtrl = std::make_unique<AcquisitionController>();
nullptr;                          // not NULL
constexpr int MaxFrames = 512;
override;                         // on virtual overrides
for (auto& frame : m_frameList) { /* ... */ }
auto fn = [this](int x) { return x * m_scale; };
std::chrono::milliseconds(100);

// PROHIBITED — C++17 and later
// [[nodiscard]]                  // use bool return convention instead
// [[maybe_unused]]               // suppress with (void)param; cast
// std::optional<T>               // use bool+out-param pattern
// if constexpr (...)             // use regular if or template specialization
// auto [a, b] = pair;            // structured bindings
// std::string_view               // use const std::string&
```

## Key Points

- When in doubt: if it wasn't in MSVC 2015, don't use it
- `[[nodiscard]]` is replaced by code review and the `bool` return convention
- `std::optional` is replaced by the bool+out-param error pattern
- Structured bindings (`auto [x, y]`) look harmless but are C++17 — use `.first`/`.second`
