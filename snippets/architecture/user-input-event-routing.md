---
title: "사용자 입력 이벤트 라우팅: FrontPanel → ESMain → Handler"
category: architecture
tags: [event-routing, front-panel, esmain, handler, gating]
difficulty: intermediate
---

하드웨어 입력(버튼/트랙볼)은 EchoFrontPanel에서 ESMain으로 전달되고, ESMain이 모드 게이팅 후 적절한 IAcqAssistHandler로 위임한다.

## Why

gipc-app은 SWE, UGAP, 일반 B-mode 등 여러 촬영 모드를 동시에 지원한다. 같은 물리 버튼(SelectDn)이 모드에 따라 완전히 다른 의미를 갖기 때문에, ESMain이 중앙 게이팅 포인트 역할을 한다. 게이팅 조건을 누락하면 잘못된 핸들러가 호출되어 아무런 에러 없이 조용히 실패하거나, 다른 모드의 ROI를 덮어쓰는 부작용이 생긴다.

## Pattern

```cpp
// ESMain 이벤트 디스패치 (단순화된 구조)
// 이벤트 구조체: 타입 + 좌표 + 수식키 + 타임스탬프
struct AcqAssistInputEvent
{
    EInputEventType m_eType;       // SelectDn, TrbMove, CursorMoved, ...
    int             m_nPosX;
    int             m_nPosY;
    bool            m_bShiftDown;
    bool            m_bCtrlDown;
    DWORD           m_dwTimestamp;
};

// ESMain: 모드 게이팅 후 핸들러 위임
void CEchoScanMain::OnAcqAssistInput(const AcqAssistInputEvent& event)
{
    IAcqAssistHandler* pHandler = nullptr;

    if (InElasto())
    {
        pHandler = m_pSWEAcqAssistHandler;   // SWE 핸들러
    }
    else if (IsInUGAPMode())
    {
        pHandler = m_pUGAPAcqAssistHandler;  // UGAP 핸들러
    }

    if (!pHandler)
        return;  // 해당 모드 없으면 무시

    switch (event.m_eType)
    {
    case EInputEventType::SelectDn:
        pHandler->HandleSelectDn(event.m_nPosX, event.m_nPosY);
        break;
    case EInputEventType::SelectUp:
        pHandler->HandleSelectUp(event.m_nPosX, event.m_nPosY);
        break;
    case EInputEventType::TrbMove:
        pHandler->HandleTrbMove(event.m_nPosX, event.m_nPosY);  // ROI 이동
        break;
    case EInputEventType::SelectMoved:
        pHandler->HandleSelectMoved(event.m_nPosX, event.m_nPosY);
        break;
    case EInputEventType::DoubleSelectDn:
        pHandler->HandleDoubleSelectDn(event.m_nPosX, event.m_nPosY);
        break;
    case EInputEventType::LongSelectDn:
        pHandler->HandleLongSelectDn(event.m_nPosX, event.m_nPosY);
        break;
    case EInputEventType::SelectCancel:
        pHandler->HandleSelectCancel();
        break;
    default:
        break;
    }
}

// IAcqAssistHandler 인터페이스 (순수 가상)
class IAcqAssistHandler
{
public:
    virtual void HandleSelectDn(int nX, int nY)       = 0;  // 버튼 누름
    virtual void HandleSelectUp(int nX, int nY)       = 0;  // 버튼 뗌
    virtual void HandleTrbMove(int nDx, int nDy)      = 0;  // 트랙볼 이동 (델타값)
    virtual void HandleSelectMoved(int nX, int nY)    = 0;  // 드래그 중
    virtual void HandleDoubleSelectDn(int nX, int nY) = 0;  // 더블클릭
    virtual void HandleLongSelectDn(int nX, int nY)   = 0;  // 길게 누름
    virtual void HandleSelectCancel()                  = 0;  // 취소
};
```

## Key Points

- **7개 이벤트 타입**: SelectDn/SelectUp(클릭), TrbMove(트랙볼 델타), SelectMoved(드래그), DoubleSelectDn/LongSelectDn(특수 클릭), SelectCancel — 모두 IAcqAssistHandler 인터페이스에 대응
- **게이팅 순서가 중요**: `InElasto()` 체크를 `IsInUGAPMode()` 보다 먼저 수행 — SWE/UGAP 동시 활성화 시나리오에서 우선순위가 결정됨
- **TrbMove는 절대 좌표가 아닌 델타값**: 트랙볼 물리 특성상 누적 이동량을 핸들러에서 ROI 중심에 더해야 함
- **SelectDn의 의미가 모드마다 다름**: SWE에서는 측정 시작 트리거, UGAP에서는 ROI 위치 확정 — 같은 이벤트가 완전히 다른 도메인 동작을 유발
- **이벤트 누락 = 조용한 실패**: 핸들러 미구현 시 컴파일 에러가 아닌 런타임 무반응 — IAcqAssistHandler에 순수 가상 함수로 강제하는 이유
