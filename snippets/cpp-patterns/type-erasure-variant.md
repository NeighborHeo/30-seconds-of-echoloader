---
title: "타입 소거(Type Erasure)와 Variant 파라미터 버스"
category: cpp-patterns
tags: [type-erasure, variant, parameter-bus, gc-parameter, runtime-polymorphism]
difficulty: advanced
---

GC 파라미터 버스는 `Variant`로 int/float/string/bool을 컴파일 타임 타입 정보 없이 전달한다.

## Why

gipc-app의 파라미터 시스템은 스캔 모드, 게인, 주파수, 레이블 등 이질적인 값을
단일 채널(GC parameter bus)로 전송한다. 수신자마다 타입이 다르므로 `void*`(unsafe)나
템플릿 파라미터화(컴파일 타임 고정)는 불가능하다. `std::any`는 C++17이라 미사용.
→ tagged union 스타일의 경량 `Variant`로 해결.

## Pattern

```cpp
// GcVariant.h — 경량 타입 소거 컨테이너
#pragma once
#include <string>
#include <cassert>

class GcVariant {
public:
    enum class Type { Empty, Int, Float, Bool, String };

    GcVariant() : m_type(Type::Empty) {}
    explicit GcVariant(int    v) : m_type(Type::Int),    m_iVal(v) {}
    explicit GcVariant(float  v) : m_type(Type::Float),  m_fVal(v) {}
    explicit GcVariant(bool   v) : m_type(Type::Bool),   m_bVal(v) {}
    explicit GcVariant(const std::string& v) : m_type(Type::String), m_sVal(v) {}

    Type GetType() const { return m_type; }

    int         AsInt()    const { assert(m_type == Type::Int);    return m_iVal; }
    float       AsFloat()  const { assert(m_type == Type::Float);  return m_fVal; }
    bool        AsBool()   const { assert(m_type == Type::Bool);   return m_bVal; }
    const std::string& AsString() const { assert(m_type == Type::String); return m_sVal; }

private:
    Type        m_type;
    // ponytail: 단순 union 대신 멤버 구조 — std::string은 union에 넣을 수 없음(C++11)
    int         m_iVal  = 0;
    float       m_fVal  = 0.f;
    bool        m_bVal  = false;
    std::string m_sVal;
};
```

```cpp
// 파라미터 버스 송신 — 타입을 몰라도 전달 가능
void GcParameterBus::Publish(const std::string& sKey, const GcVariant& value)
{
    m_params[sKey] = value;
    NotifySubscribers(sKey);
}

// 파라미터 버스 수신 — dispatch 패턴
void GainController::OnParameterChanged(const std::string& sKey, const GcVariant& value)
{
    if (sKey == "Gain") {
        switch (value.GetType()) {
        case GcVariant::Type::Int:
            SetGain(static_cast<float>(value.AsInt()));
            break;
        case GcVariant::Type::Float:
            SetGain(value.AsFloat());
            break;
        default:
            // 타입 불일치 — 로그 후 무시 (의료기기: 무음 실패 금지)
            GcLog::Error("GainController: unexpected Variant type for key=%s", sKey.c_str());
            break;
        }
    }
}
```

## Key Points

- **`void*` 대비**: 타입 태그가 있어 잘못된 캐스트를 `assert`로 조기 발견
- **`std::any` 대비**: C++17 불필요. RTTI 의존 없음. MSVC `/GR-` 설정에서도 동작
- **`std::variant` 대비**: C++17 불필요. 방문자(visitor) 패턴 없이 switch dispatch로 단순 유지
- `std::string` 멤버 때문에 naked union 사용 불가(C++11) → 멤버 필드 방식이 현실적 선택
- 수신 측 default 케이스에서 반드시 로그 남길 것 — 의료기기에서 무음 실패(silent failure)는 IEC 62304 위반으로 이어질 수 있음
- 성능이 중요한 경루프에서는 string key 대신 enum key로 교체 고려 (`ponytail: string 비교 O(n), enum 비교 O(1)`)
