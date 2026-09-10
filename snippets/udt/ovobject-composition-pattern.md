---
title: "OVObject 컴포지션 패턴: 다중 상속 대신 has-a"
category: udt
tags: [ovobject, composition, inheritance, helper-class, design-pattern]
difficulty: advanced
---

OVObject는 다중 상속이 불가능하다. GC 팩토리는 `OVObject*` 하나만 기대한다. 두 OVObject 사이의 공유 로직은 별도 헬퍼 클래스로 추출해 합성한다.

## Why

팩토리 등록은 단일 베이스 클래스(`OVObject`)를 전제로 동작한다. 공통 동작을 두 번째 베이스 클래스로 끌어올리면 팩토리가 타입을 인식하지 못하거나, 다이아몬드 상속 문제가 생긴다. 헬퍼 클래스 합성이 유일하게 안전한 재사용 경로다.

## Pattern

```cpp
// ── 잘못된 패턴: 두 OVObject가 공통 베이스를 공유하려는 시도 ─────────────────
class AcquisitionAssistantBase : public OVObject   // ❌
{
    // 공통 경고 로직을 base에 넣고 싶은 충동
    void EvaluateWarnings(const Frame& frame);
};

class SWEAcquisitionAssistant  : public AcquisitionAssistantBase {}; // ❌
class UGAPAcquisitionAssistant : public AcquisitionAssistantBase {}; // ❌
// → 팩토리는 AcquisitionAssistantBase*를 반환하고
//   GcUdtHandle<SWEAcquisitionAssistant>로 downcasting이 깨짐

// ── 올바른 패턴: 헬퍼 클래스를 멤버로 합성 ──────────────────────────────────
// 헬퍼는 OVObject를 상속하지 않음 — 순수 C++ 클래스
class WarningLogic
{
public:
    struct Result
    {
        bool isSCDLarge;
        float scdScore;
        bool isFibrosis;
    };

    // OVObject와 무관하게 독립적으로 테스트 가능
    Result Evaluate(const Frame& frame, float roiX, float roiY) const;

    void SetThreshold(float threshold) { m_threshold = threshold; }

private:
    float m_threshold;
    // NULL 로 초기화 (C++11 legacy 코드베이스)
};

// SWEAcquisitionAssistant: 헬퍼를 has-a로 보유
class SWEAcquisitionAssistant : public OVObject      // ✅ 단일 상속
{
public:
    void Render(OVGraphics::Context& ctx) override
    {
        // 헬퍼에 위임 — OVObject 인터페이스와 분리된 도메인 로직
        WarningLogic::Result result =
            m_warningLogic.Evaluate(m_currentFrame_BS, m_roiX, m_roiY);

        // 결과를 파라미터 버스로 발행
        SetParameterValue("LiverWarning.SCD.Large",
            static_cast<int>(result.isSCDLarge ? 1 : 0));
    }

private:
    WarningLogic m_warningLogic;    // ✅ has-a
    Frame        m_currentFrame_BS;
    float        m_roiX;
    float        m_roiY;
};

// UGAPAcquisitionAssistant: 동일한 헬퍼를 독립적으로 합성
class UGAPAcquisitionAssistant : public OVObject     // ✅ 단일 상속
{
public:
    void Render(OVGraphics::Context& ctx) override
    {
        // 같은 WarningLogic, 다른 컨텍스트
        WarningLogic::Result result =
            m_warningLogic.Evaluate(m_currentFrame_BS, m_ugapRoiX, m_ugapRoiY);
        // ...
    }

private:
    WarningLogic m_warningLogic;    // ✅ 별도 인스턴스, 공유 없음
    Frame        m_currentFrame_BS;
    float        m_ugapRoiX;
    float        m_ugapRoiY;
};
```

## Key Points

- `REGISTER_OV_OBJECT("Gc.X", ClassName)` 은 `OVObject` 단일 상속만 보장 — 다중 상속 시 팩토리 동작 미정의
- **공유 로직은 헬퍼 클래스로 추출** — `OVObject`를 상속하지 않는 순수 C++ 클래스
- 헬퍼 인스턴스는 OVObject가 멤버로 소유 — OVObject 수명과 동일하게 관리됨
- 헬퍼 클래스는 `OVObject` 없이 단독 테스트 가능 → 도메인 로직을 프레임워크 의존에서 분리
- OVObject 간 기능 중복이 느껴질 때마다 먼저 헬퍼 추출 가능 여부 확인 — 상속 계층 추가는 최후 수단
