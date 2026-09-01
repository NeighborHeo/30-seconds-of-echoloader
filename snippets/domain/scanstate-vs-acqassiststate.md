---
title: "ScanState vs AcqAssistState: 두 상태 머신의 계층 관계"
category: domain
tags: [state-machine, ScanState, AcqAssistState, ESMain, AcquisitionAssistant]
difficulty: intermediate
---

ScanState는 장비 전체의 전역 상태, AcqAssistState는 AA 모드 전용 로컬 상태다 — 단방향 의존.

## Why

gipc-app에는 스캔 상태를 표현하는 enum이 두 계층에 존재한다. 혼동하면 레이어 위반 버그가 생긴다: ESMain 바깥에서 ScanState를 읽어 AA 동작을 결정하는 코드가 그 예다.

## Pattern

```cpp
// ESMain.h — 장비 전체 전역 상태
enum class ScanState
{
    Idle,
    Scanning,
    Freeze,
    Live,
    // ...
};

// AcquisitionAssistant.h (Layer A) — AA 전용 세부 상태
// AcquisitionAssistantBase.h (Layer B) — 동일 enum 중복 정의 (D4 작업으로 통합 예정)
enum class AcqAssistState
{
    Idle,
    Running,
    Continuous,
    Freeze,
    Measure,
};

// ESMain이 ScanState 변화를 감지하고 AA Manager에 알림
// AA는 자신의 AcqAssistState를 업데이트
void ESMain::OnScanStateChanged(ScanState newState)
{
    if (newState == ScanState::Freeze)
    {
        m_pAcqAssistManager->NotifyFreeze();
        // AA 내부에서: m_acqAssistState = AcqAssistState::Freeze;
    }
}

// ❌ 레이어 위반 — ESMain 외부에서 ScanState로 AA 동작 결정
void SomeOtherClass::DoSomething(ScanState scanState)
{
    if (scanState == ScanState::Freeze)          // 위반: AA 로직이 외부로 누출
    {
        m_pAcqAssist->StopProcessing();
    }
}

// ✅ 올바른 패턴 — AA 자신의 상태로 판단
void AcquisitionAssistant::ProcessFrame(const Frame& frame)
{
    if (m_acqAssistState == AcqAssistState::Freeze)
    {
        return;  // AA 로컬 상태로만 판단
    }
    // ...
}
```

## Key Points

- ScanState는 ESMain 레벨의 전역 상태로 장비 전체(Idle/Scanning/Freeze/Live)를 표현한다.
- AcqAssistState는 AA 내부 전용 세부 상태(Idle/Running/Continuous/Freeze/Measure)다.
- 흐름은 단방향: ScanState 변화 → ESMain이 Manager에 알림 → AcqAssistState 업데이트.
- 역방향(AcqAssistState → ScanState)은 없다 — AA 상태 변화가 장비 전체 상태를 바꾸지 않는다.
- 현재 AcqAssistState enum이 AcquisitionAssistant.h와 AcquisitionAssistantBase.h 두 곳에 중복 정의되어 있으며, D4 작업에서 AcqAssistTypes.h 단일 파일로 통합 예정이다.
