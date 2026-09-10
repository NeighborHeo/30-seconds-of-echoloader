---
title: "SWE + UGAP 공통 기능의 이중 게이팅 패턴"
category: workflow
tags: [swe, ugap, gating, esmain, acquisition-assistant, elasto]
difficulty: intermediate
---

ESMain에서 SWE와 UGAP 두 모드 모두에서 실행되어야 하는 기능을 추가할 때 꺼내 본다.

## 언제 이 아티클을 보나

- ESMain.cpp에 새 AA(Acquisition Assistant) 호출 사이트를 추가하는 PR을 작성할 때
- 코드 리뷰 중 `Manager::Instance()` 호출에 게이팅이 있는지 확인할 때
- 기존 SWE 전용 기능을 UGAP에도 적용하라는 티켓을 받았을 때

## 잘못된 패턴

```cpp
// ❌ 게이팅 없음 — 모든 모드에서 실행됨
// B모드, M모드, CFM 등 무관하게 AA 핸들러가 호출되어 임상 데이터 오염
Manager::Instance(HandlerType::SWE)->UpdateCommonState(data);

// ❌ SWE 게이트만 — UGAP 모드에서 누락
// UGAP 모드로 전환 후 UpdateCommonState가 호출되지 않아 상태 불일치
if (InElasto())
    Manager::Instance(HandlerType::SWE)->UpdateCommonState(data);
```

## 패턴 1: 두 개의 독립적 게이팅 사이트 (권장)

핸들러별로 동작이 조금이라도 다를 때, 또는 명시성을 우선할 때 사용한다.

```cpp
// ✅ 두 개의 독립적 게이팅 사이트 (권장)
if (InElasto())
    Manager::Instance(HandlerType::SWE)->UpdateCommonState(data);

if (IsInUGAPMode())
    Manager::Instance(HandlerType::UGAP)->UpdateCommonState(data);
```

- `InElasto()`와 `IsInUGAPMode()`는 배타적이므로 동시에 두 블록이 실행되지 않는다.
- 각 사이트가 독립적이므로 나중에 핸들러별로 다른 인자를 넘기기 쉽다.
- git blame 시 "이 사이트는 UGAP용"이라는 의도가 명확히 드러난다.

## 패턴 2: 공통 로직을 인터페이스로 추출

두 핸들러에서 완전히 동일한 코드가 복사·붙여넣기되는 상황일 때만 사용한다.

```cpp
// ✅ 공통 로직은 인터페이스 기본 구현으로
// IAcqAssistHandler에 UpdateCommonState() 순수 가상 함수 추가
// AcquisitionAssistantBase에서 공통 구현 제공
// SweHandler / UgapHandler가 각자 오버라이드 (필요 시)

// ESMain 호출부는 여전히 두 사이트 유지:
if (InElasto())
    Manager::Instance(HandlerType::SWE)->UpdateCommonState(data);

if (IsInUGAPMode())
    Manager::Instance(HandlerType::UGAP)->UpdateCommonState(data);
```

핸들러 내부 구현이 공유될 뿐, ESMain의 두 게이팅 사이트 구조는 유지된다.

## 결정 규칙

```
두 핸들러에서 동일한 코드가 복사되는가?
  YES → IAcqAssistHandler 공통 구현으로 이동 (구현 공유)
  NO  → 두 개의 별도 사이트, 각각 독립 구현 유지

동작이 약간 다른가?
  YES → 두 개의 별도 사이트, 인자·로직 분기 각자
  NO  → 공통 구현 추출 검토
```

## OR 게이트를 절대 쓰면 안 되는 이유

```cpp
// ❌ OR 게이팅 — 향후 모드 추가 시 반드시 문제 발생
if (InElasto() || IsInUGAPMode())
    Manager::Instance(HandlerType::SWE)->UpdateCommonState(data);
```

- 조건 자체는 현재 시점에 맞지만, ESMain에 11개 사이트가 있을 때 새 모드(예: `IsInNewMode()`)를 추가하면 일부 사이트만 업데이트하고 나머지를 놓치는 인간 실수가 발생한다.
- "모든 사이트가 동일한 패턴을 따른다"는 불변식이 깨진다.
- SWE 핸들러가 UGAP 데이터를 처리하는 타입 오염이 발생한다.

## Key Points

- ESMain의 모든 AA 호출 사이트는 `게이트 → Instance → 위임` 세 줄 구조를 따른다.
- `InElasto()`와 `IsInUGAPMode()`는 배타적 조건이므로 동시에 양쪽 블록이 실행되지 않는다.
- 공통 기능은 핸들러 내부 구현을 공유하되 ESMain 게이팅 사이트는 분리 유지한다.
- OR 게이팅(`||`)은 새 모드 추가 시 누락 사이트를 만드는 구조적 함정이다.
- 새 사이트 추가 후 `grep -n "Manager::Instance" ESMain.cpp | wc -l`로 사이트 수 증가 확인한다.
