---
title: "26년 코드베이스 지도: 어디를 조심하고 어디에 과감하게 짜야 하나"
category: architecture
tags: [legacy, refactoring, esmain, mfc, ovobject, architecture, risk-assessment]
difficulty: advanced
---

26년 C++ 코드베이스는 지형이 고르지 않다. 어디가 지뢰밭이고 어디가 안전 지대인지 모르면 작은 수정이 예상치 못한 파급을 만든다.

## Why / 왜 알아야 하나

신규 개발자가 가장 많이 하는 실수는 두 가지다. 첫째, 오래됐다는 이유로 고위험 파일을 함부로 건드리는 것. 둘째, 반대로 안전한 영역에서도 과도하게 조심하며 속도를 잃는 것. 이 지도는 그 판단을 빠르게 내릴 수 있게 한다.

## 지뢰밭 (고위험 영역 — 수정 전 최대한 이해하라)

### 1. ESMain.cpp (약 9,954줄)

```
위험도: ████████████ 최고
마지막 안전 수정: 언제나 위험
```

ESMain.cpp는 26년간 Layer A 전체의 진입점이었다. 게이팅 로직, 라우팅, 일부 직접 구현 로직이 하나의 파일에 누적되어 있다. 분리 리팩토링은 진행 중이지만 미완이다.

**이 파일에서 무엇을 하나:**
- `AcquisitionAssistant::Manager` 11개 호출 사이트 (라인 7066~9954)
- SWE/UGAP 게이팅 술어: `InElasto()`, `IsInUGAPMode()`
- 프리셋 로딩, 워치독, 부팅/셧다운 경로

**규칙:**
- 새 비즈니스 로직을 ESMain.cpp에 직접 추가하지 않는다
- 호출 사이트를 추가할 때는 기존 11개 사이트 패턴을 정확히 따른다
- 게이팅 없이 `Manager::Instance()` 호출은 절대 금지
- 수정 전 `/* 1998 Vingmed */` 스타일 주석이 있으면 배경 조사 필수

**식별 신호:**
```cpp
// 1998년 Vingmed 시대 주석 스타일
/* ---- EchoScanner main ---- */
// FBUG 87799/CR 27138
// RBUG159787: If we booted up with HW, make sure we clear the reboot retry counter
```

---

### 2. MFC 다이얼로그 클래스 (C* 접두 패턴)

```
위험도: ████████░░░░ 높음
마지막 안전 수정: 기존 패턴 그대로 복사할 때만
```

`CApplicationTouchPanel`, `CTestGraph` 등 `C*` 접두 클래스는 MFC 메시지 맵 매크로와 강하게 결합되어 있다. 생성자에서 많은 초기화가 일어나고, 직접 ESMain을 호출하는 경우가 있다.

```cpp
// MFC 메시지 맵 — 순서와 매크로 규칙이 고정
BEGIN_MESSAGE_MAP(CApplicationTouchPanel, CDialog)
    ON_WM_PAINT()
    ON_BN_CLICKED(IDC_BUTTON_FREEZE, OnFreezeClicked)
END_MESSAGE_MAP()
```

**규칙:**
- 기존 C* 다이얼로그 수정 시 패턴을 그대로 유지한다
- 새 MFC 다이얼로그 신규 작성은 금지 — 기존 것을 확장한다
- `DoDataExchange()`와 메시지 맵 순서를 임의로 변경하지 않는다

---

### 3. COM/ATL 인터페이스 코드

```
위험도: ████████░░░░ 높음
변경 비용: GUID 변경 + 레지스트리 + 모든 구현체 업데이트
```

COM 인터페이스는 GUID가 코드와 레지스트리에 하드코딩되어 있다. 인터페이스를 변경하면 새 GUID를 발급하고 모든 구현체를 업데이트해야 한다. 하위 호환성이 깨질 수 있다.

```cpp
// COM GUID — 변경 시 레지스트리 + 모든 CoCreateInstance 사이트 영향
// MIDL_INTERFACE("a1b2c3d4-...")
// interface IAcqAssistCOM : IUnknown { ... };
```

**규칙:** COM 인터페이스 수정 = 새 GUID + Director 버전 협의 필수. 단독 판단으로 변경 금지.

---

## 안전 지대 (과감하게 짜도 되는 영역)

### 1. 새 OVObject 클래스

```
위험도: ░░░░░░░░░░░░ 낮음
변경 범위: 새 파일 + 팩토리 등록 2줄
```

OVObject 패턴은 격리가 잘 되어 있다. 팩토리에 태그를 등록하고 `OnActivate()`, `Render()`, `SetParameter()` 세 메서드를 구현하면 기존 코드를 거의 건드리지 않는다.

```cpp
// OVObject.cpp 팩토리 등록 — 이 2줄이 전부
REGISTER_OV_OBJECT("Gc.MyNewAssist", MyNewAssistant)
// 나머지는 모두 새 파일 안에서 해결

class MyNewAssistant : public OVObject {
public:
    void OnActivate() override   { /* 초기화 */ }
    void Render() override       { /* 렌더링 */ }
    void SetParameter(const std::string& key, const Variant& val) override
    {
        if (key == GcKeys::MyParam)
            m_value = val.AsFloat();
    }
};
```

