---
title: "Manager::Instance() 싱글톤 패턴"
category: cpp-patterns
tags: [singleton, static, lifetime, AcquisitionAssistant]
difficulty: intermediate
---

`HandlerType` 파라미터로 SWE/UGAP 인스턴스를 분기하는 타입-키드 싱글톤.

## Why

gipc-app의 `AcquisitionAssistant`는 SWE/UGAP 두 모드가 존재한다. 하나의 `Instance()` 호출 지점으로 통일하되, 내부 구현은 분리한다. `ESMain.cpp`에 11개 호출 사이트가 있으며, 타입 파라미터 없이 단순화하면 두 모드 간 상태 오염이 발생한다.

## Pattern

```cpp
// AcquisitionAssistant.h
enum class HandlerType {
    SWE,
    UGAP
};

class AcquisitionAssistant {
public:
    // 타입-키드 싱글톤: HandlerType으로 인스턴스 분기
    static AcquisitionAssistant& Instance(HandlerType type);

    void DoSomething();
    void DoSomethingElse();

    // 소멸자는 public — unique_ptr 소멸에 필요
    ~AcquisitionAssistant() = default;

private:
    AcquisitionAssistant() = default;

    // s_ 접두사: 정적 멤버 명명 규칙
    static std::unique_ptr<AcquisitionAssistant> s_sweInstance;
    static std::unique_ptr<AcquisitionAssistant> s_ugapInstance;
};

// AcquisitionAssistant.cpp
std::unique_ptr<AcquisitionAssistant> AcquisitionAssistant::s_sweInstance;
std::unique_ptr<AcquisitionAssistant> AcquisitionAssistant::s_ugapInstance;

AcquisitionAssistant& AcquisitionAssistant::Instance(HandlerType type) {
    if (type == HandlerType::SWE) {
        if (!s_sweInstance) {
            s_sweInstance = std::unique_ptr<AcquisitionAssistant>(
                new AcquisitionAssistant());
        }
        return *s_sweInstance;
    } else {
        if (!s_ugapInstance) {
            s_ugapInstance = std::unique_ptr<AcquisitionAssistant>(
                new AcquisitionAssistant());
        }
        return *s_ugapInstance;
    }
}

// ESMain.cpp 호출 패턴 (11개 사이트)
AcquisitionAssistant::Instance(HandlerType::SWE).DoSomething();
AcquisitionAssistant::Instance(HandlerType::UGAP).DoSomethingElse();
```

## Key Points

- 정적 멤버는 `s_` 접두사: `s_sweInstance`, `s_ugapInstance`
- `make_unique` 대신 `unique_ptr(new ...)` — C++11 호환, `make_unique`는 C++14
- SWE/UGAP를 하나의 인스턴스로 합치지 않는다 — 두 모드는 독립적인 상태를 가짐
- `Instance()` 반환은 레퍼런스. nullptr 반환 없음 — 초기화 실패는 예외 또는 assert
- 멀티스레드 환경에서 최초 초기화는 보호되지 않음 — `ESMain.cpp`의 단일 초기화 지점에서만 호출 보장
- 소멸은 프로세스 종료 시 자동. `s_instance` reset이 필요하면 명시적 `Shutdown()` 추가
