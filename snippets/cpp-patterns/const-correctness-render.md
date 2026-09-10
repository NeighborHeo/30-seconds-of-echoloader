---
title: "렌더링 파이프라인의 const 정확성"
category: cpp-patterns
tags: [const-correctness, rendering, mutable, ovobject, w4]
difficulty: intermediate
---

`Render(const RenderContext& ctx) const` 선언 하나로 렌더 경로의 상태 변이를 컴파일러가 막게 한다.

## Why

OVObject 계층구조는 렌더링 중에 객체 상태가 바뀌어서는 안 된다.
const가 없으면 렌더 스레드가 데이터를 조용히 변경하다 재현 불가능한 레이스가 생긴다.
MSVC `/W4`는 const 위반을 경고로 잡아주므로, const를 올바르게 쓰면 경고가 곧 버그 탐지기가 된다.

## Pattern

```cpp
// OVObject.h — OVObject 계층구조의 렌더 인터페이스
class OVObject {
public:
    // 렌더는 항상 const: 화면에 그리는 행위가 객체 상태를 바꿔서는 안 됨
    virtual void Render(const RenderContext& ctx) const = 0;

protected:
    // 진짜 가변 상태만 mutable 선언
    mutable ScCommon::Mutex m_renderMutex;   // 스레드 보호 — 논리적으로 const
    mutable bool            m_bCacheDirty = true;  // 지연 캐시 — 논리적으로 const
    mutable CachedGeometry  m_cachedGeom;
};
```

```cpp
// EchoOverlay.h — 구체 OVObject
class EchoOverlay : public OVObject {
public:
    void Render(const RenderContext& ctx) const override;

    // setter: const 아님 — 상태 변경은 렌더 외부에서만
    void SetGain(float fGain);
    void SetDepth(float fDepth);

    // getter: const — 읽기 전용
    float GetGain()  const { return m_fGain; }
    float GetDepth() const { return m_fDepth; }

private:
    float m_fGain  = 1.0f;
    float m_fDepth = 0.0f;

    // 지연 계산 캐시 — Render() const 내에서 갱신하므로 mutable
    mutable bool            m_bCacheDirty = true;
    mutable CachedGeometry  m_cachedGeom;
};
```

```cpp
// EchoOverlay.cpp
void EchoOverlay::Render(const RenderContext& ctx) const
{
    ScCommon::ScopedLock lock(m_renderMutex);  // mutable mutex, const 함수 내 OK

    if (m_bCacheDirty) {
        // 캐시 재계산 — mutable이므로 const 함수에서도 쓰기 가능
        m_cachedGeom = ComputeGeometry(m_fGain, m_fDepth);
        m_bCacheDirty = false;
    }

    ctx.DrawGeometry(m_cachedGeom);

    // 아래 줄은 컴파일 오류: m_fGain은 mutable이 아님
    // m_fGain = 2.0f;   // error C3490: 'const' 개체를 통해 수정할 수 없습니다
}

void EchoOverlay::SetGain(float fGain)
{
    m_fGain       = fGain;
    m_bCacheDirty = true;   // 캐시 무효화
}
```

## Key Points

- `const` 메서드에서 멤버를 수정하면 MSVC `/W4`가 C3490 오류 → 렌더 경로 실수 즉시 탐지
- **`mutable` 정당 사용 2가지**: 스레드 동기화 뮤텍스, 지연 캐시. 그 외에 `mutable`을 쓰면 const의 의미가 무너짐
- `const RenderContext& ctx`: 렌더 컨텍스트도 const 참조로 받아야 렌더러 쪽 상태도 보호됨
- OVObject 계층에서 `Render` 를 non-const로 선언하면 부모 인터페이스와 시그니처가 달라 `override` 컴파일 오류 발생 → 계층 전체를 const로 강제 유지하는 자연 방어막
- 렌더 스레드와 업데이트 스레드를 분리한 아키텍처에서 const가 없으면 렌더 스레드가 암묵적으로 공유 상태를 건드릴 수 있음; const + mutex 조합이 올바른 패턴
