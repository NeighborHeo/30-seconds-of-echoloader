---
title: "GoogleTest Fixture 작성 규칙"
category: workflow
tags: [googletest, unit-test, TDD, MFC]
difficulty: intermediate
---

`TEST_F` + `Require_that_` 네이밍, 파일은 `UnitTest/` 하위에 배치한다.

## 절차

1. **파일 위치**: `src/packages/EchoXxx/UnitTest/TestMyFeature.cpp`
2. **Fixture 클래스명**: `MyFeatureFixture : public testing::Test`
3. **테스트명 형식**: `Require_that_<메서드>_<조건>_<기대결과>`
4. **GE CONFIDENTIAL 헤더** 파일 최상단에 필수
5. `EXPECT_NO_THROW` 남발 금지 — 실제 반환값 검증 우선
6. MFC 대화상자 기반 인터랙티브 테스트(`CTestGraph` 패턴)는 별도 UnitTestApp에서 관리

## 예시

```cpp
/* -GE CONFIDENTIAL- Type: Source Code ... */
#include "gtest/gtest.h"
#include "MyFeature.h"

class MyFeatureFixture : public testing::Test
{
protected:
    MyFeature m_sut;
};

TEST_F(MyFeatureFixture, Require_that_compute_returns_zero_when_input_is_empty)
{
    const auto result = m_sut.Compute({});
    EXPECT_EQ(0, result);
}

TEST_F(MyFeatureFixture, Require_that_compute_returns_sum_when_given_valid_input)
{
    const auto result = m_sut.Compute({1, 2, 3});
    EXPECT_EQ(6, result);
}
```

## 체크리스트

- [ ] 파일 경로: `EchoXxx/UnitTest/TestMyFeature.cpp`
- [ ] GE CONFIDENTIAL 헤더 최상단
- [ ] Fixture: `class XxxFixture : public testing::Test`
- [ ] 테스트명: `Require_that_<method>_<condition>_<expected>`
- [ ] `EXPECT_NO_THROW` 대신 반환값 직접 검증
- [ ] `CTestGraph` 인터랙티브 테스트와 혼재 금지
