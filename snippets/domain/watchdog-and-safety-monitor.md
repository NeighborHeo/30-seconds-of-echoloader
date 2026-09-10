---
title: "소프트웨어 워치독과 의료기기 안전 모니터"
category: domain
tags: [watchdog, safety, timer, heartbeat, fail-safe, sccommon]
difficulty: intermediate
---

렌더링 루프 동결을 감지해 하드 크래시 대신 제어된 종료를 수행하는 소프트웨어 워치독 패턴.

## Why

실시간 초음파 스캐닝 중 렌더링 스레드가 교착(deadlock)이나 무한 루프에 빠지면 두 가지 결과 중 하나다:
1. 화면이 멈추지만 소프트웨어는 여전히 실행 중 — 임상의가 정적 이미지를 실시간으로 착각
2. OS가 강제 종료(BSOD/segfault) — 어떤 정리 코드도 실행 안 됨, 로그 유실

어느 쪽이든 환자 안전 위협이다.
소프트웨어 워치독은 "하트비트" 부재를 감지해 제어된 종료(graceful shutdown)와 경고 표시를 수행한다.

## Pattern

```cpp
// RenderWatchdog.h — 렌더 루프 감시 워치독
#pragma once
#include "ScCommon/Timer.h"
#include "ScCommon/Mutex.h"
#include "ScCommon/ScopedLock.h"
#include <cstdint>

class RenderWatchdog {
public:
    static const int WATCHDOG_TIMEOUT_MS = 3000;  // 3초 무응답 = 동결 판정

    explicit RenderWatchdog(int nTimeoutMs = WATCHDOG_TIMEOUT_MS)
        : m_nTimeoutMs(nTimeoutMs)
        , m_bArmed(false)
    {}

    // 렌더 스레드에서 매 프레임 호출 — "나 살아있어"
    void Heartbeat()
    {
        ScCommon::ScopedLock lock(m_mutex);
        m_lastHeartbeatTick = ScCommon::Timer::GetTickMs();
    }

    // 감시 스레드에서 주기적으로 호출
    bool IsRenderFrozen() const
    {
        ScCommon::ScopedLock lock(m_mutex);
        if (!m_bArmed) return false;
        uint64_t nNow = ScCommon::Timer::GetTickMs();
        return (nNow - m_lastHeartbeatTick) > static_cast<uint64_t>(m_nTimeoutMs);
    }

    void Arm()
    {
        ScCommon::ScopedLock lock(m_mutex);
        m_lastHeartbeatTick = ScCommon::Timer::GetTickMs();
        m_bArmed = true;
    }

    void Disarm() {
        ScCommon::ScopedLock lock(m_mutex);
        m_bArmed = false;
    }

private:
    mutable ScCommon::Mutex m_mutex;
    uint64_t                m_lastHeartbeatTick = 0;
    int                     m_nTimeoutMs;
    bool                    m_bArmed;
};
```

```cpp
// 렌더 루프 — 매 프레임 하트비트
void GcViewer::RenderLoop()
{
    m_watchdog.Arm();
    while (m_bRunning) {
        RenderFrame();
        m_watchdog.Heartbeat();  // 정상 동작 신호
    }
    m_watchdog.Disarm();
}

// 감시 스레드 — 별도 스레드에서 폴링
void SafetyMonitor::MonitorLoop()
{
    while (m_bRunning) {
        ScCommon::Timer::SleepMs(500);  // 500 ms 간격 확인

        if (m_renderWatchdog.IsRenderFrozen()) {
            // fail-safe: 하드 크래시 대신 제어된 종료
            GcLog::Critical("RenderWatchdog: render loop frozen — initiating safe shutdown");
            ShowUserWarning("영상이 멈췄습니다. 장비를 재시작하십시오.");
            InitiateGracefulShutdown();
            // ponytail: 단순 플래그+로그 — 자동 복구는 재스캔 부작용 위험이 있어 수동 재시작 선택
            break;
        }
    }
}
```

## Key Points

- **하드 크래시 vs 제어된 종료**: 하드 크래시는 로그 버퍼가 플러시되지 않아 원인 추적 불가. 제어된 종료는 로그를 디스크에 쓰고, 하드웨어(트랜스듀서 전원 등)를 안전 상태로 설정한 뒤 종료
- **fail-safe 원칙**: 실패 시 더 안전한 상태로 전환. 렌더 동결 → 영상 중단 + 경고 표시가 "영상이 멈춘 줄 모르고 진단"보다 낫다
- 워치독 타임아웃은 최대 허용 프레임 지연보다 훨씬 크게 설정 (예: 30 fps → 33 ms/frame, 타임아웃 3000 ms = 90 프레임)
- 자동 복구(스레드 재시작) 시도는 위험: 하드웨어 상태와 SW 상태가 불일치할 수 있어 오진단보다 위험할 수 있음 — 수동 재시작이 Class C 의료기기에서 더 안전
- IEC 62304 § 7.1 (소프트웨어 유지보수 계획)과 IEC 60601-1 § 14 (프로그래밍 가능 전자 의료 시스템) 요구사항에서 소프트웨어 워치독 구현을 안전 수단으로 인정
- `ScCommon::Timer::GetTickMs()` 는 단조 증가(monotonic) 타이머여야 함 — 시스템 시간 변경에 영향받는 `GetSystemTime()` 기반이면 워치독이 오작동
