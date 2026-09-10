---
title: "GMock 없이 인터페이스 목(Mock) 만들기"
category: workflow
tags: [googletest, testing, mock, dependency-injection, seam]
difficulty: intermediate
---

gipc-app에서 GMock 없이 수동 Test Double로 단위 테스트 의존성을 끊는 방법.

## Why

gipc-app은 GoogleTest를 사용하지만 GMock은 무거운 매크로·링크 의존성 때문에
모든 패키지에 적용되지 않는다.
`Manager::Instance()` 싱글톤은 테스트에서 교체할 수 없다 —
인터페이스 주입(Seam)으로 잘라야 한다.

## Pattern

```cpp
// ── 인터페이스 정의 (이미 있다면 재사용) ─────────────────────────────
// IAcqAssistHandler.h
class IAcqAssistHandler
{
public:
    virtual ~IAcqAssistHandler() = default;
    virtual void OnStateChanged(EAcqState eState) = 0;
    virtual bool IsReady() const = 0;
};

// ── Seam: 싱글톤 대신 참조 주입 ──────────────────────────────────────
// 수정 전 (테스트 불가)
class CESMain
{
    void Update() { CAcqAssistManager::Instance().OnStateChanged(m_eState); }
};

// 수정 후 (테스트 가능)
class CESMain
{
public:
    explicit CESMain(IAcqAssistHandler& handler) : m_handler(handler) {}
    void Update() { m_handler.OnStateChanged(m_eState); }
private:
    IAcqAssistHandler& m_handler;  // 주입된 의존성
    EAcqState          m_eState = EAcqState::Idle;
};

// ── 수동 Test Double ──────────────────────────────────────────────────
// TestDoubleAcqAssistHandler.h  (테스트 코드 전용, 프로덕션 링크 X)
struct CallRecorder
{
    int             onStateChangedCount = 0;
    EAcqState       lastState           = EAcqState::Idle;
    bool            isReadyReturnValue  = true;
};

class CFakeAcqAssistHandler : public IAcqAssistHandler
{
public:
    explicit CFakeAcqAssistHandler(CallRecorder& rec) : m_rec(rec) {}

    void OnStateChanged(EAcqState eState) override
    {
        m_rec.onStateChangedCount++;
        m_rec.lastState = eState;
    }

    bool IsReady() const override { return m_rec.isReadyReturnValue; }

private:
    CallRecorder& m_rec;
};

// ── GoogleTest 픽스처에서 사용 ────────────────────────────────────────
// ESMainTest.cpp
#include "gtest/gtest.h"
#include "CESMain.h"
#include "TestDoubleAcqAssistHandler.h"

class ESMainTest : public ::testing::Test
{
protected:
    CallRecorder            m_rec;
    CFakeAcqAssistHandler   m_fakeHandler{m_rec};
    CESMain                 m_esMain{m_fakeHandler};
};

TEST_F(ESMainTest, Update_WhenStateActive_CallsHandlerOnce)
{
    m_esMain.SetState(EAcqState::Active);
    m_esMain.Update();

    EXPECT_EQ(1, m_rec.onStateChangedCount);
    EXPECT_EQ(EAcqState::Active, m_rec.lastState);
}

TEST_F(ESMainTest, Update_WhenNotReady_DoesNotCallHandler)
{
    m_rec.isReadyReturnValue = false;
    m_esMain.Update();

    EXPECT_EQ(0, m_rec.onStateChangedCount);
}
```

**Seam 위치 찾기 원칙:**

```cpp
// Seam = 코드 변경 없이 동작을 교체할 수 있는 지점
// gipc-app에서 흔한 Seam 후보:
//   - Manager::GetInstance() 반환값에 의존하는 메서드 파라미터
//   - OVObjectHandle을 생성하는 팩토리 함수
//   - ScLog 출력 — 테스트에서는 NullLogger로 교체

// Seam이 없으면: 생성자나 Init() 파라미터로 의존성을 밖으로 꺼낸다
// Manager::Instance() 를 건드리지 않고, 그것을 *사용하는* 클래스만 수정
```

## Key Points

- GMock 없이도 `CallRecorder` struct + `virtual` 상속으로 호출 횟수·인자를 검증할 수 있다
- `Manager::Instance()` 싱글톤은 목(mock)이 불가능 — **Seam을 만들어 인터페이스를 주입**하라
- Test Double은 프로덕션 링크 대상에서 제외해야 한다 (`_test.cpp` 파일 또는 별도 테스트 라이브러리)
- `CallRecorder`를 픽스처가 소유하게 하면 여러 Fake가 같은 레코더를 공유할 수 있다
- 인터페이스가 이미 존재하면 새로 만들지 말고 재사용 — YAGNI
