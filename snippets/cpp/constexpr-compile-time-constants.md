---
title: "constexpr로 GC 파라미터 키·매직 넘버 관리"
category: cpp
tags: [constexpr, gc-param, constants, c++11, maintenance]
difficulty: beginner
---

GC 파라미터 키 문자열과 수치 임계값을 `constexpr`로 한 곳에 모아 오타와 중복을 제거하는 패턴.

## Why

`"SWEAcqAssist.State"` 같은 키 문자열이 송신 `.cpp`와 수신 `.cpp`에 각각 리터럴로 흩어지면
하나를 바꿀 때 다른 하나를 빠뜨린다. 컴파일러는 이 불일치를 잡지 못한다.

## Pattern

```cpp
// ── GC 파라미터 키 상수 ───────────────────────────────────────────────
// GcParamKeys.h (헤더 하나, 모든 패키지가 공유)
#pragma once

namespace GcKeys
{
    // SWE 관련 키
    constexpr const char* kSWEState        = "SWEAcqAssist.State";
    constexpr const char* kSWEGain         = "SWEAcqAssist.Gain";
    constexpr const char* kSWEFrameRate    = "SWEAcqAssist.FrameRate";

    // Render 관련 키
    constexpr const char* kRenderMode      = "RenderPipeline.Mode";
    constexpr const char* kRenderQuality   = "RenderPipeline.Quality";
}

// ── 수치 임계값 ───────────────────────────────────────────────────────
// RenderBudget.h
namespace RenderBudget
{
    constexpr int   kFrameBudgetMs         = 33;    // 30fps 기준 프레임 예산
    constexpr int   kWarningThresholdMs    = 28;    // 경고 발생 기준
    constexpr float kScdWarningThreshold   = 0.85f; // SCD 경고 임계값 (85%)
    constexpr int   kMaxPendingFrames      = 3;     // 렌더 큐 최대 대기
}

// ── 송신 측 사용 ──────────────────────────────────────────────────────
// CSWEAcqAssist.cpp
#include "GcParamKeys.h"

void CSWEAcqAssist::SetState(EAcqState eState)
{
    m_pGcBus->SetParameterValue(GcKeys::kSWEState, static_cast<int>(eState));
}

// ── 수신 측 사용 ──────────────────────────────────────────────────────
// CSWERenderer.cpp
#include "GcParamKeys.h"

void CSWERenderer::OnParameterChanged(const char* pKey, const CGcValue& val)
{
    if (strcmp(pKey, GcKeys::kSWEState) == 0)
    {
        m_eState = static_cast<EAcqState>(val.GetInt());
    }
}

// ── C++11 constexpr 제약 사항 ─────────────────────────────────────────
// C++11에서 constexpr 함수는 단일 return 문만 허용
// (C++14 이상은 여러 문장 가능 — gipc-app C++14 빌드에서는 OK)

// C++11 OK — 단일 return
constexpr int FramesToMs(int nFrames) { return nFrames * 33; }

// C++11 NG — 지역 변수·if 문 불가 (C++14 에서는 OK)
// constexpr int Clamp(int v, int lo, int hi)
// {
//     int result = v < lo ? lo : v;   // C++11 에서 컴파일 오류
//     return result > hi ? hi : result;
// }

// C++11 대안: 삼항 연산자로 단일 표현식
constexpr int Clamp11(int v, int lo, int hi)
{
    return v < lo ? lo : (v > hi ? hi : v);
}

// ── constexpr vs #define 비교 ────────────────────────────────────────
// #define kSWEState "SWEAcqAssist.State"  ← 타입 없음, 디버거에 안 보임, 스코프 없음
// constexpr const char* kSWEState = ...   ← 타입 있음, 네임스페이스 스코프, 권장
```

## Key Points

- GC 키 상수는 `namespace GcKeys` 한 헤더에 모으고 송신·수신 양쪽이 같은 헤더를 `#include`한다
- `constexpr const char*` 는 `#define` 대비 타입 안전, 네임스페이스 스코프, 디버거 가시성 제공
- C++11 `constexpr` 함수는 **단일 return 식** 제약 — 복잡한 로직은 삼항 연산자로 표현하거나 C++14 빌드를 확인
- 수치 임계값(프레임 예산, 경고 비율)도 `constexpr`로 선언하면 매직 넘버 리뷰 지적을 원천 차단
- 키 이름 변경 시 컴파일러가 미사용 상수 경고를 내면 — 사용처를 함께 교체했다는 신호
