---
title: "Variant 타입 디스패치: GC 파라미터의 타입 안전 처리"
category: udt
tags: [variant, gc-params, type-safety, tagged-union, render-thread]
difficulty: advanced
---

GC 파라미터 버스의 값은 `Variant`로 boxing된다. `std::any`(C++17)는 사용 불가 — tagged union 기반의 `Variant`가 int/float/bool/string을 운반한다. 타입 불일치는 예외 없이 경고 로그로 처리한다.

## Why

Layer A(정적 바이너리)와 Layer B(플러그인 렌더러)는 독립적으로 배포된다. 컴파일 타임 타입 연결이 없으므로 런타임 tagged union이 유일한 선택이다. 렌더 스레드에서 예외는 금지이므로 타입 불일치는 묵음 실패로 처리한다.

## Pattern

```cpp
// ── 발신자 (Layer A) — 타입 명시적 boxing ────────────────────────────────────
// int: 상태 enum을 int로 변환 후 전달
ESMain::SetParameterValue("SWEAcqAssist.State",
    static_cast<int>(AcqAssistState::Active));
//  ^^^^^^^^^^^^^^^ Variant(int) 생성

// float: 좌표, 점수 등
ESMain::SetParameterValue("SWEAcqAssist.RoiX",
    static_cast<float>(roiX));

// bool 없음 — int 0/1로 대체 (legacy 규칙)
ESMain::SetParameterValue("LiverWarning.SCD.Large",
    static_cast<int>(isSCDLarge ? 1 : 0));

// string: 프리셋 이름 등
ESMain::SetParameterValue("SWEAcqAssist.PresetName",
    std::string("LiverStandard"));

// ── 수신자 (Layer B) — GetType() switch로 안전 추출 ─────────────────────────
void SWEAcquisitionAssistant::SetParameter(
    const std::string& key, const Variant& value)
{
    if (key == "SWEAcqAssist.State")
    {
        // 타입 확인 먼저
        if (value.GetType() != Variant::INT)
        {
            // 렌더 스레드: throw 금지, 로그만
            ScLogWarning("SWEAA",
                "SWEAcqAssist.State: expected INT, got %d", value.GetType());
            return;  // 묵음 실패 — 이전 상태 유지
        }

        int state = value.GetInt();
        m_currentState = static_cast<AcqAssistState>(state);
        return;
    }

    if (key == "SWEAcqAssist.RoiX")
    {
        if (value.GetType() != Variant::FLOAT)
        {
            ScLogWarning("SWEAA", "SWEAcqAssist.RoiX: expected FLOAT");
            return;
        }
        m_roiX = value.GetFloat();
        return;
    }

    if (key == "SWEAcqAssist.PresetName")
    {
        if (value.GetType() != Variant::STRING)
        {
            ScLogWarning("SWEAA", "SWEAcqAssist.PresetName: expected STRING");
            return;
        }
        m_presetName = value.GetString();
        return;
    }
}

// ── Variant 타입 열거 (개념적 정의) ─────────────────────────────────────────
// class Variant {
// public:
//     enum Type { INT, FLOAT, STRING, BOOL_AS_INT };
//     Type        GetType()   const;
//     int         GetInt()    const;  // Type != INT 이면 경고 후 0 반환
//     float       GetFloat()  const;  // Type != FLOAT 이면 경고 후 0.0f 반환
//     std::string GetString() const;  // Type != STRING 이면 경고 후 "" 반환
// };
// ponytail: GetType() 없이 GetInt()만 호출하면 경고 로그 발생 — 항상 타입 확인 선행
```

## Key Points

- `Variant`는 C++17 `std::any` 없이 int/float/string을 tagged union으로 운반 — MSVC C++14 제약
- **항상 `GetType()` 먼저** — `GetInt()` 직접 호출은 타입 불일치 시 경고 로그 + 기본값 반환(묵음 실패)
- 렌더 스레드에서 예외 throw 금지 — 타입 불일치는 `ScLogWarning` + `return`으로 처리
- bool은 int 0/1로 전달 — `Variant::BOOL` 타입이 없는 codebase가 존재하므로 수신 측에서 `GetInt() != 0` 패턴 사용
- 타입 안전성 없음은 **의도된 trade-off** — Layer A와 Layer B가 독립 배포되어야 하므로 컴파일 타임 연결 불가
