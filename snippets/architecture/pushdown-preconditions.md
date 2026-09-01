---
title: "Push-Down Pre-Conditions: When to Safely Move Code Down"
category: architecture
tags: [push-down, refactoring, warning, dependency, strategy]
difficulty: advanced
---

Base → SWE/UGAP 푸시다운은 모든 코드에 동시에 적용할 수 없다. Warning과의 결합도에 따라 세 그룹으로 분류된다.

## 분류 기준

```
즉시 푸시다운 가능 (Warning 미참조):
  D5-stub  — 사용자 입력 핸들러 7개 (SelectDn/TrbMove 등)
             총 ~21줄, Warning 플래그 미참조
             → 지금 당장 SWE/UGAP에 복제 가능

P2+P4 후 푸시다운 (Warning과 결합):
  D1 Progress UI — m_progressIndicator (STEP1=Warning 조건)
  D2 Geometry    — m_leftTiltAngle 등 (PoorProbe에서 직접 읽음)
  D3 Lifecycle   — DebugParams (Warning 필드 혼재)
  D5 OVObject I/F — SetParameter, ChangeState (Warning 키 라우팅)

변경 불요 (이미 서브클래스 구현):
  ROI 가상 메서드 — 이미 SWE/UGAP 각자 구현, Base가 비워 있음
```

## D5-stub: 즉시 가능한 푸시다운

```cpp
// Base 에 있는 trivial stub 핸들러들 (경고 미참조)
// 7개 모두 이 형태:
void AcquisitionAssistantBase::HandleSelectDn(const Event& ev)
{
    // SWE, UGAP 서브클래스에서 재정의 예정 — 현재 stub
}

// 푸시다운 후:
// Base에서 제거 → SWE/UGAP에 각자 필요한 구현 추가
void SWEAcquisitionAssistant::HandleSelectDn(const Event& ev)
{
    // SWE 전용 SelectDn 로직
}

void UGAPAcquisitionAssistant::HandleSelectDn(const Event& ev)
{
    // UGAP 전용 SelectDn 로직 (다를 수 있음)
}
```

## 푸시다운 안전성 검증

```cpp
// 푸시다운 전 체크리스트:
// [ ] grep: Base의 해당 멤버/메서드가 Warning 플래그를 읽거나 쓰는가?
// [ ] grep: 해당 멤버가 Base 외부에서 사용되는가? (0건이어야 함)
// [ ] 전후 동작이 동일한가? (임상 녹화본 재생으로 검증)
```

## Key Points

- D5-stub 7개는 **지금 바로 가능** — P2+P4를 기다릴 필요 없음
- D4(enum 추출)도 즉시 가능 — `AcqAssistState` → `AcqAssistTypes.h` 이동
- D1/D2/D5-non-stub는 **Warning Base authority가 Panel로 이전된 후에만** 안전
- "Warning과 독립"을 grep으로 증명하지 않고 푸시다운하면 숨은 결합이 런타임 버그로 나타남
- 푸시다운 PR도 관련 티켓 ID(FBUG/RBUG) 인용 의무
