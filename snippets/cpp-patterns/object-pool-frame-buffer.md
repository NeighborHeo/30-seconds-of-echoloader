---
title: "오브젝트 풀: 프레임 버퍼 재사용"
category: cpp-patterns
tags: [object-pool, memory, performance, frame-buffer]
difficulty: intermediate
---

초음파 프레임마다 `new Frame()`을 호출하면 30fps × 대형 픽셀 배열 = 1~3ms의 할당 스파이크와 24/7 운영 시 힙 단편화가 발생한다.

## Why

실시간 스캔에서 프레임은 초당 30회 생성·파기된다. 운영 시간이 길어질수록 힙 단편화로 할당 시간이 증가한다. 파이프라인 깊이(통상 트리플 버퍼링 = 3)만큼 고정 크기 풀을 미리 할당하면 Render() 경로에서 동적 할당이 사라진다.

## Pattern

```cpp
// GE naming conventions:
// PascalCase classes, m_camelCase members, PascalCase methods
// ScCommon::ScopedLock (not std::lock_guard)
// ScCommon::Mutex (not std::mutex)
// NULL not nullptr
// No auto for non-trivial types

static const int POOL_SIZE = 3; // 트리플 버퍼링 기준 파이프라인 깊이

class FramePool
{
public:
    FramePool() : m_nFreeCount(POOL_SIZE)
    {
        for (int i = 0; i < POOL_SIZE; ++i)
        {
            m_bFree[i] = true;
        }
    }

    Frame* Acquire()
    {
        ScCommon::ScopedLock lock(m_Mutex);
        for (int i = 0; i < POOL_SIZE; ++i)
        {
            if (m_bFree[i])
            {
                m_bFree[i] = false;
                return &m_Frames[i];
            }
        }
        // ponytail: O(n) scan, n=3이므로 충분. 풀 크기가 커지면 free-list로 교체
        return NULL; // 풀 소진 — 호출자가 처리
    }

    void Release(Frame* pFrame)
    {
        ScCommon::ScopedLock lock(m_Mutex);
        for (int i = 0; i < POOL_SIZE; ++i)
        {
            if (&m_Frames[i] == pFrame)
            {
                m_bFree[i] = true;
                return;
            }
        }
    }

private:
    Frame          m_Frames[POOL_SIZE]; // 스택/멤버에 미리 할당
    bool           m_bFree[POOL_SIZE];
    int            m_nFreeCount;
    ScCommon::Mutex m_Mutex;
};

// RAII 래퍼: 스코프 종료 시 자동 반환
class ScopedFrame
{
public:
    ScopedFrame(FramePool& pool) : m_Pool(pool), m_pFrame(pool.Acquire()) {}
    ~ScopedFrame() { if (m_pFrame) m_Pool.Release(m_pFrame); }

    Frame* Get() const { return m_pFrame; }

private:
    FramePool& m_Pool;
    Frame*     m_pFrame;

    // 복사 금지
    ScopedFrame(const ScopedFrame&);
    ScopedFrame& operator=(const ScopedFrame&);
};

// 사용 예
void AcquisitionCallback(FramePool& pool, const RawScanData& data)
{
    ScopedFrame scopedFrame(pool); // 풀에서 획득
    if (!scopedFrame.Get()) { return; } // 풀 소진 처리

    FillFrameData(scopedFrame.Get(), data);
    SubmitForRender(scopedFrame.Get());
    // 스코프 종료 시 자동 Release()
}
```

## Key Points

- 풀 크기는 파이프라인 깊이(트리플 버퍼링 = 3)와 일치시킨다
- ScopedFrame RAII 래퍼로 Release() 누락을 방지한다
- 풀 소진(`Acquire()` → NULL) 시 드롭/스킵 처리를 명시적으로 구현해야 한다
- `new Frame()` per 콜백 대비: 힙 단편화 없이 24/7 장기 운영이 가능하다
- 프레임 크기가 변하는 경우(모드 전환) OnActivate()에서 풀을 재초기화한다
