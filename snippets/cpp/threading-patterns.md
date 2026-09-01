---
title: "Threading Patterns"
category: cpp
tags: [threading, mutex, lock-guard, safety, concurrency]
difficulty: intermediate
---

`std::thread` + `std::mutex`를 직접 쓰거나 `<ScCommon/Thread.h>` 래퍼를 사용한다. 스레드 안전 어노테이션은 없으므로 친화도 주석이 유일한 문서다.

## Pattern

```cpp
#include <mutex>
#include <thread>
#include <ScCommon/Thread.h>   // GE 래퍼 (내부 풀 관리)

class ImageProcessor
{
public:
    // ScCommon 래퍼 — 스레드 풀 사용
    void StartProcessing()
    {
        ScCommon::Thread::Run([this]() { ProcessLoop(); });
    }

    // 직접 std::thread
    void StartMonitor()
    {
        m_monitorThread = std::thread([this]() { MonitorLoop(); });
    }

    // lock_guard: 짧은 임계구역
    void UpdateBuffer(const Frame& frame)
    {
        std::lock_guard<std::mutex> lock(m_bufferMutex);
        m_buffer = frame;
    }

    // unique_lock: 조건 변수와 함께
    void WaitForFrame()
    {
        std::unique_lock<std::mutex> lock(m_bufferMutex);
        m_cv.wait(lock, [this]{ return m_bFrameReady; });
    }

private:
    std::mutex              m_bufferMutex;
    std::condition_variable m_cv;
    std::thread             m_monitorThread;
    bool                    m_bFrameReady = false;
    Frame                   m_buffer;
    // GEOComm::InitFromMainUIThread — UI 친화도 주석 예시
};
```

## COM 스레딩 모델

```cpp
// ATL COM 컴포넌트 — Apartment 모델 명시 (필수)
[
    coclass,
    threading("apartment"),   // _ATL_APARTMENT_THREADED
    ...
]
class ATL_NO_VTABLE CAcqAssistServer : public IAcqAssistServer { ... };
```

## Key Points

- `std::lock_guard` 는 예외 안전 + RAII → 짧은 임계구역에 항상 선택
- `std::unique_lock` 은 `wait()` / `try_lock()` 이 필요할 때만
- **스레드 안전 어노테이션(`THREAD_SAFE`/`NOT_THREAD_SAFE`) 없음** — 친화도 주석으로 보완
  ```cpp
  // Thread local storage for internal use only — do not call from UI thread
  // GEOComm::InitFromMainUIThread — must be called on main UI thread
  ```
- COM 컴포넌트는 `_ATL_APARTMENT_THREADED` 명시 의무
- 새 스레드 도입 시 호출 친화도(어느 스레드에서 호출 가능/불가) 주석 필수
