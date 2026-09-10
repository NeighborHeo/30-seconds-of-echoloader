---
title: "Facade 패턴: Manager 클래스로 SWE/UGAP 복잡성 캡슐화"
category: patterns
tags: [facade, singleton, Manager, ESMain, lifecycle, SWE, UGAP]
difficulty: intermediate
---

`ESMain`은 `Manager::Instance()->OnNewFrame(frame)` 한 줄만 안다 — Handler 생성/소멸, 상태 머신, 파라미터 라우팅은 Manager Facade 뒤에 숨는다.

## Why

`ESMain`이 Handler 내부에 직접 접근하면: Handler 생성 순서, NULL 검사, 파라미터 배포 로직이 `ESMain` 곳곳에 흩어진다. 서브시스템 내부 변경이 `ESMain` 수정을 유발한다. Facade는 서브시스템의 "단일 진입 창구"로서 이 결합을 끊는다.

## Pattern

```cpp
// === Before: ESMain이 내부를 직접 조작 (Facade 없음) ===
// 이런 코드가 ESMain 여기저기 퍼진다
void ESMain::OnNewFrameBefore(const AcqFrame& frame)
{
    // ESMain이 Handler 상태를 직접 판단해야 함
    if (m_eCurrentMode == EMode::SWE)
    {
        if (m_pSWEHandler != NULL && m_pSWEHandler->IsReady())
        {
            m_pSWEHandler->PrepareFrame(frame);
            m_pSWEHandler->ProcessVelocity(frame);
            m_pSWEHandler->UpdateDisplay();
        }
    }
    else if (m_eCurrentMode == EMode::UGAP)
    {
        if (m_pUGAPHandler != NULL)
        {
            m_pUGAPHandler->ValidateFrame(frame);
            m_pUGAPHandler->ComputeShearWave(frame);
        }
    }
    // 새 모드 추가 시 ESMain 수정 필요 → Open/Closed 위반
}

// ============================================================

// === After: Manager Facade 적용 ===

// Facade: 서브시스템 전체의 단일 진입점
class AcqAssistManager
{
public:
    // ponytail: singleton + facade 조합 — 테스트에서 교체 불가.
    //           테스트 필요 시 인터페이스(IAcqAssistManager)로 추출하고
    //           ESMain에 주입(DI)하는 방식으로 업그레이드.
    static AcqAssistManager& Instance()
    {
        static AcqAssistManager s_instance;
        return s_instance;
    }

    // Facade 인터페이스 — ESMain이 아는 전부
    void OnNewFrame(const AcqFrame& frame)
    {
        ScCommon::ScopedLock lock(m_mutex);

        if (!EnsureHandlerReady())
            return;

        m_pCurrentHandler->OnNewFrame(frame);
        m_nTotalFrameCount++;
    }

    void OnModeChanged(EHandlerType eNewType)
    {
        ScCommon::ScopedLock lock(m_mutex);

        TeardownCurrentHandler();
        m_pCurrentHandler = CreateHandler(eNewType);
        m_eCurrentType = eNewType;

        ScCommon::Logger::Info("AcqAssistManager: mode switched to %d",
                               static_cast<int>(eNewType));
    }

    void OnParameterChanged(
        const ScCommon::GcString& strKey,
        const ScCommon::GcVariant& varValue)
    {
        ScCommon::ScopedLock lock(m_mutex);

        // 파라미터 라우팅 로직도 Facade 안에 — ESMain 무관
        if (IsHandlerParameter(strKey) && m_pCurrentHandler != NULL)
            m_pCurrentHandler->OnParameterChanged(strKey, varValue);
    }

    void Shutdown()
    {
        ScCommon::ScopedLock lock(m_mutex);
        TeardownCurrentHandler();
    }

private:
    AcqAssistManager()
        : m_pCurrentHandler(NULL)
        , m_eCurrentType(EHandlerType::None)
        , m_nTotalFrameCount(0)
    {}

    ~AcqAssistManager()
    {
        TeardownCurrentHandler();
    }

    // 서브시스템 내부 복잡성 — 외부에서 보이지 않는다
    bool EnsureHandlerReady()
    {
        if (m_pCurrentHandler == NULL)
        {
            ScCommon::Logger::Warning(
                "AcqAssistManager::OnNewFrame - no handler, dropping frame");
            return false;
        }
        return true;
    }

    void TeardownCurrentHandler()
    {
        if (m_pCurrentHandler != NULL)
        {
            m_pCurrentHandler->Reset();
            delete m_pCurrentHandler;
            m_pCurrentHandler = NULL;
        }
    }

    IAcqAssistHandler* CreateHandler(EHandlerType eType)
    {
        switch (eType)
        {
        case EHandlerType::SWE:  return new SWEAcqAssistHandler();
        case EHandlerType::UGAP: return new UGAPAcqAssistHandler();
        default:
            ScCommon::Logger::Error("AcqAssistManager: unknown type %d",
                                    static_cast<int>(eType));
            return NULL;
        }
    }

    bool IsHandlerParameter(const ScCommon::GcString& strKey) const
    {
        return strKey.StartsWith("SWE.") || strKey.StartsWith("UGAP.");
    }

    IAcqAssistHandler* m_pCurrentHandler;
    EHandlerType       m_eCurrentType;
    int                m_nTotalFrameCount;
    ScCommon::Mutex    m_mutex;
};

// === ESMain: Facade만 사용, 내부 완전히 몰라도 됨 ===
void ESMain::OnNewFrame(const AcqFrame& frame)
{
    AcqAssistManager::Instance().OnNewFrame(frame);
}

void ESMain::OnModeChanged(EHandlerType eType)
{
    AcqAssistManager::Instance().OnModeChanged(eType);
}
```

## Key Points

- **ESMain의 지식 최소화**: Before에서 ESMain이 알던 `IsReady()`, `PrepareFrame()`, `ProcessVelocity()` 등이 Facade 뒤로 완전히 숨는다 — 새 Handler 추가 시 ESMain 무수정.
- **Singleton + Facade 조합의 트레이드오프**: 간결하지만 단위 테스트에서 Manager를 교체할 수 없다. 테스트가 중요한 경로라면 `IAcqAssistManager` 인터페이스를 추출하고 ESMain에 생성자 주입으로 전환 (`ponytail:` 주석 참조).
- **락 범위**: `OnNewFrame()`이 락을 잡고 들어가므로, Handler 내부에서 Manager를 재진입 호출하면 데드락. Handler 콜백은 락 밖에서 실행하도록 설계하거나 `ScCommon::RecursiveMutex` 검토.
- **Shutdown 명시 호출**: 소멸자에서 `TeardownCurrentHandler()`를 부르지만, 정적 singleton의 소멸 순서는 불확실하다. 프로그램 종료 전 `Shutdown()`을 명시적으로 호출하라.
- **Facade는 팻(Fat) 클래스가 되기 쉽다**: Facade가 비즈니스 로직을 직접 가지기 시작하면 God Object로 타락한다. Facade의 역할은 "위임"이지 "처리"가 아님을 경계하라.
