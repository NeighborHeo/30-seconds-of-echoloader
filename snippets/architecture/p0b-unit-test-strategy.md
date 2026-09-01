---
title: "P0b: Warning Unit Test Strategy"
category: architecture
tags: [testing, gtest, warning, p0b, refactoring, safety]
difficulty: advanced
---

Warning authority flip(P2+P4)에 진입하기 전 반드시 Warning 플래그 3개에 대한 단위 테스트를 작성해야 한다. 현재 커버리지 0%.

## 현황

```
현재 테스트 커버리지:
  TestSWEAlgorithm/  → ROI 알고리즘 전용 ✅
  TestUGAPAlgorithm/ → ROI 알고리즘 전용 ✅
  Warning UI 로직    → 0% ❌ ← P0b 목표

Warning 플래그 3개:
  m_bLargeSCDWarning         — AcquisitionAssistantBase.cpp:130 write, :538-540 read
  m_bPoorProbeContactWarning — AcquisitionAssistantBase.cpp:135 write, :594-599 read
  m_bObliqueCapsuleWarning   — AcquisitionAssistantBase.cpp:140 write
```

## 목표 테스트 구조

```cpp
// 파일: src/GcViewer/ObjectViewer/UnitTest/LiverWarning3PanelFixture.cpp

class LiverWarning3PanelFixture : public testing::Test
{
protected:
    void SetUp() override
    {
        m_panel = std::make_unique<LiverWarning3Panel>();
        m_panel->Init("SWEPreset");  // P-INIT 완료 후 파라미터화
    }
    std::unique_ptr<LiverWarning3Panel> m_panel;
};

TEST_F(LiverWarning3PanelFixture,
    "Require_that_LargeSCD_warning_triggers_when_distance_exceeds_threshold")
{
    m_panel->UpdateSkinCapsuleDistance(65.0f);  // mm, 임계값 초과
    EXPECT_TRUE(m_panel->IsLargeSCDWarning());
}

TEST_F(LiverWarning3PanelFixture,
    "Require_that_all_warnings_clear_on_reset")
{
    m_panel->UpdateSkinCapsuleDistance(65.0f);
    m_panel->Reset();
    EXPECT_FALSE(m_panel->IsLargeSCDWarning());
    EXPECT_FALSE(m_panel->IsPoorProbeContactWarning());
    EXPECT_FALSE(m_panel->IsObliqueCapsuleWarning());
}

TEST_F(LiverWarning3PanelFixture,
    "Require_that_GuidelinesGray_reflects_any_active_warning")
{
    m_panel->UpdateSkinCapsuleDistance(65.0f);
    EXPECT_TRUE(m_panel->ShouldShowGuidelinesGray());
}
```

## P0b 체크리스트

```
[ ] LargeSCD 경고 트리거 조건 확인
[ ] LargeSCD 경고 해제 조건 확인
[ ] PoorProbeContact 경고 트리거/해제
[ ] ObliqueCapsule 경고 트리거/해제
[ ] 복합 경고 시 GuidelinesGray = true
[ ] 모든 경고 없을 때 GuidelinesGray = false
[ ] Init(presetName) 미호출 시 기본값 안전 확인  ← P-INIT spike 결과 반영
```

## Key Points

- **P0b 없이 P2+P4(authority flip)에 진입하면 안 됨** — 회귀를 잡을 방법이 없음
- 예상 소요: 2~3일 (GoogleTest `*Fixture` + 7개 `TEST_F` 작성)
- 테스트가 실행 가능하려면 `LiverWarning3Panel::Init()` 이 먼저 동작해야 함 → P-INIT spike 선행
- `ShouldShowGuidelinesGray` 고아 오버라이드의 호출자 확인(P1-spike)도 P0b 전에 처리
- 테스트 파일 위치: `src/GcViewer/ObjectViewer/UnitTest/`
