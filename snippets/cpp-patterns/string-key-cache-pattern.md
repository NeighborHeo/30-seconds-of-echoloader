---
title: "String Key 캐싱: 프레임당 할당 제거"
category: cpp-patterns
tags: [string, cache, performance, gc-params, memory]
difficulty: beginner
---

GC 파라미터 키(`"SWEAcqAssist.State"`)를 Render() 안에서 매번 `std::string`으로 생성하면 프레임당 heap alloc이 발생한다. 10분 라이브 스캔(30fps) = 18,000번의 쓸모없는 할당.

## Why

`std::string` 생성은 SSO(Small String Optimization)를 넘는 길이에서 malloc을 호출한다. GC 파라미터 키 문자열("SWEAcqAssist.State" = 18자)은 MSVC STL의 SSO 임계값(15자)을 넘어 매번 heap에 올라간다. Render()에서 반복되면 힙 단편화와 alloc 레이턴시가 누적된다.

## Pattern

```cpp
// GE naming conventions:
// PascalCase classes, m_camelCase members, PascalCase methods
// ScCommon::ScopedLock (not std::lock_guard)
// ScCommon::Mutex (not std::mutex)
// NULL not nullptr
// No auto for non-trivial types

// ---- BEFORE: 매 Render()마다 string 생성 (나쁜 예) ----
void Render_Bad(ScCommon::IRenderContext* pCtx)
{
    // 30fps = 초당 30번 malloc 발생
    std::string strKey("SWEAcqAssist.State");
    int nState = GetGCParam(strKey);
    DrawState(pCtx, nState);
}

// ---- AFTER 1: static const — 프로세스 수명 동안 한 번만 생성 ----
void Render_Good(ScCommon::IRenderContext* pCtx)
{
    // string 객체는 프로세스 시작 시 한 번 생성됨
    static const std::string s_strStateKey("SWEAcqAssist.State");
    int nState = GetGCParam(s_strStateKey);
    DrawState(pCtx, nState);
}

// ---- AFTER 2: 클래스 멤버 캐시 — OnActivate()에서 초기화 ----
class SweOvObject : public OVObject
{
public:
    void OnActivate() override
    {
        // Render() 호출 전 딱 한 번 초기화
        m_strStateKey   = "SWEAcqAssist.State";
        m_strAssistKey  = "SWEAcqAssist.AssistLevel";
    }

    void Render(ScCommon::IRenderContext* pCtx) override
    {
        // 할당 없음 — 멤버 참조만
        int nState = GetGCParam(m_strStateKey);
        int nLevel = GetGCParam(m_strAssistKey);
        DrawSweInfo(pCtx, nState, nLevel);
    }

private:
    std::string m_strStateKey;
    std::string m_strAssistKey;
};

// ---- AFTER 3: constexpr — 컴파일 타임 리터럴 (C++11) ----
// 단순 const char* 비교로 충분한 경우
static constexpr const char* SWE_STATE_KEY   = "SWEAcqAssist.State";
static constexpr const char* SWE_ASSIST_KEY  = "SWEAcqAssist.AssistLevel";
```

## Key Points

- MSVC STL SSO 임계값(15자)을 넘는 GC 파라미터 키는 Render()에서 생성하면 반드시 malloc이 발생한다
- `static const std::string`은 가장 단순한 해법 — 함수 내에 선언해도 프로세스 수명 동안 한 번만 생성된다
- 클래스 멤버 `std::string`은 OnActivate()에서 초기화하면 모드 전환 시 키를 바꿀 수 있어 유연하다
- `constexpr const char*`는 문자열 비교가 필요 없는 API(포인터를 key로 받는 경우)에서 최적이다
- 10분 30fps 기준 18,000회 할당 제거 → heap 단편화 누적 없이 24/7 운영 안정성 향상
