---
title: "GcUdtHandle 커스텀 핸들 패턴"
category: cpp-patterns
tags: [GcUdtHandle, handle, RAII, OVGraphics, GcViewer]
difficulty: advanced
---

GE 프레임워크 객체는 `std::unique_ptr`/`shared_ptr`로 커버되지 않으므로 `GcUdtHandle`/`OVGraphics::Pointer` 같은 GE 전용 핸들 타입으로 소유권을 표현한다.

## Why

GcViewer 레이어의 렌더링 객체는 GE 내부 프레임워크가 생명주기를 관리한다. 표준 스마트 포인터를 사용하면 이중 해제 또는 프레임워크 모르게 삭제되는 버그가 발생한다. 원시 포인터를 그대로 쓰면 소유권이 불명확해진다. GE 전용 핸들 타입이 RAII 원칙을 유지하는 올바른 방법이다.

## Pattern

```cpp
#include <GcViewer/GcUdtHandle.h>
#include <OVGraphics/Pointer.h>

// GcUdtHandle: UDT(User-Defined Type) 객체에 대한 GE 전용 핸들
// 프레임워크가 소유권을 관리 — 핸들이 스코프를 벗어나면 프레임워크에 반환
class SweOverlayRenderer {
public:
    void Initialize(GcViewer* pViewer) {
        // GcUdtHandle로 렌더링 객체 보유 (원시 포인터 금지)
        m_udtHandle = pViewer->CreateUdtObject<SweUdt>();
        // OVGraphics::Pointer로 그래픽 객체 보유
        m_graphicsPtr = OVGraphics::CreatePointer<SweGraphicsNode>();
    }

    void Render() {
        if (m_udtHandle.IsValid()) {
            m_udtHandle->Draw();
        }
        if (m_graphicsPtr) {
            m_graphicsPtr->Render();
        }
    }

    void Shutdown() {
        // 핸들 해제 — 프레임워크에 반환
        m_udtHandle.Release();
        m_graphicsPtr.Reset();
    }

private:
    GcUdtHandle<SweUdt>          m_udtHandle;    // UDT 객체 핸들
    OVGraphics::Pointer<SweGraphicsNode> m_graphicsPtr; // 그래픽 객체 핸들
};

// 잘못된 패턴 (금지)
class BadRenderer {
    SweUdt* m_pUdt;  // 원시 포인터 — 소유권 불명확, 이중 해제 위험
};

// GcUdtHandle 유효성 확인
void UseHandle(GcUdtHandle<SweUdt>& handle) {
    if (!handle.IsValid()) return;  // nullptr 역참조 방지
    handle->DoWork();
}
```

## Key Points

- `GcUdtHandle<T>`: GE 프레임워크가 소유권을 관리하는 객체에 사용. 프레임워크 모르게 `delete` 하면 이중 해제
- `OVGraphics::Pointer<T>`: 그래픽 렌더링 객체(GcViewer 레이어) 전용 핸들
- 두 타입 모두 RAII: 스코프 종료 또는 명시적 `Release()`/`Reset()`으로 정리
- `IsValid()` 확인 후 역참조 — 무효 핸들 역참조는 프레임워크 내부 크래시
- 원시 포인터(`SweUdt*`)를 멤버로 보유하는 패턴은 신규 코드에서 금지
- 두 핸들 타입이 커버하지 않는 GE 객체가 있다면 해당 프레임워크의 핸들 타입 확인 후 사용
