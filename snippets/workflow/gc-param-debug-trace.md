---
title: "GC 파라미터 흐름 추적 — 기능이 반응하지 않을 때"
category: workflow
tags: [debugging, gc-param, sclog, tracing, swe]
difficulty: intermediate
---

Layer A에서 상태를 변경했는데 Layer B의 OVObject가 반응하지 않을 때 원인을 찾는 체크리스트.

## Why

GC(Global Configuration) 파라미터 버스는 문자열 키로 값을 전달한다.
키 오타, 대소문자 불일치, OVObject 비활성 상태 — 세 가지 중 하나가 거의 전부다.

## Pattern

```cpp
// ── 증상 ──────────────────────────────────────────────────────────────
// SWEAcqAssist::SetState(EState::Active) 를 Layer A에서 호출했는데
// Layer B의 CSWERenderer 는 여전히 이전 상태를 유지함

// ── 체크 1: SetParameterValue가 실제로 호출됐는지 확인 ─────────────
// grep으로 송신 측 코드 찾기
// $ grep -rn "SWEAcqAssist.State" src/

// 로그 추가 (임시, PR 전 제거)
void CSWEAcqAssist::SetState(EState eState)
{
    SCLOG_MODULE(SCLOG_DEBUG, "SWEAcqAssist", "SetState called: %d", (int)eState);
    m_pGcBus->SetParameterValue("SWEAcqAssist.State", (int)eState);
}

// ── 체크 2: 키 이름 정확히 일치하는지 확인 ───────────────────────────
// GC 버스는 대소문자 구분. 아래는 모두 다른 키다:
//   "SWEAcqAssist.State"   ← 올바른 키
//   "SWEAcqAssist.Stat"    ← 오타: 'e' 누락
//   "sweacqassist.state"   ← 소문자 → 매칭 안 됨
//   "SWEAcqAssist.state"   ← 's' 소문자 → 매칭 안 됨

// constexpr 키 상수로 오타 방지 (forward ref → constexpr-compile-time-constants.md)
namespace GcKeys {
    constexpr const char* kSWEState = "SWEAcqAssist.State";
}
// 송신 측:
m_pGcBus->SetParameterValue(GcKeys::kSWEState, (int)eState);
// 수신 측:
void CSWERenderer::OnParameterChanged(const char* pKey, const CGcValue& val)
{
    if (strcmp(pKey, GcKeys::kSWEState) == 0) { ... }
}

// ── 체크 3: 수신 OVObject가 Active 상태인지 확인 ─────────────────────
// OVObject가 Inactive 상태면 파라미터 콜백이 오지 않음
COVObjectHandle<CSWERenderer> hRenderer = GetRenderer();
if (!hRenderer.IsValid())
{
    SCLOG_MODULE(SCLOG_WARNING, "SWEAcqAssist", "Renderer handle invalid — param not delivered");
    return;
}

// ── 체크 4: 수신 측에 임시 로그 추가 ─────────────────────────────────
void CSWERenderer::SetParameter(const char* pKey, const CGcValue& val)
{
    // 디버그 전용 — 실 배포 전 제거
    SCLOG_MODULE(SCLOG_DEBUG, "SWERenderer", "SetParameter key=%s", pKey);

    if (strcmp(pKey, GcKeys::kSWEState) == 0)
    {
        // ...
    }
}

// ── 체크 5: 등록 시점 확인 ───────────────────────────────────────────
// RegisterForParameter 가 OVObject Activate 이후에 호출됐는지 확인
// Activate 전에 등록하면 콜백이 등록되지 않음
void CSWERenderer::Activate()
{
    COVObject::Activate();  // 반드시 먼저
    m_pGcBus->RegisterForParameter(GcKeys::kSWEState, this);  // 그 다음 등록
}
```

**디버그 빌드에서만 로그를 남기는 패턴:**

```cpp
// 프로덕션 영향 없이 임시 로그 추가
#ifdef _DEBUG
    SCLOG_MODULE(SCLOG_DEBUG, "SWEAcqAssist", "GC param sent: key=%s val=%d",
                 GcKeys::kSWEState, (int)eState);
#endif
```

## Key Points

- GC 버스 키는 **대소문자 구분** — 오타가 가장 흔한 원인이므로 `constexpr` 키 상수로 통일하라
- OVObject가 `IsValid()` 아닐 때 파라미터 콜백은 묵묵히 무시된다 — 로그조차 없음
- `RegisterForParameter`는 반드시 `COVObject::Activate()` **이후**에 호출해야 한다
- 임시 `SCLOG_MODULE` 추가 시 `_DEBUG` 조건부 또는 PR 설명에 "디버그 로그 제거 예정" 명시
- 송신 측과 수신 측 키 문자열이 서로 다른 파일에 리터럴로 흩어져 있으면 리팩터링 후보
