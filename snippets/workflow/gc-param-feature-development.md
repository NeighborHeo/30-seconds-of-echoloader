---
title: "GC 파라미터 기반 기능 개발: 처음부터 끝까지"
category: workflow
tags: [gc-param, ovobject, layer-a, layer-b, atomic, setparameter]
difficulty: beginner
---

Layer A에 새 상태/데이터가 생기고 Layer B가 화면에 표시해야 할 때 사용하는 표준 개발 절차.

## 언제 이 아티클을 보나

- ESMain 또는 Manager/Handler에서 계산한 값을 OVObject(GcViewer 쪽)에 전달해야 할 때
- 기존 GC 파라미터 키가 없는 새 데이터 타입을 처음 추가할 때
- "SetParameter가 왜 안 불리지?" 디버깅 시

## Step 1: 파라미터 키 설계 (5분)

키 이름 규칙: `"PackageName.Feature.SubFeature"` 형식.

```cpp
// src/EchoScanner/AcquisitionAssistant/GcParamKeys.h
namespace GcParamKeys {
    // 기존 예시
    constexpr const char* SWEState  = "SWEAcqAssist.State";
    // 신규 추가
    constexpr const char* SWEResultMean = "SWEAcqAssist.Result.Mean";
}
```

발행측과 수신측 모두 이 헤더를 `#include` 한다. 문자열 리터럴을 직접 쓰지 않는다.

## Step 2: Layer A — 파라미터 발행 (ESMain 또는 Manager)

Manager 또는 Handler가 계산을 완료한 시점에 GC 버스로 발행한다.

```cpp
// src/EchoScanner/AcquisitionAssistant/SWEAcqAssistHandler.cpp
#include "GcParamKeys.h"

void SWEAcqAssistHandler::OnNewFrame(const Frame& frame)
{
    float mean = CalculateMean(frame);
    SetParameterValue(GcParamKeys::SWEResultMean, static_cast<float>(mean));
}
```

`SetParameterValue`는 프레임워크가 라우팅한다. ESMain.cpp를 수정하지 않는다.

## Step 3: Layer B — OVObject에서 수신

```cpp
// src/GcViewer/AcquisitionAssistant/SWEAcquisitionAssistant.h
#include "GcParamKeys.h"
#include <atomic>

class SWEAcquisitionAssistant : public OVObject
{
    std::atomic<float> m_mean{0.0f};  // SetParameter 스레드와 Render 스레드가 공유

    void SetParameter(const std::string& key, const Variant& value) override
    {
        if (key == GcParamKeys::SWEResultMean)
        {
            if (value.GetType() != Variant::Float)
            {
                ScLogWarn("SWEAA", "SWEResultMean: 예상 Float, 실제 %d", value.GetType());
                return;
            }
            m_mean.store(value.GetFloat());
        }
    }
};
```

## Step 4: Layer B — Render()에서 사용

```cpp
void SWEAcquisitionAssistant::Render()
{
    const float mean = m_mean.load();  // atomic 읽기
    if (mean > 0.0f)
        DrawMeanValue(mean);
}
```

`m_mean`을 `load()` 없이 직접 읽으면 torn read가 발생한다. 항상 `load()`를 쓴다.

## Step 5: 검증 — 파라미터 흐름 확인

```cpp
// 임시 디버그 로그. 확인 후 반드시 제거.
#ifdef _DEBUG
void SWEAcquisitionAssistant::SetParameter(const std::string& key, const Variant& value)
{
    ScLogInfo("SWEAA_DBG", "SetParameter: key=%s", key.c_str());
    // ... 위 Step 3 로직 ...
}
#endif
```

발행측에도 같은 방식으로 로그를 추가하고, 두 로그를 비교해 흐름을 확인한다.

## Step 6: 흔한 실수와 진단

```
증상: OVObject::SetParameter()가 호출되지 않음
진단:
  [ ] 키가 발행측/수신측에서 완전히 동일한지 확인 (대소문자 포함)
  [ ] OVObject가 Active 상태인지 (IsValid() 반환값 확인)
  [ ] SetParameterValue가 실제로 호출되는지 (발행측에 ScLog 추가)

증상: Render()에서 m_mean 값이 0이거나 찢긴 값
진단:
  [ ] m_mean이 std::atomic<float>인지 확인 (일반 float이면 torn read 가능)
  [ ] SetParameter()와 Render()가 항상 다른 스레드에서 실행됨을 전제

증상: 빌드 에러 — GcParamKeys 심볼 미정의
진단:
  [ ] GcParamKeys.h를 발행측 .cpp에 #include했는지
  [ ] GcParamKeys.h를 수신측 .h 또는 .cpp에 #include했는지
```

## 전체 플로우 요약

```
[발행측 .cpp] SetParameterValue(GcParamKeys::SWEResultMean, value)
     │
     ▼ GC 파라미터 버스 (프레임워크가 라우팅)
     │
[수신측 .cpp] OVObject::SetParameter(key, value)
                  → m_mean.store(value.GetFloat())
     │
     ▼ 다음 Render() 틱
     │
[수신측 .cpp] Render() → m_mean.load() → DrawMeanValue()
```

## Key Points

- 파라미터 키를 `constexpr` 상수로 선언하는 것이 개발 시작점 — 문자열 리터럴 직접 사용 금지
- `SetParameter()`와 `Render()`는 다른 스레드에서 실행됨 → `std::atomic` 필수
- 새 파라미터 추가 시 `ESMain.cpp` 수정 불필요 — Manager/Handler에서 직접 발행
- 수신측 OVObject가 Active 상태가 아니면 `SetParameter()` 자체가 호출되지 않음
- 디버깅: 발행 로그 + 수신 로그를 동시에 켜고 비교하는 것이 가장 빠른 방법
