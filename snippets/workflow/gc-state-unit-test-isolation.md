---
title: "GrandCentral 상태 의존 코드의 단위 테스트 격리"
category: workflow
tags: [googletest, testing, gc, grandcentral, dependency-injection, seam, GcUdtRegistry]
difficulty: intermediate
---

`GcUdtRegistry::Get<T>()` 또는 `GcUdtHandle`에 의존하는 함수를 GC 초기화 없이 단위 테스트하는 방법.

## 언제 이 아티클을 보나

테스트 환경에서 GC가 초기화되지 않아 `GcUdtRegistry::Get<T>()`가 항상 Invalid 핸들을 반환할 때.
함수의 핵심 로직(계산, 상태 전이)은 테스트하고 싶지만 GC 없이는 그 경로에 진입할 수 없을 때.

## 문제

```cpp
// 테스트하고 싶은 함수
float SWEAcquisitionAssistant::CalculateWarningScore()
{
    auto handle = GcUdtRegistry::Get<LiverAIProcessor>("Gc.LiverAI");
    if (!handle.IsValid())
        return 0.0f;  // ← 테스트 환경에서 항상 이 경로만 실행됨
    return handle->GetAiScore() * m_weight;
}
```

GC 없는 환경에서는 `IsValid()` 가 항상 `false` → 곱셈 로직을 테스트할 수 없다.

## Solution 1: IsValid() 경로 자체를 테스트 (최소 침습)

GC 없을 때의 안전한 동작을 테스트한다. 코드 변경 없음.

```cpp
// GC가 없으면 0.0f 반환 — 이 경로 자체를 테스트
// ESMain 등 호출자가 "GC 없을 때 점수를 어떻게 처리하는가"를 검증

// SWEAcquisitionAssistantTest.cpp
#include "gtest/gtest.h"
#include "SWEAcquisitionAssistant.h"

class SWEAcquisitionAssistantTest : public ::testing::Test
{
protected:
    SWEAcquisitionAssistant m_assistant;
};

TEST_F(SWEAcquisitionAssistantTest,
    Require_that_CalculateWarningScore_ReturnsZero_WhenGcNotInitialized)
{
    // GC 없음 → IsValid() == false → 0.0f
    EXPECT_FLOAT_EQ(0.0f, m_assistant.CalculateWarningScore());
}
```

이 패턴의 의미: "GC 없을 때 크래시 없이 안전하게 동작하는가"를 테스트한다.
AI 점수 곱셈 로직은 Solution 2 또는 3으로 별도 테스트.

## Solution 2: 의존성 주입으로 Seam 만들기 (중간)

```cpp
// IAiScoreProvider.h
class IAiScoreProvider
{
public:
    virtual ~IAiScoreProvider() {}
    virtual float GetScore() const = 0;
};

// DefaultAiScoreProvider.h — 프로덕션: GC 사용
class DefaultAiScoreProvider : public IAiScoreProvider
{
public:
    float GetScore() const override
    {
        auto handle = GcUdtRegistry::Get<LiverAIProcessor>("Gc.LiverAI");
        return handle.IsValid() ? handle->GetAiScore() : 0.0f;
    }
};

// SWEAcquisitionAssistant.h — 주입 가능한 생성자 추가
class SWEAcquisitionAssistant
{
public:
    // 프로덕션 경로: 기본 인자로 DefaultAiScoreProvider 사용
    explicit SWEAcquisitionAssistant(IAiScoreProvider* pProvider = NULL)
        : m_pAiProvider(pProvider != NULL ? pProvider : &m_defaultProvider)
    {}

    float CalculateWarningScore()
    {
        return m_pAiProvider->GetScore() * m_weight;
    }

private:
    DefaultAiScoreProvider m_defaultProvider;  // GC 사용 (프로덕션)
    IAiScoreProvider*      m_pAiProvider;      // 테스트에서 교체
    float                  m_weight = 1.0f;
};

// ── 테스트 Double ─────────────────────────────────────────────────────
// FakeAiScoreProvider.h  (테스트 코드 전용)
class CFakeAiScoreProvider : public IAiScoreProvider
{
public:
    explicit CFakeAiScoreProvider(float score) : m_score(score) {}
    float GetScore() const override { return m_score; }
private:
    float m_score;
};

// ── GoogleTest 픽스처 ─────────────────────────────────────────────────
class SWEAcquisitionAssistantDITest : public ::testing::Test
{
protected:
    CFakeAiScoreProvider    m_fakeProvider{0.85f};
    SWEAcquisitionAssistant m_assistant{&m_fakeProvider};
};

TEST_F(SWEAcquisitionAssistantDITest,
    Require_that_CalculateWarningScore_MultipliesScoreByWeight)
{
    // 0.85f * m_weight(1.0f) = 0.85f
    EXPECT_NEAR(0.85f, m_assistant.CalculateWarningScore(), 0.001f);
}
```

## Solution 3: 레거시 코드 — Subclass and Override (최후 수단)

기존 클래스를 거의 바꾸지 않아야 할 때. GC 호출 부분만 `protected virtual`로 추출한다.

```cpp
// SWEAcquisitionAssistant.h — protected virtual 한 줄만 추가
class SWEAcquisitionAssistant
{
public:
    float CalculateWarningScore()
    {
        return GetAiScore() * m_weight;  // 이 호출만 바꿈
    }

protected:
    virtual float GetAiScore()  // ← 신규 추가
    {
        auto h = GcUdtRegistry::Get<LiverAIProcessor>("Gc.LiverAI");
        return h.IsValid() ? h->GetAiScore() : 0.0f;
    }

private:
    float m_weight = 1.0f;
};

// ── 테스트용 서브클래스 ───────────────────────────────────────────────
// TestableSWEAcquisitionAssistant.h  (테스트 코드 전용)
class CTestableSWEAcquisitionAssistant : public SWEAcquisitionAssistant
{
protected:
    float GetAiScore() override { return 0.85f; }  // GC 우회
};

// ── 테스트 ───────────────────────────────────────────────────────────
TEST(SWEAcquisitionAssistantSubclassTest,
    Require_that_CalculateWarningScore_UsesInjectedScore)
{
    CTestableSWEAcquisitionAssistant assistant;
    EXPECT_NEAR(0.85f, assistant.CalculateWarningScore(), 0.001f);
}
```

## 어느 패턴을 쓸지

| 상황 | 권장 패턴 |
|------|-----------|
| 신규 코드, 처음부터 작성 | Solution 2 (DI) |
| 레거시 코드, 최소 변경 | Solution 3 (Subclass and Override) |
| 테스트 목표가 "GC 없을 때 안전한가" | Solution 1 (코드 변경 없음) |
| GC 자체를 초기화할 수 있는 통합 테스트 | GC 초기화 후 실제 `GcUdtRegistry::Get<T>()` 사용 |

## Key Points

- GC가 없으면 `GcUdtRegistry::Get<T>()`는 항상 Invalid 핸들을 반환한다 — 이 동작 자체가 테스트 가능한 명세다 (Solution 1)
- Solution 2의 `IAiScoreProvider* pProvider = NULL` 기본 인자는 프로덕션 호출자가 변경 없이 동작하게 한다
- Solution 3의 `protected virtual`은 최소한의 Seam — 인터페이스 도입 없이 GC 호출 경로만 끊는다
- 테스트 Double 클래스(`CFakeAiScoreProvider`, `CTestableSWEAcquisitionAssistant`)는 프로덕션 링크 대상에 포함하지 않는다
- GoogleTest 테스트명은 `Require_that_<함수>_<기대동작>_<조건>` 형식을 유지한다 (gipc-app 컨벤션)
