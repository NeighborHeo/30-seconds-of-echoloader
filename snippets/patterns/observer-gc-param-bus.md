---
title: "Observer 패턴: GC 파라미터 버스를 통한 레이어 간 통신"
category: patterns
tags: [observer, gc-param-bus, OVObject, decoupling, SWE, UGAP]
difficulty: intermediate
---

레이어 A가 레이어 B를 직접 참조하지 않고 GC 파라미터 버스를 통해 상태를 전파한다 — gipc-app에서 SWE/UGAP 레이어 간 결합을 끊는 핵심 메커니즘.

## Why

직접 콜백이나 포인터 전달로 레이어를 연결하면 컴파일 의존성이 생기고, 단위 테스트 시 목(mock) 치환이 어렵다. GC 파라미터 버스는 Publisher/Subscriber를 물리적으로 분리한다: Publisher는 key-value만 게시하고, Subscriber는 `OVObject::SetParameter()`를 구현하여 수신한다. 두 쪽 모두 서로의 헤더를 include할 필요가 없다.

## Pattern

```cpp
// === Publisher 쪽 (예: SWEAcqAssist) ===
// 레이어 A는 레이어 B 타입을 전혀 모른다.
void SWEAcqAssist::OnFrameProcessed(const AcqFrame& frame)
{
    // GC 파라미터 버스에 상태 게시
    // "SWEAcqAssist.State" 키를 구독하는 모든 OVObject가 수신한다.
    ScCommon::GcParamBus::SetParameterValue(
        "SWEAcqAssist.State",
        static_cast<int>(m_eCurrentState));

    ScCommon::GcParamBus::SetParameterValue(
        "SWEAcqAssist.FrameCount",
        m_nProcessedFrameCount);
}

// === Subscriber 쪽 (예: SWEDisplayManager) ===
// OVObject를 상속받으면 파라미터 버스 수신 인터페이스가 자동으로 붙는다.
class SWEDisplayManager : public OVObject
{
public:
    // OVObject::SetParameter() 오버라이드 — 버스가 호출한다
    virtual void SetParameter(
        const ScCommon::GcString& strKey,
        const ScCommon::GcVariant& varValue) OVERRIDE
    {
        if (strKey == "SWEAcqAssist.State")
        {
            int nState = varValue.ToInt();
            HandleStateChange(static_cast<EAcqState>(nState));
        }
        else if (strKey == "SWEAcqAssist.FrameCount")
        {
            m_nLastKnownFrameCount = varValue.ToInt();
        }
        // 알 수 없는 키는 부모로 위임 (다른 구독자 체인 유지)
        else
        {
            OVObject::SetParameter(strKey, varValue);
        }
    }

private:
    void HandleStateChange(EAcqState eState);
    int m_nLastKnownFrameCount;
};

// === 구독 등록 (초기화 시점) ===
void SWEDisplayManager::Initialize()
{
    // 관심 키를 버스에 등록 — Publisher가 누구인지 알 필요 없다
    ScCommon::GcParamBus::Subscribe("SWEAcqAssist.State", this);
    ScCommon::GcParamBus::Subscribe("SWEAcqAssist.FrameCount", this);
}

void SWEDisplayManager::Shutdown()
{
    // RAII가 없으므로 Shutdown에서 명시적으로 해제 (누락 시 dangling pointer)
    ScCommon::GcParamBus::Unsubscribe("SWEAcqAssist.State", this);
    ScCommon::GcParamBus::Unsubscribe("SWEAcqAssist.FrameCount", this);
}
```

## Key Points

- **물리적 분리**: `SWEAcqAssist.h`와 `SWEDisplayManager.h`가 서로를 include하지 않는다 — 빌드 의존성 사이클 차단.
- **키 문자열이 계약(contract)**: `"SWEAcqAssist.State"` 같은 키는 사실상 인터페이스다. 오타는 런타임 침묵 실패이므로, 전용 상수 헤더(`GcParamKeys.h`)에 `const char* kSWEAcqAssistState = "SWEAcqAssist.State";` 형태로 정의하라.
- **스레드 안전성**: `SetParameterValue()`가 UI 스레드와 Acq 스레드 양쪽에서 호출될 수 있다. 버스 구현이 내부 락을 갖더라도, `SetParameter()` 핸들러 안에서 오래 걸리는 작업은 금물.
- **Unsubscribe 누락 = 크래시**: `OVObject` 소멸 전에 반드시 Unsubscribe. 소멸자에서 처리하거나 RAII 래퍼로 감싸는 것이 이상적이다.
- **UGAP에도 동일 패턴 적용**: Publisher 키 이름만 `"UGAPAcqAssist.State"`로 바뀔 뿐, Subscriber 구조는 동일 — 전략 교체(SWE↔UGAP) 시 Subscriber 코드 수정 불필요.
