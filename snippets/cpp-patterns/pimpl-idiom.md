---
title: "PIMPL 이디엄으로 컴파일 의존성 끊기"
category: cpp-patterns
tags: [pimpl, compilation, abi, unique_ptr, forward-declaration]
difficulty: intermediate
---

구현 세부사항을 `Impl` 구조체 뒤로 숨겨 ~50개 패키지 간 컴파일 의존성을 끊는다.

## Why

gipc-app은 EchoScanner, GcViewer 등 거대한 클래스가 패키지 경계를 넘어 헤더를 포함한다.
헤더 하나를 수정하면 수백 개 번역 단위(TU)가 재컴파일된다.
PIMPL을 쓰면 헤더는 forward declaration만 노출하고, 구현은 `.cpp`에 격리된다.

- **컴파일 타임 단축**: 헤더 변경이 구현 파일 변경으로 흡수됨
- **ABI 안정성**: 멤버 추가/제거가 헤더 변경 없이 가능
- **패키지 격리**: 외부 패키지가 내부 타입(`ScAcquisition*` 등)을 몰라도 됨

## Pattern

```cpp
// EchoScanner.h  ─ 외부 패키지에 노출되는 헤더
#pragma once
#include <memory>   // std::unique_ptr (C++11)

// forward declaration — #include "ScAcquisitionEngine.h" 불필요
class ScAcquisitionEngine;
class ScBeamformer;

class EchoScanner {
public:
    EchoScanner();
    ~EchoScanner();   // Impl 소멸자를 알아야 하므로 .cpp에 정의

    void StartAcquisition(int nMode);
    bool IsScanning() const;

private:
    struct Impl;                   // 선언만
    std::unique_ptr<Impl> m_pImpl; // C++11: unique_ptr이 소유권 명확화
};
```

```cpp
// EchoScanner.cpp  ─ 실제 구현 격리
#include "EchoScanner.h"
#include "ScAcquisitionEngine.h"   // 이 헤더는 .cpp에서만 보임
#include "ScBeamformer.h"

struct EchoScanner::Impl {
    ScAcquisitionEngine m_engine;
    ScBeamformer        m_beamformer;
    bool                m_bScanning = false;
};

EchoScanner::EchoScanner()
    : m_pImpl(new Impl())   // make_unique는 C++14
{}

EchoScanner::~EchoScanner() = default;  // Impl 완전 정의 후 소멸자 확정

void EchoScanner::StartAcquisition(int nMode)
{
    m_pImpl->m_engine.Start(nMode);
    m_pImpl->m_bScanning = true;
}

bool EchoScanner::IsScanning() const
{
    return m_pImpl->m_bScanning;
}
```

## Key Points

- `~EchoScanner()` 를 헤더에서 `= default` 로 쓰면 컴파일 오류: 소멸자 시점에 `Impl`이 incomplete type이므로 반드시 `.cpp`에 정의
- C++11 환경: `std::make_unique`가 없으므로 `new Impl()`을 직접 사용하거나 헬퍼 래퍼 준비
- 복사 생성자/대입 연산자는 PIMPL 클래스에서 기본 삭제됨 (`unique_ptr`의 `= delete` 전파) — 필요하면 deep copy 구현 또는 `= delete` 명시
- **컴파일 타임 vs 바이너리 호환성**: PIMPL은 TU 재컴파일을 막지만 Impl 레이아웃이 바뀌면 재링크는 필요. 공유 DLL 경계를 넘으면 ABI 단절 방지 효과가 극대화됨
- GE 코드베이스에서 `m_pImpl` 멤버가 이미 이 관용구를 따르고 있으면 동일 패턴으로 확장하는 것이 일관성 유지에 유리
