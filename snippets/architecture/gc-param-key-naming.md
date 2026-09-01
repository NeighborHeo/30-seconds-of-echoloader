---
title: "GC 파라미터 키 네임스페이스 규칙"
category: architecture
tags: [parameter-bus, naming-convention, key-namespace, esmain, ovovject]
difficulty: intermediate
---

GC 파라미터 키는 `"<ObjectTag>.<PropertyName>"` 형식을 따르며 두 레이어가 같은 문자열을 공유한다.

## Why

ESMain(레이어 A)과 OVObject(레이어 B)는 파라미터 버스를 통해 통신하고 키를 문자열로 지정한다.  
키 오타는 컴파일 타임에 잡히지 않으며 SetParameterValue가 무시되거나 OnSetParameter가 누락된다.  
규칙을 알면 새 키를 예측하고 두 레이어 사이의 동기화 누락을 찾아낼 수 있다.

## Pattern

```cpp
// --- 키 형식: "<ObjectTag>.<PropertyName>" ---
// ObjectTag = OVObject 팩토리 태그에서 "Gc." 제거
//   "Gc.SWEAcqAssist"  →  ObjectTag = "SWEAcqAssist"
//   "Gc.UGAPAcqAssist" →  ObjectTag = "UGAPAcqAssist"

// PropertyName은 PascalCase
//   "SWEAcqAssist.RoiX"
//   "SWEAcqAssist.RoiY"
//   "SWEAcqAssist.State"
//   "SWEAcqAssist.GuidelinesVisible"

// --- 상수 관리 권장 (오타 방지) ---
namespace SWEParamKeys
{
    constexpr const char* State              = "SWEAcqAssist.State";
    constexpr const char* RoiX              = "SWEAcqAssist.RoiX";
    constexpr const char* RoiY              = "SWEAcqAssist.RoiY";
    constexpr const char* GuidelinesVisible = "SWEAcqAssist.GuidelinesVisible";
}

// --- 레이어 A (ESMain): SetParameterValue 호출 ---
void ESMain::OnSWERoiChanged(float x, float y)
{
    m_pParamBus->SetParameterValue(SWEParamKeys::RoiX, Variant(x));
    m_pParamBus->SetParameterValue(SWEParamKeys::RoiY, Variant(y));
}

// --- 레이어 B (OVObject): OnSetParameter 수신 ---
bool SWEAcqAssistObject::OnSetParameter(const std::string& key, const Variant& value)
{
    if (key == SWEParamKeys::RoiX)
    {
        m_roiX = value.AsFloat();
        return true;
    }
    if (key == SWEParamKeys::RoiY)
    {
        m_roiY = value.AsFloat();
        return true;
    }
    return false;
}

// --- 특수 키: 명령(Command) — 동사형 ---
// 상태 조회가 아닌 액션을 트리거하는 키는 동사 또는 동사구
//   "SWEAcqAssistToggleContinuous"   (Toggle + 대상)
//   "UGAPAcqAssistAnalyze"           (동사만)
// PropertyName이 없고 ObjectTag+동사 형태 — 점(.) 없음이 관례
m_pParamBus->SetParameterValue("SWEAcqAssistToggleContinuous", Variant(true));

// --- 새 파라미터 추가 체크리스트 ---
// 1. 상수 네임스페이스에 키 추가
// 2. ESMain(레이어 A)에서 SetParameterValue 호출 추가
// 3. OVObject(레이어 B)의 OnSetParameter에 분기 추가
// 4. Variant 타입이 두 곳에서 일치하는지 확인
```

## Key Points

- 키 형식: `"<ObjectTag>.<PropertyName>"` — ObjectTag는 팩토리 태그(`"Gc.SWEAcqAssist"`)에서 `"Gc."` 제거.
- PropertyName은 PascalCase; 명령 키는 동사형이며 점(`.`) 없이 ObjectTag+동사 형태를 쓴다.
- 키 오타는 런타임 묵음 실패 — 반드시 `constexpr const char*` 상수로 관리해 오타를 컴파일 타임에 차단.
- ESMain(SetParameterValue)과 OVObject(OnSetParameter)는 같은 키 문자열을 공유 — 하나만 바꾸면 통신이 단절된다.
- 새 파라미터 추가 시 레이어 A(호출부)와 레이어 B(수신부)를 동시에 수정해야 하며 Variant 타입도 일치시켜야 한다.
