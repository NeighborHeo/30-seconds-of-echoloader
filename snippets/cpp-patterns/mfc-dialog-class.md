---
title: "MFC CDialog 파생 클래스 패턴"
category: cpp-patterns
tags: [MFC, CDialog, UI, legacy]
difficulty: intermediate
---

MFC 다이얼로그 클래스는 `C` 접두사 + Dialog Template 연결 + `m_b*` bool 멤버 패턴을 따른다.

## Why

gipc-app은 MFC 기반 UI를 대규모로 사용한다. 팀 전체가 동일한 명명 규칙을 따라야 코드 탐색과 리뷰가 가능하다. 레거시 코드에는 알려진 기술 부채(getter const 누락)가 존재하지만, 신규 코드는 올바른 패턴을 따른다.

## Pattern

```cpp
// MyDialog.h
class CApplicationTouchPanel : public CDialog {
    DECLARE_DYNAMIC(CApplicationTouchPanel)

public:
    // 생성자: 리소스 ID 연결
    explicit CApplicationTouchPanel(CWnd* pParent = nullptr);
    virtual ~CApplicationTouchPanel();

    // 다이얼로그 데이터
    enum { IDD = IDD_APPLICATION_TOUCH_PANEL };

    // bool 멤버: m_b* 접두사
    bool m_bNeedDrawAgain;
    bool m_bIsInitialized;

    // 신규 코드: getter에 const 명시
    bool IsNeedDrawAgain() const { return m_bNeedDrawAgain; }

    // 레거시 패턴: const 누락 (기술 부채, 수정 자제 — 시그니처 변경이 연쇄 영향)
    bool GetSomeValue() { return m_bSomeValue; }

protected:
    virtual void DoDataExchange(CDataExchange* pDX) override;
    virtual BOOL OnInitDialog() override;

    DECLARE_MESSAGE_MAP()
    afx_msg void OnBnClickedOk();

private:
    bool m_bSomeValue;
};

// MyDialog.cpp
#ifdef _DEBUG
#define new DEBUG_NEW  // 메모리 릭 감지 (Debug 빌드 전용)
#endif

IMPLEMENT_DYNAMIC(CApplicationTouchPanel, CDialog)

CApplicationTouchPanel::CApplicationTouchPanel(CWnd* pParent)
    : CDialog(IDD, pParent)
    , m_bNeedDrawAgain(false)
    , m_bIsInitialized(false)
    , m_bSomeValue(false)
{}

void CApplicationTouchPanel::DoDataExchange(CDataExchange* pDX) {
    CDialog::DoDataExchange(pDX);
    // DDX_Control(pDX, IDC_BUTTON1, m_btnOk);
}

// MFC 기반 인터랙티브 테스트 패턴
class CTestGraph : public CDialog {
    // 프로덕션 다이얼로그와 동일한 구조
    // 수동 QA 및 개발 중 시각 확인용
public:
    CTestGraph() : CDialog(IDD_TEST_GRAPH) {}
};
```

## Key Points

- 클래스명은 반드시 `C` 접두사: `CApplicationTouchPanel`, `CSweControlPanel`
- bool 멤버는 `m_b*`: `m_bNeedDrawAgain`, `m_bIsVisible`
- `#define new DEBUG_NEW`는 `_DEBUG` 빌드에서만 — .cpp 파일 상단, include 이후
- `DoDataExchange`에서 `DDX_*` 매크로로 컨트롤 바인딩. 직접 `GetDlgItem` 남용 금지
- 레거시 getter의 `const` 누락은 알려진 기술 부채 — 시그니처 수정은 연쇄 영향이 크므로 신중히
- `CTestGraph` 같은 테스트 다이얼로그: 수동 인터랙티브 테스트용. 자동화 테스트 대체가 어려운 UI 검증에만 사용
- DIALOG TEMPLATE(리소스 파일 `.rc`)와 `IDD` enum 값이 반드시 일치해야 함
