---
title: "ESMain 내부 구조: 11개 호출 사이트와 게이팅 로직"
category: architecture
tags: [esmain, acquisition, gating, entry-point, layer-a, swe, ugap]
difficulty: advanced
---

`ESMain.cpp`는 Layer A의 중심축이다. ~9,954줄 중 AcquisitionAssistant 진입점 11개가 7066~9954 라인에 집중 분포한다.

## 11개 호출 사이트 구조

```cpp
// ESMain.cpp 의 AA 호출 패턴 — 모드 게이팅 후 Manager 위임
// (7066~9954 라인 범위, 개략적 표현)

// ── SWE 게이팅 ──────────────────────────────────────────────────
// 사이트 1: 프레임 수신 시
if (ESMain::InElasto())                          // SWE 모드 확인
    Manager::Instance(HandlerType::SWE)
        ->OnNewFrame(currentFrame);

// 사이트 2: 사용자 입력 (키/버튼)
if (ESMain::InElasto())
    Manager::Instance(HandlerType::SWE)
        ->HandleUserInput(inputEvent);

// 사이트 3: 상태 전환
if (ESMain::InElasto())
    Manager::Instance(HandlerType::SWE)
        ->UpdateAcqAssistState(newState);

// ── UGAP 게이팅 ─────────────────────────────────────────────────
// 사이트 4~7: 동일 구조, 다른 모드 술어
if (ESMain::IsInUGAPMode())
    Manager::Instance(HandlerType::UGAP)
        ->OnNewFrame(currentFrame);
// ... (사이트 8~11도 동일 패턴)
```

## 두 게이팅 술어

```cpp
// SWE 게이팅
bool ESMain::InElasto()
{
    // Elastography(탄성 초음파) 모드 활성화 여부
    // m_scanMode == ScanMode::Elastography 등
    return /* SWE 모드 판단 로직 */;
}

// UGAP 게이팅
bool ESMain::IsInUGAPMode()
{
    // UGAP 전용 모드 활성화 여부
    // SWE와 UGAP는 동시 활성화 불가 (배타적)
    return /* UGAP 모드 판단 로직 */;
}

// 두 술어는 절대 동시에 true가 되면 안 됨
// assert(!(InElasto() && IsInUGAPMode()));  ← 실제로는 없음 (assert 회피 관행)
```

## Manager::Instance() 호출 규칙

```cpp
// Manager는 싱글톤 — HandlerType으로 구체 핸들러 선택
namespace AcquisitionAssistant {
    Manager& Manager::Instance(HandlerType type)
    {
        // HandlerType::SWE → SWEAcqAssistHandler
        // HandlerType::UGAP → UGAPAcqAssistHandler
        static Manager s_instance;  // 싱글톤
        return s_instance;
    }
}

// 외부 컨트랙트: ESMain.cpp의 이 패턴은 변경 불가
// Manager::Instance(HandlerType::SWE)  — 문자열 아님, 강타입 enum
// Manager::Instance(HandlerType::UGAP) — 이 시그니처가 외부 API
```

## ESMain의 GC 파라미터 발행

```cpp
// ESMain이 Layer B에 상태 전달하는 시점
void ESMain::OnAcqAssistStateChanged(AcqAssistState newState)
{
    // SWE Layer A → Layer B (GC 파라미터)
    SetParameterValue("SWEAcqAssist.State",
        static_cast<int>(newState));

    // AutoPos 키 (Layer B → Layer A 역방향 응답)
    std::string autoPosKey =
        Manager::Instance(HandlerType::SWE)
            ->ResolveAutoPosKey(newState);
    if (!autoPosKey.empty())
        ProcessAutoPos(autoPosKey);
}
```

## Key Points

- 11개 사이트 전부 **게이팅 후 위임** 구조 — ESMain이 직접 AA 로직을 갖지 않음
- `InElasto()` 와 `IsInUGAPMode()` 는 배타적 — 동시 true = 상태 버그
- `Manager::Instance(HandlerType::SWE/UGAP)` — 이 시그니처가 외부 컨트랙트 3번째 항목
- 11개 사이트 중 하나라도 게이팅을 누락하면 잘못된 모드에서 AA 동작 → 임상 결과 오염
- ESMain.cpp는 9,954줄 — 파일 전체를 읽지 말고, AA 관련 사이트만 7066~9954 범위에서 grep
