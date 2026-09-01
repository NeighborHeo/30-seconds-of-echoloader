---
title: "const 정확성 패턴"
category: cpp-patterns
tags: [const, constexpr, const-correctness, naming]
difficulty: beginner
---

읽기 전용 메서드와 매개변수에 `const`를 명시해 의도를 컴파일러가 검증하게 한다.

## Why

`const`는 컴파일러가 강제하는 문서다. 호출자는 `const` 메서드를 보고 사이드 이펙트 없음을 확신할 수 있고, `const&` 매개변수는 불필요한 복사를 막는다. gipc-app MFC 레거시에는 `const` 누락 getter가 있지만 신규 코드는 올바른 패턴을 따른다.

## Pattern

```cpp
class ProbeManager {
public:
    // 읽기 전용 메서드: const 명시
    bool IsReady() const { return m_bReady; }
    const std::string& GetName() const { return m_name; }
    const ProbeState& GetProbeState() const { return m_probeState; }

    // 매개변수: 읽기 전용 복합 타입은 const& 전달
    bool Init(const std::string& presetName, std::string& errorMsg);
    //         ^^^^^^^^^^^^^^^^ 읽기만 함   ^^^^^^^^^^^^^ 출력용 out-param

    // constexpr 상수: PascalCase (gipc-app 명명 규칙)
    static constexpr int Course = 1;
    static constexpr int MaxChannels = 192;

    // override 명시: 가상 함수 재정의 시 필수
    void OnProbeConnected() override;
    void OnProbeDisconnected() override;

private:
    bool        m_bReady;
    std::string m_name;
    ProbeState  m_probeState;
};

// 레거시 패턴 — const 누락 (기술 부채, 수정 자제)
class CLegacyDialog : public CDialog {
public:
    bool GetSomeFlag() { return m_bFlag; }  // const 빠짐 — 레거시 시그니처 유지
private:
    bool m_bFlag;
};

// 포인터 const 구분
void Process(const ProbeState* pState);   // 가리키는 객체 변경 불가
void Register(ProbeState* const pState);  // 포인터 자체 변경 불가 (드문 패턴)
```

## Key Points

- 상태를 바꾸지 않는 메서드는 전부 `const` — "나중에 붙이기"는 없다. 처음부터 붙인다
- `const std::string&` 반환: 임시 객체 반환 시 댕글링 레퍼런스 주의. 멤버 변수 레퍼런스만 반환
- `constexpr` 상수는 PascalCase: `constexpr int MaxChannels = 192;` (`MACRO_CASE` 대신)
- `[[nodiscard]]`는 C++17 — gipc-app(C++11/14)에서 사용 금지
- `override` 키워드: 신규 코드 필수. MFC 레거시 일부는 미적용 상태 (기술 부채)
- MFC getter의 `const` 누락은 알려진 기술 부채 — 시그니처 변경은 const-overload 연쇄를 유발하므로 단독 수정 자제
