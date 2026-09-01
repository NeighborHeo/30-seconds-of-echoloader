---
title: "ScCommon 라이브러리 지도"
category: architecture
tags: [ScCommon, library, utilities, threading, logging, infrastructure]
difficulty: intermediate
---

`<ScCommon/...>` 는 gipc-app 전체가 공유하는 GE 공통 인프라 라이브러리다. 주요 헤더 10개를 알면 코드베이스 탐색 속도가 두 배 빨라진다.

## 핵심 헤더 맵

```
<ScCommon/>
│
├── Thread.h          ← 스레드 풀 래퍼, 작업 큐
│   ScCommon::Thread::Run(lambda)
│   ScCommon::Thread::RunPeriodic(lambda, intervalMs)
│
├── LogShim.h         ← ILogSimple 레거시 래퍼 (신규 코드에서 직접 사용 금지)
│   ScLog("MODULE", ScLog::Error, "msg")
│
├── AuditLog.h        ← 감사 로그 (FDA/규제)
│   AuditLogUserEvent(AUDIT_LOG, "MODULE", "format", ...)
│   AuditLogUserEvent(SECURITY_ALERT, "MODULE", "format", ...)
│
├── Mutex.h / Lock.h  ← std::mutex 래퍼 (GE 명명 컨벤션 적용)
│   ScCommon::Mutex m_mutex;
│   ScCommon::ScopedLock lock(m_mutex);  // std::lock_guard 상당
│
├── Timer.h           ← 고해상도 타이머, 스로틀링
│   ScCommon::Timer t; t.Start(); t.ElapsedMs();
│
├── String.h          ← 문자열 유틸 (UTF-8 ↔ UTF-16, 분리, 트림)
│   ScCommon::String::ToWide(str)
│   ScCommon::String::Split(str, ',')
│
├── Path.h            ← 파일 경로 유틸 (C++17 std::filesystem 미도입 대체)
│   ScCommon::Path::Combine(base, relative)
│   ScCommon::Path::Exists(path)
│
├── Registry.h        ← Windows 레지스트리 접근
│   ScCommon::Registry::Read(key, valueName, defaultVal)
│
├── Version.h         ← 버전 정보 접근
│   ScCommon::Version::GetBuildString()
│
└── Platform.h        ← OS/플랫폼 추상화 (WINVER 등)
    ScCommon::Platform::IsDebugBuild()
```

## 사용 예시

```cpp
#include <ScCommon/Thread.h>
#include <ScCommon/Timer.h>
#include <ScCommon/Mutex.h>

class FrameProcessor
{
public:
    void StartAsync()
    {
        // 스레드 풀에 작업 투입
        ScCommon::Thread::Run([this]() { ProcessLoop(); });
    }

    void AddFrame(const Frame& frame)
    {
        ScCommon::ScopedLock lock(m_mutex);  // lock_guard 상당
        m_queue.push(frame);
    }

private:
    void ProcessLoop()
    {
        ScCommon::Timer timer;
        timer.Start();

        while (m_bRunning)
        {
            ProcessNextFrame();

            // 16ms 이상 걸리면 경고
            if (timer.ElapsedMs() > 16.0)
                ScLogWarn("FRAMEPROC", "Frame processing overrun: %.1f ms",
                          timer.ElapsedMs());
            timer.Reset();
        }
    }

    ScCommon::Mutex   m_mutex;
    std::queue<Frame> m_queue;
    bool              m_bRunning = true;
};
```

## Key Points

- `<ScCommon/Thread.h>` 는 `std::thread` 직접 사용보다 선호 — 스레드 풀 관리, 예외 처리 포함
- `ScCommon::ScopedLock` 은 `std::lock_guard` 와 동일 의미지만 GE 명명 규칙 적용
- `<ScCommon/Path.h>` 는 C++17 `std::filesystem` 미도입 코드베이스의 대체 — 신규 코드에서도 이것을 사용 (C++17 도입 금지)
- `<ScCommon/LogShim.h>` 는 레거시 전용 — **신규 코드에서 직접 포함 금지**
- `<ScCommon/AuditLog.h>` 는 보안/규제 이벤트 전용 — 일반 진단 로그에 오용 금지
