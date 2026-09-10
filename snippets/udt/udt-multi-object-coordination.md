---
title: "OVObject 간 협력: 직접 참조 없이 조율하기"
category: udt
tags: [ovobject, coordination, gc-params, decoupling, lifecycle]
difficulty: intermediate
---

두 OVObject는 서로의 헤더를 `#include`할 수 없다. GC 파라미터 버스가 유일한 합법적 조율 채널이다.

## Why

OVObject는 각기 다른 수명을 가진다. `OnDeactivate()` 후 포인터를 들고 있으면 dangling이다. 버스를 통하면 수신자 존재 여부와 무관하게 안전하게 발행할 수 있다.

## Pattern

```cpp
// ── 잘못된 패턴: 직접 포인터 보유 ───────────────────────────────────────────
class SWEAcquisitionAssistant : public OVObject
{
    // ❌ LiverWarning3Panel의 헤더를 include하면 순환 의존 발생
    // ❌ 포인터로 들고 있으면 상대방 OnDeactivate() 후 dangling
    LiverWarning3Panel* m_pWarningPanel;  // ❌ 절대 금지
};

// ── 올바른 패턴: GC 파라미터 버스로 발행 ────────────────────────────────────
// Object A: SWEAcquisitionAssistant (발행자)
void SWEAcquisitionAssistant::Render(OVGraphics::Context& ctx)
{
    bool isSCDLarge = EvaluateSCDSize(m_currentFrame_BS);

    // 파라미터 버스로 발행 — LiverWarning3Panel의 존재를 몰라도 됨
    // 수신자가 비활성 상태여도 묵음 드롭 → 크래시 없음
    SetParameterValue("LiverWarning.SCD.Large",
        static_cast<int>(isSCDLarge ? 1 : 0));

    SetParameterValue("LiverWarning.SCD.Score",
        m_scdScore);  // float
}

// Object B: LiverWarning3Panel (수신자)
// SWEAcquisitionAssistant를 include하지 않음, 포인터도 없음
void LiverWarning3Panel::SetParameter(
    const std::string& key, const Variant& value)
{
    if (key == "LiverWarning.SCD.Large")
    {
        m_isSCDLarge = (value.GetInt() != 0);
        return;
    }

    if (key == "LiverWarning.SCD.Score")
    {
        m_scdScore = value.GetFloat();
        return;
    }
}

// ── 수명 차이가 안전한 이유 ──────────────────────────────────────────────────
// 시나리오: SWE 모드 → UGAP 모드 전환
//
// 1. LiverWarning3Panel::OnDeactivate() 호출 → 패널 비활성화
// 2. SWEAcquisitionAssistant::Render()가 아직 실행 중
//    → SetParameterValue("LiverWarning.SCD.Large", ...) 발행
//    → RouteParameter()가 비활성 수신자 못 찾음 → 묵음 드롭
//    → 크래시 없음 ✅
//
// 직접 포인터였다면:
//    → m_pWarningPanel->SetSCDLarge(...)
//    → OnDeactivate() 이후 dangling dereference → 크래시 ❌
```

## Key Points

- OVObject끼리 `#include` 금지 — 팩토리/프레임워크가 인스턴스를 소유하므로 헤더 노출은 계층 위반
- **발행자는 수신자의 수명을 신경 쓰지 않아도 된다** — 버스가 묵음 드롭으로 처리
- 파라미터 키 네임스페이스 prefix("LiverWarning.")가 수신자를 특정 — prefix 충돌 시 의도치 않은 OVObject가 수신
- 직접 포인터 보유는 `OnDeactivate()` 이후 dangling → 의료기기 안전 기준상 허용 불가
- 고빈도 데이터(프레임당 픽셀 버퍼)는 파라미터 버스 부적합 — 그 경우 공유 메모리나 콜백 채널 별도 설계 필요
