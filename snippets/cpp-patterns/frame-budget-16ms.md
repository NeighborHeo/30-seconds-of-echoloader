---
title: "16ms 프레임 버짓: 실시간 초음파 렌더링"
category: cpp-patterns
tags: [render, performance, timing, real-time]
difficulty: intermediate
---

60fps 디스플레이에서 Render()는 반드시 16ms 이내에 완료되어야 한다. 초과 시 라이브 스캔 중 가시적 랙이 발생한다.

## Why

GC 렌더 스레드는 모든 OVObject를 순차적으로 호출한다. 하나의 Render()가 늦으면 해당 프레임 전체가 드롭된다. 16.67ms(60fps) 예산 중 Render()에 허용되는 시간은 8ms이며, SetParameter()는 2ms, 나머지는 OS 스케줄링 오버헤드로 쓰인다.

| 슬롯 | 허용 시간 |
|------|----------|
| Render() | ≤ 8ms |
| SetParameter() | ≤ 2ms |
| OS 오버헤드 | ~6ms |
| 합계 | 16.67ms |

## Pattern

```cpp
// GE naming conventions:
// PascalCase classes, m_camelCase members, PascalCase methods
// ScCommon::ScopedLock (not std::lock_guard)
// ScCommon::Mutex (not std::mutex)
// NULL not nullptr
// No auto for non-trivial types

class MySweOvObject : public OVObject
{
public:
    // BAD: 이런 것들을 Render() 안에서 하지 말 것
    // - new / delete (heap allocation)
    // - 파일 I/O, 네트워크 호출
    // - ScCommon::Mutex 잠금 경합
    // - std::string key("SWEAcqAssist.State") 생성 후 GC 파라미터 조회

    void OnActivate() override
    {
        // 초기화 단계에서 미리 계산하고 캐싱
        m_nRenderWidth  = GetCanvasWidth();
        m_nRenderHeight = GetCanvasHeight();
        m_pPixelBuffer  = m_FramePool.Acquire(); // 미리 할당된 버퍼
    }

    void Render(ScCommon::IRenderContext* pCtx) override
    {
        ScCommon::Timer timer;
        timer.Start();

        // 캐싱된 값 사용 — 조회 없음, 할당 없음
        DrawFrame(pCtx, m_pPixelBuffer, m_nRenderWidth, m_nRenderHeight);

        const long long nElapsedMs = timer.GetElapsedMs();
        if (nElapsedMs > 8)
        {
            ScLogWarn("MySweOvObject::Render() overran budget: %lldms", nElapsedMs);
        }
    }

private:
    int    m_nRenderWidth;   // OnActivate()에서 캐시
    int    m_nRenderHeight;  // OnActivate()에서 캐시
    Frame* m_pPixelBuffer;   // 미리 할당된 프레임 버퍼
    FramePool m_FramePool;
};
```

## Key Points

- Render() 안에서 heap alloc, 파일 I/O, mutex 경합은 프레임 드롭의 직접 원인이다
- 연산 결과는 OnActivate()나 SetParameter()에서 미리 계산하고 멤버에 캐싱한다
- ScCommon::Timer로 오버런을 측정하고, 8ms 초과 시 ScLogWarn으로 기록한다
- GC 파라미터 조회(string key 생성 포함)는 Render()가 아닌 SetParameter()에서 수행한다
- 24/7 운영 장비이므로 한 번의 오버런이 아닌 지속적 패턴을 로그로 추적해야 한다
