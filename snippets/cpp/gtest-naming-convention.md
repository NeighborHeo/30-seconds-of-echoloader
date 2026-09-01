---
title: "GoogleTest Naming Convention"
category: cpp
tags: [testing, gtest, conventions, naming]
difficulty: beginner
---

gipc-app의 GoogleTest 패턴: `*Fixture` + `TEST_F(Fixture, "Require_that_...")`. 테스트 이름이 요구사항 문장처럼 읽혀야 한다.

## Pattern

```cpp
// 파일 위치: src/packages/EchoScanner/AcquisitionAssistant/UnitTest/
// #include <gtest/gtest.h>

class AcqAssistHandlerFixture : public testing::Test
{
protected:
    void SetUp() override
    {
        // 초기화
        m_handler = std::make_unique<SWEAcqAssistHandler>();
    }

    void TearDown() override
    {
        m_handler.reset();
    }

    std::unique_ptr<SWEAcqAssistHandler> m_handler;
};

// 테스트 이름: "Require_that_<subject>_<predicate>"
TEST_F(AcqAssistHandlerFixture, "Require_that_AutoPos_returns_empty_when_frozen")
{
    m_handler->SetState(AcqAssistState::Freeze);
    EXPECT_EQ(m_handler->ResolveAutoPosKey(), "");
}

TEST_F(AcqAssistHandlerFixture, "Require_that_state_transitions_to_Continuous_on_toggle")
{
    m_handler->SetState(AcqAssistState::Continuous);
    EXPECT_EQ(m_handler->GetState(), AcqAssistState::Continuous);
    EXPECT_NO_THROW(m_handler->ToggleContinuous());
}
```

## 디렉토리 구조

```
src/packages/EchoScanner/
├── AcquisitionAssistant/
│   ├── AcquisitionAssistant.h
│   ├── AcquisitionAssistant.cpp
│   └── UnitTest/
│       ├── AcqAssistHandlerFixture.h
│       └── AcqAssistHandlerTest.cpp   ← Require_that_* 테스트들
├── TestSWEAlgorithm/   ← ROI 알고리즘 전용 (UI warning 커버리지 없음)
└── TestUGAPAlgorithm/
```

## Key Points

- 클래스명은 반드시 `*Fixture` 접미어 + `: public testing::Test` 상속
- 테스트명은 `"Require_that_..."` 형식 — 문서처럼 읽혀야 함 (주석 불필요)
- `EXPECT_NO_THROW` 는 사용 관찰됨; 일반 예외 흐름은 없으므로 드물게 등장
- 현재 Warning UI(`m_bLargeSCDWarning` 등) 테스트 커버리지 **0%** — P0b 작업에서 채워야 할 영역
- `TestSWEAlgorithm/`, `TestUGAPAlgorithm/` 은 ROI 알고리즘 전용; UI/Warning 로직은 미커버
