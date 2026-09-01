---
title: "ScCommon Thread.h 래퍼 패턴"
category: cpp-patterns
tags: [threading, ScCommon, COM, STA, lock]
difficulty: advanced
---

`<ScCommon/Thread.h>` 래퍼와 `std::thread`를 병행 사용하되, COM STA 스레딩 모델과 스레드 친화도 제약을 명시적 주석으로 표시한다.

## Why

gipc-app COM 컴포넌트는 STA(Single-Threaded Apartment) 모델을 요구한다. UI 스레드에서만 호출 가능한 GE 내부 API가 존재하며, 신규 병렬화 작업에서 이를 모르면 재현하기 어려운 스레드-안전 버그가 발생한다. `ScCommon::Thread`는 GE 플랫폼 레이어의 래퍼로, `std::thread`와 동시에 사용된다.

## Pattern

```cpp
#include <ScCommon/Thread.h>  // GE 플랫폼 스레드 래퍼
#include <thread>              // std::thread — 둘 다 사용

// COM STA 컴포넌트: 아파트 스레딩 명시
// _ATL_APARTMENT_THREADED — 반드시 메인 UI 스레드에서 생성/소멸
class CMyComComponent : public CComObjectRootEx<CComSingleThreadModel> {
    // ...
};

// 스레드 친화도 주석 — 신규 코드에서 의무
// GEOComm::InitFromMainUIThread: UI 스레드에서만 호출 가능
void Initialize() {
    // Thread local storage for internal state — internal use only
    // ponytail: 스레드 안전 어노테이션(THREAD_SAFE) 없음, 명시적 주석으로 대체
    GEOComm::InitFromMainUIThread();  // Must be called from main UI thread
}

// 뮤텍스 패턴: std::lock_guard (짧은 임계구역)
class AcquisitionAssistant {
public:
    void UpdateState(AcqAssistState newState) {
        std::lock_guard<std::mutex> lock(m_mutex);
        m_state = newState;
    }

    AcqAssistState GetState() const {
        std::lock_guard<std::mutex> lock(m_mutex);
        return m_state;
    }

private:
    mutable std::mutex m_mutex;
    AcqAssistState m_state;
};

// 조건부 대기: std::unique_lock (wait/notify 필요 시)
class WorkQueue {
public:
    void WaitForWork() {
        std::unique_lock<std::mutex> lock(m_mutex);
        m_cv.wait(lock, [this] { return !m_queue.empty(); });
    }

private:
    std::mutex m_mutex;
    std::condition_variable m_cv;
    std::queue<WorkItem> m_queue;
};

// ScCommon::Thread 사용 패턴 (GE 플랫폼 래퍼)
ScCommon::Thread m_workerThread;
// std::thread와 인터페이스 유사하나 GE 플랫폼 초기화 처리 포함
```

## Key Points

- COM STA 컴포넌트는 생성·소멸·메서드 호출을 모두 동일 스레드에서 수행
- `GEOComm::InitFromMainUIThread` 같은 UI 스레드 전용 API는 호출 지점에 `// Must be called from main UI thread` 주석 필수
- `std::lock_guard`: 짧은 임계구역, RAII 자동 해제. `std::unique_lock`: `wait`/`try_lock`/수동 해제가 필요할 때
- `THREAD_SAFE` 어노테이션 없음 — 신규 병렬화 코드에는 스레드 안전성을 명시적 주석으로 문서화
- Thread local storage는 `// Thread local storage for ... internal use only` 주석으로 표시
- `ScCommon::Thread`와 `std::thread` 혼용 시 생명주기 관리 주의 — 소멸 전 `join()` 보장
