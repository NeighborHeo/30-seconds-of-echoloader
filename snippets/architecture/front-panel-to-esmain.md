---
title: "EchoFrontPanel → ESMain 입력 흐름"
category: architecture
tags: [front-panel, esmain, message-queue, ui-thread, touch-panel, mfc]
difficulty: beginner
---

물리적 컨트롤(버튼, 트랙볼, 터치 패널)의 입력은 EchoFrontPanel에서 메시지 큐를 통해 ESMain 이벤트 루프로 전달되며, ESMain이 모드 게이팅 후 핸들러로 위임한다.

## Why

EchoFrontPanel은 하드웨어 추상화 계층이다. FrontPanel이 EchoScanner에 직접 의존하지 않고 메시지 버스를 통해 통신하기 때문에, 하드웨어 플랫폼이 바뀌어도 EchoScanner 코드를 건드리지 않아도 된다. 이 흐름을 모르면 물리 버튼과 소프트웨어 핸들러 사이 어디서 이벤트가 소실되는지 추적하기 어렵다. 특히 UI 스레드 친화도(COM Apartment 규칙)를 위반하면 MFC 메시지 펌프가 멈추는 무반응 버그가 나타난다.

## Pattern

```cpp
// ─── Layer 0: 물리 하드웨어 ───────────────────────────────────────────────
// 버튼/트랙볼 → HW 드라이버 → OS 메시지

// ─── Layer 1: EchoFrontPanel ─────────────────────────────────────────────
// CApplicationTouchPanel (MFC): 터치 패널 입력 처리
class CApplicationTouchPanel : public CWnd
{
protected:
    // MFC 메시지 핸들러 — 반드시 UI 스레드에서 실행 (COM STA 규칙)
    afx_msg void OnLButtonDown(UINT nFlags, CPoint point)
    {
        // 터치 좌표 → 내부 이벤트 구조체로 변환
        FrontPanelEvent evt;
        evt.m_eType  = EFPEventType::SelectDn;
        evt.m_nPosX  = point.x;
        evt.m_nPosY  = point.y;
        evt.m_bShift = (nFlags & MK_SHIFT) != 0;

        // 메시지 큐에 게시 — EchoScanner에 직접 의존하지 않음
        PostFrontPanelEvent(evt);
    }
};

// 트랙볼: 별도 HID 드라이버 경로 → 같은 큐로 수렴
class CEchoTrackball
{
public:
    void OnTrackballMove(int nRawDx, int nRawDy)
    {
        FrontPanelEvent evt;
        evt.m_eType = EFPEventType::TrbMove;
        evt.m_nDx   = nRawDx;
        evt.m_nDy   = nRawDy;
        PostFrontPanelEvent(evt);
    }
};

// 키 바인딩: 물리 버튼 → 논리 액션 매핑
// AutoPos 키는 "SWEAcqAssistToggleContinuous" 액션에 매핑
void RegisterKeyBindings()
{
    // 물리 버튼 ID → 액션 문자열
    m_keyMap["BTN_AUTOPOS"] = "SWEAcqAssistToggleContinuous";
    m_keyMap["BTN_SELECT"]  = "AcqAssistSelectDn";
}

// ─── Layer 2: ESMain 이벤트 루프 ─────────────────────────────────────────
class CEchoScanMain
{
public:
    // 메시지 큐에서 FrontPanelEvent를 꺼내 처리
    void ProcessFrontPanelEvent(const FrontPanelEvent& evt)
    {
        // 모드 게이팅: InElasto / IsInUGAPMode
        // (상세 흐름은 user-input-event-routing.md 참조)
        RouteToAcqAssistHandler(evt);
    }
};
```

## Key Points

- **FrontPanel ↔ EchoScanner 직접 의존 없음**: 메시지 큐(PostFrontPanelEvent)로만 통신 — 하드웨어 교체 시 FrontPanel 구현만 교체하면 됨
- **CApplicationTouchPanel은 MFC CWnd 서브클래스**: OnLButtonDown 등 MFC 메시지 핸들러가 터치 입력을 처리하며, 반드시 UI 스레드(COM STA)에서 실행되어야 함
- **트랙볼과 터치 패널은 같은 FrontPanelEvent 큐로 수렴**: 입력 소스가 달라도 ESMain은 동일한 이벤트 구조체를 처리 — 라우팅 로직 단순화
- **키 바인딩은 물리 버튼 ID → 논리 액션 문자열**: `"SWEAcqAssistToggleContinuous"` 같은 액션 문자열이 중간 계층 역할 — 하드웨어 레이아웃 변경 시 키맵 테이블만 수정
- **UI 스레드 친화도 위반 = MFC 무반응**: FrontPanel 핸들러에서 다른 스레드의 함수를 직접 호출하면 COM Apartment 규칙 위반 → 메시지 큐를 통한 마샬링 필수
