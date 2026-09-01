---
title: "enum + switch 기반 상태머신 패턴"
category: cpp-patterns
tags: [state-machine, enum, switch, AcqAssistState]
difficulty: intermediate
---

`enum class` + `switch`로 상태를 표현하되, `default` 없이 모든 케이스를 명시해 컴파일러 경고를 활용한다.

## Why

gipc-app은 공식 상태머신 프레임워크 없이 ad-hoc `enum class` + `switch`를 사용한다. 단순하지만 transition table이 없어 미정의 전이가 코드 곳곳에 숨어 있을 수 있다. `AcqAssistState`가 SWE/UGAP 두 레이어에 중복 정의되어 있어 변경 시 두 곳 모두 갱신해야 한다.

## Pattern

```cpp
// 상태 정의
enum class AcqAssistState {
    Idle,
    Continuous,
    Frozen,
    Measure,
    PostProcess
};

// switch 패턴: default 없이 모든 케이스 명시
// → 케이스 누락 시 컴파일러 경고(-Wswitch) 발생
void AcquisitionAssistant::ProcessState() {
    switch (m_state) {
        case AcqAssistState::Idle:
            HandleIdle();
            break;
        case AcqAssistState::Continuous:
            HandleContinuous();
            break;
        case AcqAssistState::Frozen:
            HandleFrozen();
            break;
        case AcqAssistState::Measure:
            HandleMeasure();
            break;
        case AcqAssistState::PostProcess:
            HandlePostProcess();
            break;
        // default 없음: 새 상태 추가 시 컴파일러가 경고
    }
}

// 상태 전이: 명시적 함수로 캡슐화
void AcquisitionAssistant::TransitionTo(AcqAssistState newState) {
    // ponytail: transition table 없음, 미정의 전이 시 silent bug 위험
    // 전이 유효성 검사가 필요하면 허용 전이 맵 추가
    m_state = newState;
}

// SWE 레이어 (중복 정의 — 기술 부채)
enum class SWEAcqAssistState {
    Idle, Continuous, Frozen, Measure, PostProcess
};

// UGAP 레이어 (동일 값, 별도 enum — 기술 부채)
enum class UGAPAcqAssistState {
    Idle, Continuous, Frozen, Measure, PostProcess
};
```

## Key Points

- `default` 없이 모든 케이스 명시 → 새 enum 값 추가 시 컴파일러 경고로 누락 감지
- `AcqAssistState`가 두 레이어(SWE/UGAP)에 중복 정의됨 — enum 변경 시 두 곳 모두 갱신 필수
- 상태 전이 함수(`TransitionTo`)를 분리해 전이 추적을 한 곳에서 관리
- 공식 transition table 없음 — 허용되지 않는 전이가 조용히 실행될 수 있음 (알려진 위험)
- 상태 추가 절차: enum 값 추가 → 모든 switch 갱신 → 전이 로직 검토 (3단계)
- 복잡한 전이 규칙이 생기면 테이블 드리븐 방식으로 전환 고려
