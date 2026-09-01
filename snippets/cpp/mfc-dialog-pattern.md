---
title: "MFC Dialog Class Pattern"
category: cpp
tags: [mfc, dialog, naming, legacy, ui]
difficulty: intermediate
---

MFC 다이얼로그 클래스는 `C` 접두어 + PascalCase. 레거시 코드이지만 gipc-app UI 레이어를 이해하려면 이 패턴을 알아야 한다.

## Pattern

```cpp
// 파일: ApplicationTouchPanel.h
/*
-GE CONFIDENTIAL-
...
*/
#pragma once
#include "StdAfx.h"
#include <afxwin.h>

// MFC 다이얼로그 서브클래스 — 'C' 접두어 필수
class CApplicationTouchPanel : public CDialog
{
    DECLARE_DYNAMIC(CApplicationTouchPanel)

public:
    CApplicationTouchPanel(CWnd* pParent = nullptr);
    virtual ~CApplicationTouchPanel();

    // 리소스 ID
    enum { IDD = IDD_APPLICATION_TOUCH_PANEL };

    // 데이터 바인딩
    void DoDataExchange(CDataExchange* pDX) override;

    // MFC 메시지 맵
    DECLARE_MESSAGE_MAP()

    afx_msg void OnBnClickedScan();
    afx_msg void OnBnClickedFreeze();

private:
    // 멤버: m_ 접두어 유지
    bool m_bIsVisible  = false;
    int  m_nActiveMode = 0;
};

// 정적 인스턴스 패턴 (싱글톤-ish)
// s_ 접두어
static CApplicationTouchPanel* s_applicationTouchPanel = nullptr;
```

## 인터랙티브 테스트 (레거시)

```cpp
// MFC 다이얼로그 기반 인터랙티브 테스트 — 일부 잔존
// GoogleTest로 대체 중이지만 UI 종속적인 것은 여전히 이 방식
class CTestGraph : public CDialog
{
    // 직접 화면에 띄워서 수동 검증하는 구식 테스트
};
```

## Key Points

- MFC 클래스는 `C` 접두어 + PascalCase: `CApplicationTouchPanel`, `CTestGraph`
- 일반(비MFC) 클래스는 `C` 없이 PascalCase: `ProbeManager`, `AcquisitionAssistantBase`
- `m_b*` bool 멤버, `m_n*` int 멤버: 헝가리안 접미어는 MFC 코드에서 더 강하게 지켜짐
- `const` 누락이 많음 — 레거시 패턴이므로 수정 시 범위 한정해서 변경
- CDialog 서브클래스는 MFC 메시지 루프에 묶여 있어 GoogleTest로 완전 커버 어려움
