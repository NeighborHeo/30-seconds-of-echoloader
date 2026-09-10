---
title: "UDT 레지스트리 조회: OVObject 내에서 다른 UDT 노드 찾기"
category: udt
tags: [udt, registry, GcUdtHandle, RAII, lookup, lifecycle]
difficulty: advanced
---

`GcUdtRegistry::Get<T>("Gc.TagName")`으로 다른 UDT 노드 핸들을 조회한다. 태그는 캐시해도 되지만 핸들은 절대 멤버 변수로 보관하지 않는다.

## Why

UDT 노드는 모드 전환 시 독립적으로 활성화·비활성화된다. Render() 시점에 살아 있던 핸들이 다음 프레임엔 무효일 수 있다. 핸들을 멤버로 보관하면 상대방 `OnDeactivate()` 이후 dangling 역참조가 된다.

## Pattern

```cpp
// ── 올바른 패턴: Render() 내에서 조회 + 즉시 사용 ───────────────────────────
class SWEAcquisitionAssistant : public OVObject
{
public:
    void Render(OVGraphics::Context& ctx) override
    {
        // 1. 레지스트리에서 핸들 조회 — 매 Render()마다
        //    태그 문자열은 상수로 관리 (오타 방지)
        GcUdtHandle<LiverAIProcessor> liverAiHandle =
            GcUdtRegistry::Get<LiverAIProcessor>(k_LiverAIProcessorTag);
        //                                        ^^^^^^^^^^^^^^^^^^^^
        //                                        "Gc.LiverAIProcessor"

        // 2. IsValid() 선행 확인 — 생략 불가
        if (!liverAiHandle.IsValid())
        {
            // LiverAIProcessor가 비활성 상태 (UGAP 모드 등)
            // 묵음 처리 — 크래시 없음
            return;
        }

        // 3. 유효한 핸들 즉시 사용
        liverAiHandle->SetInputFrame(m_currentFrame_BS);
        LiverAIProcessor::Result result = liverAiHandle->RunInference();

        // 4. 핸들은 스코프 종료 시 자동 해제 (RAII)
        // liverAiHandle.Release() 호출 불필요
    }

    // ── 태그는 상수로 캐시 가능 ────────────────────────────────────────────
    // 태그 문자열은 런타임에 변하지 않음 → 상수 정의 OK
    // 핸들은 노드 수명에 따라 변함 → 멤버 변수 절대 금지

private:
    // ✅ 태그: 상수, 변하지 않음
    static const char* const k_LiverAIProcessorTag;  // = "Gc.LiverAIProcessor"

    // ❌ 핸들: 멤버 변수 금지
    // GcUdtHandle<LiverAIProcessor> m_liverAiHandle;
    // → OnDeactivate() 이후 무효화되었지만 멤버는 여전히 보유
    // → 다음 Render()에서 IsValid() 통과 못해도 Release()가 안 불린 채 남음
    // → 프레임워크 내부 참조 카운트 누수 또는 stale 접근
};

const char* const SWEAcquisitionAssistant::k_LiverAIProcessorTag =
    "Gc.LiverAIProcessor";

// ── 안티패턴: 핸들을 멤버로 보관 ────────────────────────────────────────────
class SWEAcquisitionAssistant_WRONG : public OVObject
{
public:
    bool Init(const std::string& preset, std::string& errMsg)
    {
        // Init 시점엔 IsValid() — 문제 없어 보임
        m_liverAiHandle =
            GcUdtRegistry::Get<LiverAIProcessor>("Gc.LiverAIProcessor");
        return m_liverAiHandle.IsValid();
    }

    void Render(OVGraphics::Context& ctx) override
    {
        // ❌ LiverAIProcessor::OnDeactivate()가 이미 호출됐다면
        //    m_liverAiHandle.IsValid() == false 이지만
        //    Release()가 안 불린 채 멤버에 남아 있음
        if (!m_liverAiHandle.IsValid()) return;
        m_liverAiHandle->RunInference();  // ← 운 좋으면 통과, 아니면 stale
    }

private:
    GcUdtHandle<LiverAIProcessor> m_liverAiHandle;  // ❌ stale 위험
};

// ── OnDeactivate()에서 멤버 핸들을 보유한 경우의 올바른 정리 ─────────────────
// (어쩔 수 없이 멤버로 가져야 한다면 — Init() 비용이 과도할 때만)
void SWEAcquisitionAssistant::OnDeactivate() override
{
    // 반드시 명시적 Release() — RAII만 믿으면 타이밍 문제
    m_liverAiHandle.Release();
}
```

## Key Points

- `GcUdtRegistry::Get<T>(tag)` → `IsValid()` → 사용 → 스코프 종료 RAII 해제 — 이 순서가 전체 안전 체인
- **태그(문자열 상수)는 캐시 가능, 핸들은 캐시 금지** — 태그는 상수, 핸들 유효성은 모드 전환마다 변함
- 핸들을 멤버로 보관할 경우 `OnDeactivate()`에서 반드시 `Release()` 명시 — RAII 소멸자 타이밍이 프레임워크 셧다운 순서와 충돌할 수 있음
- `Get()` 비용이 우려되면 측정 먼저 — 실제로는 해시맵 조회 수준이며 Render() 루프에서 허용 범위
- 멤버 핸들 패턴이 필요하다고 판단될 때 → `Init()` 비용이 실제로 과도한지 측정 후 결정, 기본은 Render() 내 조회
