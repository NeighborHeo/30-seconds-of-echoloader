---
title: "OVGraphics::Pointer<T> — 렌더링 객체 수명 관리"
category: cpp-patterns
tags: [ovgraphics, smart-pointer, rendering, ref-counted, gcviewer]
difficulty: intermediate
---

GcViewer 렌더링 서브시스템의 리소스는 `OVGraphics::Pointer<T>` (ref-counted)로 관리한다 — `GcUdtHandle`과 혼용 금지.

## Why

GcViewer는 텍스처, 오버레이, 폴리라인 같은 렌더링 리소스를 자체 ref-count 시스템으로 추적한다.  
`std::shared_ptr`을 쓰면 GcViewer 내부 생명주기와 맞지 않아 댕글링 포인터나 이중 해제가 발생한다.  
`GcUdtHandle`은 UDT(UltraSound Domain Type) 비즈니스 객체용이고, `OVGraphics::Pointer`는 렌더링 리소스 전용이다 — 어느 포인터를 쓰는지만 봐도 객체의 계층을 알 수 있다.

## Pattern

```cpp
// --- 멤버 선언 ---
class SWEOverlayObject : public OVObject
{
    OVGraphics::Pointer<OVGraphics::Overlay>  m_pOverlay;
    OVGraphics::Pointer<OVGraphics::Texture>  m_pColorMapTex;
    // GcUdtHandle은 여기 없음 — 렌더링 리소스만 OVGraphics::Pointer
};

// --- 생성: Render() 또는 OnActivate() 안에서만 ---
void SWEOverlayObject::OnActivate()
{
    m_pOverlay = OVGraphics::Overlay::Create();   // ref-count 1
    m_pColorMapTex = OVGraphics::Texture::Create(256, 1, OVGraphics::PixelFormat::RGBA8);
}

// --- 유효성 확인: bool 변환 ---
void SWEOverlayObject::Render()
{
    if (!m_pOverlay)   // bool 변환 — nullptr이면 false
    {
        ScLogInfo("SWEOverlay", "Overlay not created, skipping render");
        return;
    }
    m_pOverlay->Draw(/* ... */);
}

// --- 해제: OnDeactivate에서 Reset() 필수 ---
void SWEOverlayObject::OnDeactivate()
{
    m_pOverlay.Reset();       // ref-count 감소, 0이면 소멸
    m_pColorMapTex.Reset();
    // Reset() 없이 소멸자에만 맡기면 GcViewer 셧다운 순서와 충돌 가능
}

// --- 렌더링 외부에서 임시 사용 시 ---
{
    OVGraphics::Pointer<OVGraphics::Polyline> pLine = OVGraphics::Polyline::Create();
    pLine->AddPoint(/* ... */);
    pLine->Draw();
    // 블록 탈출 시 자동 Reset (ref-count 0 → 소멸)
}
```

## Key Points

- `OVGraphics::Pointer<T>`는 GcViewer 전용 ref-counted 포인터 — `std::shared_ptr`, `GcUdtHandle`과 교환 불가.
- 멤버로 보유할 때는 `OnDeactivate()`에서 `Reset()` 명시 필수 — GcViewer 셧다운 순서 보장.
- 유효성 검사는 `if (m_pOverlay)` bool 변환만 사용 — nullptr 비교나 `.get()` 호출 불필요.
- 생성은 Render/OnActivate 안에서만 — Render 루프 외부에서 매 프레임 Create() 호출 시 리소스 누수.
- `GcUdtHandle` vs `OVGraphics::Pointer`: 전자는 도메인 객체(SWE ROI, 측정값), 후자는 렌더링 리소스(텍스처, 오버레이).
