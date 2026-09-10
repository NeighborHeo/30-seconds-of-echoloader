---
title: "Lock-free 상태 플래그: atomic으로 렌더 스레드 통신"
category: cpp-patterns
tags: [atomic, lock-free, threading, render-thread, priority-inversion]
difficulty: advanced
---

렌더 스레드(고우선순위)와 취득 스레드(중간우선순위)가 단일 bool을 공유할 때 Mutex를 쓰면 우선순위 역전이 발생한다. `std::atomic<bool>`이 올바른 도구다.

## Why

렌더 스레드는 GC에서 가장 높은 우선순위로 돌아간다. Mutex로 단순 bool을 보호하면 낮은 우선순위 스레드가 락을 잡은 채 선점될 때 렌더 스레드가 블로킹되는 우선순위 역전이 생긴다. `std::atomic`은 잠금 없이 가시성을 보장한다.

규칙: **POD 상태 플래그는 atomic, 여러 필드가 함께 일관성을 유지해야 하는 복합 상태는 Mutex.**

## Pattern

```cpp
// GE naming conventions:
// PascalCase classes, m_camelCase members, PascalCase methods
// ScCommon::ScopedLock (not std::lock_guard)
// ScCommon::Mutex (not std::mutex)
// NULL not nullptr
// No auto for non-trivial types

// ---- 잘못된 방법: bool 하나에 Mutex ----
// ScCommon::Mutex m_Mutex;
// bool m_bSweActive; // 렌더 스레드가 락 대기로 블로킹될 수 있음

// ---- 올바른 방법: atomic<bool> ----
class SweOvObject : public OVObject
{
public:
    SweOvObject() : m_bSweActive(false), m_eAcqState(STATE_IDLE) {}

    // 취득 스레드에서 호출
    void OnSweStarted()
    {
        m_bSweActive.store(true, std::memory_order_release);
    }

    void OnSweStopped()
    {
        m_bSweActive.store(false, std::memory_order_release);
    }

    // 렌더 스레드에서 호출 — 락 없음
    void Render(ScCommon::IRenderContext* pCtx) override
    {
        if (m_bSweActive.load(std::memory_order_acquire))
        {
            RenderSweOverlay(pCtx);
        }

        // 상태 enum 예시
        const AcqState eState = static_cast<AcqState>(
            m_eAcqState.load(std::memory_order_acquire));
        switch (eState)
        {
        case STATE_SCANNING: DrawScanIndicator(pCtx); break;
        case STATE_FROZEN:   DrawFrozenBadge(pCtx);   break;
        default:             break;
        }
    }

    // ---- 복합 상태는 Mutex가 필요한 경우 ----
    // 여러 필드가 원자적으로 일관성을 가져야 할 때:
    // ScCommon::ScopedLock lock(m_StateMutex);
    // m_nFrameIndex = nNewIndex;   // 이 두 필드가
    // m_pCurrentFrame = pNewFrame; // 항상 같이 바뀌어야 함

private:
    enum AcqState { STATE_IDLE = 0, STATE_SCANNING, STATE_FROZEN };

    std::atomic<bool> m_bSweActive;  // 취득↔렌더 단방향 플래그
    std::atomic<int>  m_eAcqState;   // 상태 enum (int로 저장)
};
```

## Key Points

- `std::atomic<bool>` / `std::atomic<int>`은 MSVC C++11에서 락-프리가 보장된다(x86/x64 기준)
- `store(..., memory_order_release)` + `load(..., memory_order_acquire)`: 쓰기 쪽 변경이 읽기 쪽에 보이는 순서를 보장한다
- 여러 관련 필드를 한 번에 바꿔야 할 때는 atomic이 아닌 Mutex(ScCommon::ScopedLock)를 써야 한다
- atomic 사용 범위: POD 단일 값(bool, int, 포인터) 상태 플래그에 한정한다
- 우선순위 역전은 디버거로 잡기 어렵고 24/7 장비에서 간헐적 프레임 드롭으로 나타난다
