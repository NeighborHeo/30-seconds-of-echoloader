---
title: "CRTP: Render() 핫 패스의 제로 오버헤드 정적 다형성"
category: patterns
tags: [crtp, template, static-polymorphism, OVObject, render, performance, cpp11]
difficulty: advanced
---

가상 함수 디스패치 비용이 측정 가능한 Render() 핫 패스에서, CRTP(Curiously Recurring Template Pattern)로 vtable 오버헤드 없이 다형성을 구현한다.

## Why

초음파 Render() 루프는 프레임당 수천 번 호출된다. `virtual Render()`의 간접 호출 비용(vtable 조회 + icache miss)이 누적되면 실시간 렌더링 레이턴시에 영향을 준다. CRTP는 컴파일 타임에 구체 타입을 결정하므로 vtable이 없고, 인라인 최적화도 가능하다. 단, 타입이 컴파일 타임에 고정되어야 하므로 런타임 교체(Strategy 패턴)가 필요한 곳에는 쓰지 말 것.

## Pattern

```cpp
// === CRTP 베이스: 파생 타입을 템플릿 인자로 받는다 ===
template <typename TDerived>
class OVObjectBase
{
public:
    // public non-virtual — 모든 호출은 여기로
    void Render(RenderContext& ctx)
    {
        // pre: 공통 클리핑/변환 설정
        ctx.PushTransform(m_transform);

        // 정적 디스패치 — 가상 함수 없음, 인라인 가능
        static_cast<TDerived*>(this)->DoRender(ctx);

        // post: 공통 정리
        ctx.PopTransform();
    }

    void Update(const FrameData& data)
    {
        static_cast<TDerived*>(this)->DoUpdate(data);
    }

protected:
    Transform m_transform;
};

// === Concrete A: SWE 컬러맵 오브젝트 ===
class SWEColorMapObject : public OVObjectBase<SWEColorMapObject>
{
public:
    // DoRender는 public이지만, 외부에서 직접 호출하지 않는다
    // (베이스의 Render()를 통해서만 접근)
    void DoRender(RenderContext& ctx)
    {
        // SWE 전용: 속도 컬러맵 그리기
        ctx.DrawColorMap(m_sweColorTable, m_velocityBuffer);
    }

    void DoUpdate(const FrameData& data)
    {
        m_velocityBuffer = data.GetSWEVelocity();
    }

private:
    SWEColorTable  m_sweColorTable;
    VelocityBuffer m_velocityBuffer;
};

// === Concrete B: UGAP 엘라스토그램 오브젝트 ===
class UGAPElastogramObject : public OVObjectBase<UGAPElastogramObject>
{
public:
    void DoRender(RenderContext& ctx)
    {
        // UGAP 전용: 조직 경도 맵 그리기
        ctx.DrawElastogram(m_stiffnessMap, m_colorLUT);
    }

    void DoUpdate(const FrameData& data)
    {
        m_stiffnessMap = data.GetUGAPStiffness();
    }

private:
    StiffnessMap m_stiffnessMap;
    ElastoColorLUT m_colorLUT;
};

// === 컴파일 타임 다형성 사용 ===
// 타입이 고정된 컨텍스트 (예: 렌더 파이프라인 템플릿 함수)
template <typename TObject>
void RenderObject(OVObjectBase<TObject>& obj, RenderContext& ctx)
{
    obj.Render(ctx);  // vtable 없이 TObject::DoRender() 직접 호출
}

// === C++11 제약: if constexpr 없음 → enable_if 또는 태그 디스패치 ===
// 타입별로 다른 초기화가 필요할 때:

// 태그 디스패치 (C++11 친화적)
struct SWETag  {};
struct UGAPTag {};

template <typename TObject>
void InitializeObject(OVObjectBase<TObject>& obj, SWETag)
{
    // SWE 전용 초기화
    ScCommon::Logger::Info("Initializing SWE object");
}

template <typename TObject>
void InitializeObject(OVObjectBase<TObject>& obj, UGAPTag)
{
    // UGAP 전용 초기화
    ScCommon::Logger::Info("Initializing UGAP object");
}

// enable_if (C++11, 타입 특성으로 분기)
template <typename TObject>
typename ScCommon::EnableIf<ScCommon::IsSWEObject<TObject>::value, void>::type
ConfigureColorMap(OVObjectBase<TObject>& obj)
{
    // SWE 오브젝트에만 컴파일됨
}

// === 렌더 루프: 가상 함수 없이 인라인 최적화 ===
void RenderPipeline::ExecuteHotPath(RenderContext& ctx, const FrameData& data)
{
    // 타입 고정 → 루프 안에서 vtable 조회 없음
    m_sweColorMap.Update(data);
    m_sweColorMap.Render(ctx);   // 인라인 가능

    m_ugapElasto.Update(data);
    m_ugapElasto.Render(ctx);    // 인라인 가능
}

// 멤버 선언 (타입 고정, 런타임 교체 불필요한 경우)
SWEColorMapObject   m_sweColorMap;
UGAPElastogramObject m_ugapElasto;
```

## Key Points

- **적용 기준**: Render() 루프처럼 타입이 컴파일 타임에 결정되고, 프로파일링으로 `virtual` 비용이 실측된 경우에만 사용. 먼저 `virtual`로 작성하고, 병목이 확인되면 CRTP로 전환하라 — 가독성 비용이 크다.
- **C++11 제약 — `if constexpr` 없음**: 타입별 분기는 함수 템플릿 특수화, `enable_if`, 또는 태그 디스패치로 처리. `if constexpr`는 C++17이므로 MSVC/gipc-app 빌드에서 사용 불가.
- **CRTP는 런타임 교체 불가**: `OVObjectBase<SWEColorMapObject>`와 `OVObjectBase<UGAPElastogramObject>`는 공통 베이스가 없다 (타입이 다름). 런타임에 SWE↔UGAP을 교체해야 한다면 Strategy + virtual 패턴을 쓰라.
- **`DoRender()` 접근 제한**: `DoRender()`를 `public`으로 두면 외부에서 직접 호출하여 pre/post를 우회할 수 있다. NVI와 조합하거나 명명 규칙(`Do` 접두사)으로 "직접 호출 금지" 의도를 표시하라.
- **`static_cast` 안전성**: CRTP의 `static_cast<TDerived*>(this)`는 `TDerived`가 실제로 `OVObjectBase<TDerived>`를 상속했을 때만 유효하다. 잘못된 타입 인자는 UB — 상속 구조 검증에 `static_assert(sizeof(TDerived) > 0, ...)` 등을 활용하라.