ESMain.cpp는 변하지 않는다. 빌드 영향 범위가 새 파일 2개로 제한된다.

---

### 2. AcquisitionAssistant 내부 로직 (Handler/Manager 내부)

```
위험도: ░░░░░░░░░░░░ 낮음
조건: IAcqAssistHandler 인터페이스 시그니처를 지킬 때
```

`SWEAcqAssistHandler`, `UGAPAcqAssistHandler` 내부 구현은 `IAcqAssistHandler` 인터페이스 뒤에 캡슐화되어 있다. 인터페이스 시그니처만 유지하면 내부 알고리즘은 자유롭게 변경할 수 있다.

```
외부 계약 (변경 금지)          내부 구현 (자유롭게 변경 가능)
─────────────────────────       ────────────────────────────────
IAcqAssistHandler::             SWEAcqAssistHandler::
  UpdateAcqAssistState()    →     내부 상태머신 로직
  ResolveAutoPosKey()       →     키 반환 로직
  GetCurrentState()         →     상태 반환 로직
```

단, SWE와 UGAP은 절대 합치지 않는다. 게이팅 술어, 상태머신, ROI 결과 타입이 모두 다르다.

---

### 3. ScCommon 래퍼 사용 코드

```
위험도: ░░░░░░░░░░░░ 낮음
조건: ScCommon 패턴을 그대로 따를 때
```

`ScCommon::Thread`, `ScCommon::Mutex`, `ScCommon::Timer`를 사용하는 코드는 스레드 안전 패턴이 이미 내장되어 있다. 이 패턴대로 새 코드를 작성하면 스레드 안전성을 별도로 고민할 필요가 없다.

```cpp
// ScCommon 패턴 — 안전, 그대로 따를 것
{
    ScCommon::ScopedLock lock(m_mutex);
    m_sharedData = newValue;
}  // ← 자동 해제

// 직접 std::mutex 없이 위험한 패턴
m_mutex.lock();
m_sharedData = newValue;
m_mutex.unlock();  // ← 예외 발생 시 unlock 누락 위험
```

---

## 레거시 코드를 만났을 때 체크리스트

수정 대상 파일을 열었을 때 다음 항목을 확인한다:

```
[ ] 마지막 수정일이 5년 이상 전인가?          (git log --follow -- <file>)
[ ] 파일 상단에 오래된 저작권 연도가 있나?     (/* Copyright (c) 1998-2005 ... */)
[ ] 헝가리안 표기법이 쓰이나?                 (pszBuffer, dwSize, lpfnCallback)
[ ] 주석이 없거나 미완성 설명만 있나?
[ ] ESMain.cpp 또는 MFC C* 클래스에서 직접 호출되나?
[ ] COM GUID가 파일 내에 있나?

→ 3개 이상 해당: 수정 전 아키텍처 리뷰 필요
→ 5개 이상 해당: Director 확인 없이 변경 금지
```

## 신규 코드 의사결정 트리

```
새 기능을 추가해야 한다
        │
        ▼
ESMain 수정 없이 완성 가능한가?
    ├─ YES → OVObject 패턴으로 작성
    │         GC 파라미터 버스로 통신
    │         ScCommon::ScopedLock 사용
    │         → 안전, 진행
    │
    └─ NO  → ESMain 호출 사이트 추가 필요한가?
                ├─ YES → 기존 11개 사이트 패턴 정확히 따름
                │         게이팅 술어 반드시 추가
                │         아키텍처 리뷰 후 진행
                │
                └─ COM 인터페이스 변경 필요한가?
                          → Director 협의 후 새 GUID 발급
```

## 고위험/저위험 영역 요약

| 영역 | 위험도 | 변경 방식 |
|------|--------|-----------|
| `ESMain.cpp` | 최고 | 기존 패턴 엄격히 따름, 신규 로직 추가 금지 |
| MFC `C*Dialog` 클래스 | 높음 | 패턴 복사+수정, 신규 다이얼로그 작성 금지 |
| COM/ATL 인터페이스 | 높음 | GUID 변경 + Director 협의 필수 |
| 기존 OVObject 내부 | 중간 | 인터페이스 시그니처 유지 시 내부 자유 |
| **새 OVObject 클래스** | **낮음** | **과감하게 작성, 격리 보장** |
| **Handler/Manager 내부** | **낮음** | **인터페이스 유지 시 자유** |
| **ScCommon 패턴 코드** | **낮음** | **패턴 그대로, 스레드 안전 자동** |

## Key Points

- `ESMain.cpp` 9,954줄은 신규 로직 추가 금지 구역이다 — 기존 11개 호출 사이트 패턴을 따르는 수정만 허용한다.
- 레거시 코드 5개 항목 중 3개 이상 해당하면 수정 전 아키텍처 리뷰가 필요하다 — 혼자 판단하지 않는다.
- 새 OVObject 클래스는 가장 안전한 확장 지점이다 — ESMain.cpp를 건드리지 않고 기능을 추가할 수 있다.
- COM GUID 변경은 레지스트리와 모든 구현체를 동시에 업데이트해야 한다 — 단독 판단으로 절대 변경하지 않는다.
- `InElasto()` / `IsInUGAPMode()` 게이팅은 코드가 아니라 임상 안전 요구사항이다 — 누락하면 임상 데이터 오염이다.
