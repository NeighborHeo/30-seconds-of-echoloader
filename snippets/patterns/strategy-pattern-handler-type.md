---
title: "Strategy 패턴: HandlerType enum으로 알고리즘 패밀리 교체"
category: patterns
tags: [strategy, HandlerType, IAcqAssistHandler, SWE, UGAP, runtime-selection]
difficulty: intermediate
---

`Manager::Instance(HandlerType::SWE)` 한 줄로 SWE/UGAP 알고리즘 패밀리 전체를 교체한다 — if/else 분기를 인터페이스 뒤에 숨기는 Strategy 패턴.

## Why

SWE와 UGAP은 프레임 처리, 컬러 매핑, 파라미터 해석 방식이 다르다. if/else로 구분하면 새 모드 추가 시 모든 분기점을 수정해야 하고, 분기 누락이 필드 버그로 이어진다. Strategy 패턴은 "알고리즘 선택"을 한 곳(Factory/Manager)에 모으고, 나머지 코드는 인터페이스만 바라본다.

## Pattern

```cpp
// === Strategy 인터페이스 ===
class IAcqAssistHandler
{
public:
    virtual ~IAcqAssistHandler() {}

    virtual void OnNewFrame(const AcqFrame& frame) = 0;
    virtual void OnParameterChanged(
        const ScCommon::GcString& strKey,
        const ScCommon::GcVariant& varValue) = 0;
    virtual void Reset() = 0;
    virtual EHandlerType GetHandlerType() const = 0;
};

// === Concrete Strategy A: SWE ===
class SWEAcqAssistHandler : public IAcqAssistHandler
{
public:
    virtual void OnNewFrame(const AcqFrame& frame) OVERRIDE
    {
        // SWE 속도 데이터 처리
        m_pVelocityProcessor->Process(frame.GetSWEData());
        m_pColorMapper->Apply(frame, m_sweColorTable);
    }

    virtual void OnParameterChanged(
        const ScCommon::GcString& strKey,
        const ScCommon::GcVariant& varValue) OVERRIDE
    {
        if (strKey == "SWE.PRF")
            m_pVelocityProcessor->SetPRF(varValue.ToFloat());
    }

    virtual void Reset() OVERRIDE
    {
        m_pVelocityProcessor->Reset();
    }

    virtual EHandlerType GetHandlerType() const OVERRIDE
    {
        return EHandlerType::SWE;
    }

private:
    SWEVelocityProcessor* m_pVelocityProcessor;
    SWEColorTable         m_sweColorTable;
    ColorMapper*          m_pColorMapper;
};

// === Concrete Strategy B: UGAP ===
class UGAPAcqAssistHandler : public IAcqAssistHandler
{
public:
    virtual void OnNewFrame(const AcqFrame& frame) OVERRIDE
    {
        // UGAP 전단파 처리 — SWE와 완전히 다른 알고리즘
        m_pShearWaveEngine->ProcessFrame(frame.GetUGAPData());
        m_pElastogramRenderer->Render(m_pShearWaveEngine->GetResult());
    }

    virtual void OnParameterChanged(
        const ScCommon::GcString& strKey,
        const ScCommon::GcVariant& varValue) OVERRIDE
    {
        if (strKey == "UGAP.StiffnessRange")
            m_pElastogramRenderer->SetStiffnessRange(varValue.ToFloat());
    }

    virtual void Reset() OVERRIDE
    {
        m_pShearWaveEngine->Reset();
    }

    virtual EHandlerType GetHandlerType() const OVERRIDE
    {
        return EHandlerType::UGAP;
    }

private:
    UGAPShearWaveEngine*   m_pShearWaveEngine;
    ElastogramRenderer*    m_pElastogramRenderer;
};

// === Context: Manager가 Strategy를 보유하고 위임 ===
class AcqAssistManager
{
public:
    static AcqAssistManager& Instance()
    {
        static AcqAssistManager s_instance;
        return s_instance;
    }

    // 런타임에 Strategy 교체 — 호출자는 구체 타입을 모른다
    void SetHandler(EHandlerType eType)
    {
        ScCommon::ScopedLock lock(m_mutex);

        delete m_pCurrentHandler;
        m_pCurrentHandler = NULL;

        switch (eType)
        {
        case EHandlerType::SWE:
            m_pCurrentHandler = new SWEAcqAssistHandler();
            break;
        case EHandlerType::UGAP:
            m_pCurrentHandler = new UGAPAcqAssistHandler();
            break;
        default:
            ScCommon::Logger::Error("AcqAssistManager: unknown HandlerType");
            break;
        }
    }

    // 호출자는 인터페이스만 사용
    IAcqAssistHandler* GetHandler() const
    {
        return m_pCurrentHandler;
    }

private:
    AcqAssistManager() : m_pCurrentHandler(NULL) {}
    ~AcqAssistManager() { delete m_pCurrentHandler; }

    IAcqAssistHandler* m_pCurrentHandler;
    ScCommon::Mutex    m_mutex;
};

// === 호출 측 (ESMain) — if/else 없음 ===
void ESMain::OnModeChanged(EHandlerType eNewType)
{
    AcqAssistManager::Instance().SetHandler(eNewType);
}

void ESMain::OnNewFrame(const AcqFrame& frame)
{
    IAcqAssistHandler* pHandler = AcqAssistManager::Instance().GetHandler();
    if (pHandler != NULL)
        pHandler->OnNewFrame(frame);  // SWE인지 UGAP인지 ESMain은 모른다
}
```

## Key Points

- **if/else 제거**: 새 모드(예: `EHandlerType::cSWE`) 추가 시 `switch` 한 곳과 새 Concrete Strategy만 추가 — `ESMain`, `DisplayManager` 등 나머지 코드 무수정.
- **인터페이스가 계약**: `IAcqAssistHandler`의 메서드 시그니처가 SWE/UGAP 양쪽이 지켜야 할 행동 스펙이다. 누락하면 컴파일 에러로 즉시 발견.
- **Strategy 교체 시 이전 객체 해제**: `SetHandler()`에서 `delete m_pCurrentHandler` 순서에 주의. 진행 중인 `OnNewFrame()` 호출과 race condition이 생기므로 `ScopedLock` 필수.
- **테스트 격리**: `IAcqAssistHandler*`를 주입받는 구조이므로 `MockAcqAssistHandler`로 단위 테스트 가능 — if/else 스파게티에서는 불가능.
- **`GetHandlerType()` 반환값 활용**: 로깅, 파라미터 버스 키 생성 등에서 현재 Strategy 타입 확인이 필요할 때 사용. 다운캐스트(`dynamic_cast`) 대신 이 메서드를 쓰라.
