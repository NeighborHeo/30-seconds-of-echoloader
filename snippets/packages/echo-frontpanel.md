---
title: "EchoFrontpanel / EchoTouchPanel — 물리 프론트패널 & 터치스크린 UI"
category: packages
tags: [MFC, touch, hardware, event, state-machine, C-prefix]
difficulty: intermediate
---

물리 버튼·노브와 터치스크린 입력을 받아 AA 상태머신에 이벤트를 전달하는 하드웨어-소프트웨어 경계 패키지. MFC C접두 명명 패턴과 정적 멤버 패턴이 사용된다.

## 역할

- 물리 프론트패널(버튼, 노브, 트랙볼) 입력 이벤트 처리
- 터치스크린 입력을 애플리케이션 커맨드로 변환
- 하드웨어 이벤트 → AA 상태머신 입력 경로 제공
- `CApplicationTouchPanel` — MFC C접두 패턴의 터치 패널 클래스

## 위치

```
src/packages/EchoFrontpanel/
├── CApplicationTouchPanel.h/.cpp  ← MFC 터치패널 (C접두 패턴)
├── CFrontPanelController.h/.cpp   ← 물리 버튼/노브 이벤트 처리
├── HardwareEventDispatcher.h/.cpp ← 이벤트 → AA 상태머신 라우팅
└── TouchPanelLayout.h             ← 터치 영역 레이아웃 정의
```

## 핵심 클래스

| 클래스 | 역할 |
|---|---|
| `CApplicationTouchPanel` | MFC 기반 터치스크린 입력 처리 (정적 멤버 `s_applicationTouchPanel`) |
| `CFrontPanelController` | 물리 버튼·노브 HID 이벤트 수신 및 처리 |
| `HardwareEventDispatcher` | 하드웨어 이벤트를 AA Manager로 라우팅 |

## 정적 멤버 패턴

```cpp
// CApplicationTouchPanel.h
class CApplicationTouchPanel : public CWnd {
    static CApplicationTouchPanel* s_applicationTouchPanel; // 전역 접근점
    // ...
};
```

## 의존 관계

```
EchoFrontpanel
  → EchoScanner  (IAcqAssistManager — 버튼 → AA 상태 전환)
  → GcViewer     (터치 영역 렌더링 좌표 참조)
  ← EchoLoader   (하드웨어 초기화)
```

## 주의사항

- `s_applicationTouchPanel` 정적 멤버: 멀티 모니터·멀티 터치 환경에서 단일 인스턴스 가정 깨질 수 있음
- MFC `CWnd` 기반이므로 메시지 루프 외부에서 UI 접근 금지 (크래시 원인)
- 물리 패널 이벤트와 터치 이벤트가 동시 발생 시 순서 보장 필요 — `HardwareEventDispatcher` 큐 확인
