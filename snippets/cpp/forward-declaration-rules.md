---
title: "전방 선언 vs #include — gipc-app 규칙"
category: cpp
tags: [forward-declaration, include, compile-time, coupling, headers]
difficulty: intermediate
---

~50개 패키지 코드베이스에서 불필요한 `#include`를 전방 선언으로 교체해 컴파일 시간과 결합도를 줄이는 규칙.

## Why

헤더 파일 하나가 수백 개 번역 단위에 포함된다.
넓게 포함된 `.h` 파일에 `#include`가 하나 추가되면 그 파일을 포함하는 모든 `.cpp`가 재컴파일된다.
전방 선언은 의존성 그래프를 끊는 가장 저렴한 방법이다.

## Pattern

```cpp
// ── 전방 선언으로 충분한 경우 ─────────────────────────────────────────

// CFoo.h — 포인터·참조 파라미터만 사용할 때
class CBar;  // 전방 선언으로 충분

class CFoo
{
public:
    void Process(const CBar& bar);  // 참조: 크기 불필요 → 전방 선언 OK
    void Attach(CBar* pBar);        // 포인터: 크기 불필요 → 전방 선언 OK

private:
    CBar* m_pBar = nullptr;         // 포인터 멤버: 전방 선언 OK
};

// CFoo.cpp — 실제 사용 시점에만 #include
#include "CFoo.h"
#include "CBar.h"  // 여기서 #include

void CFoo::Process(const CBar& bar)
{
    bar.DoWork();  // 멤버 접근 → .cpp 에서 완전한 타입 필요
}

// ── #include 가 반드시 필요한 경우 ───────────────────────────────────

// 1. 값 타입 멤버 (크기 알아야 함)
#include "CBar.h"
class CFoo {
    CBar m_bar;  // 값 멤버: sizeof(CBar) 필요 → #include 필수
};

// 2. 상속 (베이스 클래스 레이아웃 필요)
#include "COVObject.h"
class CSWEAcqAssist : public COVObject  // 상속: #include 필수
{
};
// OVObject 서브클래스는 절대 전방 선언으로 상속 불가

// 3. 인라인 함수 본문이 헤더에 있을 때
#include "CBar.h"
class CFoo {
    void Init() { m_pBar->Reset(); }  // 인라인 멤버 접근 → #include 필수
    CBar* m_pBar;
};

// ── gipc-app 크로스 패키지 규칙 ──────────────────────────────────────

// 다른 패키지 타입을 헤더에서 포인터로만 받을 때:
// ScCommon/include/ScCommon/ScLock.h 를 직접 포함하지 말고
class ScCommon_ScScopedLock;  // 전방 선언 (패키지 경계 유지)

// 단, typedef / using / enum 은 전방 선언 불가 → #include 필요
// enum class EAcqState : int;  // C++11 전방 선언 가능 (기반 타입 명시 필수)
```

**빌드 시간 영향 측정 (실제 사례 패턴):**

```
; 넓게 포함된 헤더 (예: COVObjectManager.h) 에서
; #include "CHeavySubsystem.h" → class CHeavySubsystem; 로 교체하면
; CHeavySubsystem.h 를 포함하는 모든 번역 단위가 재컴파일 대상에서 제외됨

; 영향 범위 확인:
; cl /P /EP CFoo.cpp   → 전처리 결과에서 포함 체인 확인
; /showIncludes 플래그로 실제 포함 깊이 측정 가능
```

## Key Points

- `.h` 파일에서는 포인터·참조 파라미터와 포인터 멤버에 **전방 선언**을 쓴다
- 상속, 값 멤버, 인라인 본문에는 `#include`가 반드시 필요하다 — 규칙의 예외가 아니라 필수
- OVObject 서브클래스는 `#include "COVObject.h"` 없이 상속 불가 — 전방 선언으로 상속 시도하면 C2504
- `enum class`는 C++11 이상에서 기반 타입을 명시하면 전방 선언 가능: `enum class EState : int;`
- 크로스 패키지 헤더 하나에서 불필요한 `#include` 제거 한 건이 수십 초 빌드 단축으로 이어질 수 있다
