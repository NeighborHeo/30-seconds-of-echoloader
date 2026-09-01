---
title: "Variant — GC 파라미터 버스의 타입-안전 값 컨테이너"
category: cpp-patterns
tags: [variant, parameter-bus, type-safety, gc-framework]
difficulty: intermediate
---

GC 파라미터 버스는 `Variant`를 통해 이종(heterogeneous) 값을 단일 인터페이스로 전달한다.

## Why

`SetParameterValue` / `GetParameterValue`는 키마다 타입이 다른 값을 주고받아야 한다.  
`void*`나 `std::string` 직렬화 대신 Variant를 사용해 int, float, bool, std::string을 하나의 타입으로 표현한다.  
단, 타입 불일치는 컴파일 타임에 잡히지 않고 묵음 실패(silent failure) 또는 기본값 반환으로 이어지므로  
SetParameter 쪽과 GetParameter 쪽의 타입이 반드시 일치해야 한다.

## Pattern

```cpp
// --- 파라미터 쓰기 (ESMain 레이어) ---
// "SWEAcqAssist.State"는 int Variant
Variant stateVal(static_cast<int>(SWEAcqAssistState::Running));
m_pParamBus->SetParameterValue("SWEAcqAssist.State", stateVal);

// "SWEAcqAssist.RoiX"는 float Variant
Variant roiXVal(m_roiX);   // float
m_pParamBus->SetParameterValue("SWEAcqAssist.RoiX", roiXVal);

// --- 파라미터 읽기 (OVObject 레이어) ---
bool OnSetParameter(const std::string& key, const Variant& value) override
{
    if (key == "SWEAcqAssist.State")
    {
        // AsInt() — 타입이 맞으면 값 반환, 틀리면 0 반환 (묵음 실패)
        m_state = static_cast<SWEAcqAssistState>(value.AsInt());
        return true;
    }
    if (key == "SWEAcqAssist.RoiX")
    {
        m_roiX = value.AsFloat();  // float Variant에 AsInt() 호출 시 0 반환
        return true;
    }
    return false;
}

// --- 타입별 접근자 ---
// value.AsInt()     → int   (Variant가 int가 아니면 0)
// value.AsFloat()   → float (Variant가 float이 아니면 0.0f)
// value.AsBool()    → bool  (Variant가 bool이 아니면 false)
// value.AsString()  → std::string (Variant가 string/BSTR이 아니면"")
```

## Key Points

- Variant는 int / float / bool / std::string / BSTR을 담는 discriminated union이다.
- SetParameter 호출부와 수신부의 타입이 다르면 AsXxx()가 기본값을 반환하며 오류 로그가 없다 — 버그가 조용히 숨는다.
- `"SWEAcqAssist.State"` → int, `"SWEAcqAssist.RoiX"` → float: 키별 타입은 도메인 규칙이므로 팀 내 문서화 필수.
- bool Variant와 int Variant는 별개다 — `Variant(1)` 과 `Variant(true)` 를 혼용하지 말 것.
- 새 파라미터 추가 시 SetParameterValue 호출부(ESMain)와 OnSetParameter 수신부(OVObject)에 같은 타입 명시.
