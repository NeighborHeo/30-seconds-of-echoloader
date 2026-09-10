---
title: "GC 파라미터 버스 동시 발행 시 순서 보증"
category: architecture
tags: [gc-params, threading, ordering, race-condition, SetParameterValue, ESMain]
difficulty: advanced
---

UI 스레드와 Acquisition 스레드가 같은 키에 동시에 `SetParameterValue()`를 호출할 때 어떤 값이 OVObject에 도달하는가.

## 언제 이 아티클을 보나

`SWEAcqAssist.State` 같은 파라미터가 16ms 프레임 안에서 두 스레드로부터 발행될 때
렌더 결과가 간헐적으로 예상과 다를 경우. 버스 자체의 스레드 안전성과 순서 보증을 혼동하지 않기 위해.

## 시나리오

```cpp
// UI 스레드: 사용자가 모드 전환 버튼을 누름
void ESMain::OnModeButton()
{
    SetParameterValue("SWEAcqAssist.State",
                      static_cast<int>(AcqAssistState::Idle));  // ← 발행 1
}

// Acquisition 스레드: 동시에 새 프레임 도착
void ESMain::OnNewFrame(const Frame& frame)
{
    if (InElasto())
        SetParameterValue("SWEAcqAssist.State",
                          static_cast<int>(AcqAssistState::Acquiring));  // ← 발행 2
}
```

두 호출이 같은 16ms 프레임 안에서 실행될 수 있다.

## GC 파라미터 버스의 실제 동작

버스 내부에 잠금이 있으므로 **데이터 레이스(data race)는 없다**.
하지만 **발행 순서는 보증되지 않는다** — 두 스레드 중 잠금을 마지막으로 획득한 스레드의 값이 "최종 값"이 된다.

`OVObject::SetParameter()`는 `SetParameterValue()` 호출마다 한 번씩 실행된다 (배칭 없음).
따라서 Render() 틱 이전에 두 호출이 모두 완료되면 OVObject는 두 번 `SetParameter()`를 받는다.

## 두 가지 가능한 결과

```
시나리오 A — 발행 1 → 발행 2 순서로 잠금 획득:
  OVObject::SetParameter("SWEAcqAssist.State", Idle)      → m_state = Idle
  OVObject::SetParameter("SWEAcqAssist.State", Acquiring) → m_state = Acquiring
  Render(): Active 표시

시나리오 B — 발행 2 → 발행 1 순서로 잠금 획득:
  OVObject::SetParameter("SWEAcqAssist.State", Acquiring) → m_state = Acquiring
  OVObject::SetParameter("SWEAcqAssist.State", Idle)      → m_state = Idle
  Render(): 비활성 표시

런타임에 어느 시나리오가 실행될지 비결정적
```

## 올바른 설계 규칙

동시 발행 자체가 발생하지 않도록 **게이팅**으로 설계한다.

```cpp
// ✅ 올바른 설계: InElasto() 게이트가 겹침 창을 없앤다
// 모드 전환 중에는 InElasto() == false
// → OnNewFrame()에서 Acquiring 발행 안 됨
// → OnModeButton()의 Idle 발행만 존재 → 결과 결정적

void ESMain::OnNewFrame(const Frame& frame)
{
    if (InElasto())  // ← 이 조건이 전환 중엔 false를 보장해야 함
        SetParameterValue("SWEAcqAssist.State",
                          static_cast<int>(AcqAssistState::Acquiring));
}

// ❌ 게이팅 없이 두 스레드가 동시에 발행하는 설계
// → SetParameterValue 앞에 명시적 직렬화(뮤텍스)가 필요
// → 그게 없으면 간헐적 표시 불일치 버그 발생
```

`InElasto()` 게이트가 전환 완료 전에 `true`를 반환하는 상태 머신 버그가 있다면,
이 겹침 창이 열리고 시나리오 A/B 비결정성이 재현된다.

## 게이팅이 보장되지 않을 때의 대안

```cpp
// 직렬화가 필요한 경우: 발행 측에 단일 소유권 부여
// UI 스레드만 State를 발행할 권한을 가짐
// Acquisition 스레드는 별도 키("SWEAcqAssist.FrameReady")를 발행

// UI 스레드:
SetParameterValue("SWEAcqAssist.State", static_cast<int>(AcqAssistState::Idle));

// Acquisition 스레드: State는 건드리지 않음
SetParameterValue("SWEAcqAssist.FrameReady", 1);

// OVObject에서 두 정보를 조합해 실제 상태 결정
// → 같은 키에 두 스레드가 쓰는 구조 자체를 제거
```

## Key Points

- GC 파라미터 버스는 스레드 안전(내부 잠금)이지만 **발행 순서는 보증하지 않는다** — 두 가지를 혼동하지 않는다
- `SetParameterValue()` 호출은 배칭되지 않는다 — 같은 키에 두 번 발행하면 `OVObject::SetParameter()`도 두 번 호출된다
- 16ms 프레임 안에서 같은 키에 두 스레드가 발행하면 어느 값이 최종값이 될지 런타임 스케줄러가 결정한다
- 올바른 설계는 겹침 창 자체를 없애는 것이다 — 게이팅 조건(`InElasto()` 등)이 그 역할을 한다
- 게이팅으로 겹침을 막을 수 없는 설계라면, 같은 키의 소유권을 단일 스레드에 부여하거나 발행 측에 명시적 잠금을 추가한다
