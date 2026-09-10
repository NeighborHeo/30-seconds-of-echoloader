---
title: "GC 파라미터 디스패치: SetParameterValue에서 OVObject까지"
category: udt
tags: [gc-params, dispatch, ovobject, routing, parameter-bus]
difficulty: intermediate
---

`ESMain::SetParameterValue("SWEAcqAssist.State", 2)`가 호출되면, GC 버스는 prefix 매칭으로 OVObject를 찾아 `SetParameter()`를 호출한다. 수신자가 없으면 묵음으로 버려진다.

## Why

Layer A(EchoScanner)는 Layer B(ObjectViewer)의 구체 클래스를 알 수 없다. 문자열 키 기반 파라미터 버스가 유일한 레이어 간 통신 경로다. 발신자는 수신자 존재 여부를 확인하지 않으며, 반환값도 없다.

## Pattern

```cpp
// ── Layer A (ESMain / EchoScanner) ──────────────────────────────────────────
// SetParameterValue는 fire-and-forget — 반환값 없음, ACK 없음
ESMain::SetParameterValue("SWEAcqAssist.State",
    static_cast<int>(AcqAssistState::Active));
//                   ^^^^^^^^^^^^^^^
//                   Variant(int) 로 boxing됨

// ── GC 버스 내부 (ObjectViewerImpl) ─────────────────────────────────────────
// ObjectViewerImpl::RouteParameter() 개념적 구현
void ObjectViewerImpl::RouteParameter(
    const std::string& key, const Variant& value)
{
    // prefix = 첫 번째 '.' 이전 문자열
    // "SWEAcqAssist.State" → prefix = "SWEAcqAssist"
    std::string prefix = key.substr(0, key.find('.'));

    // 등록된 OVObject 목록에서 prefix 일치 객체 탐색
    // "Gc.SWEAcqAssist" 태그로 등록된 SWEAcquisitionAssistant 발견
    OVObject* pTarget = FindRegisteredObject(prefix);  // "SWEAcqAssist"

    if (pTarget == NULL)
    {
        // 수신자 없음 → 묵음 드롭, 에러 없음
        // 오타("SWEAcqAssst.State")도 여기서 조용히 사라짐
        return;
    }

    // 전체 키와 값을 OVObject에 전달
    pTarget->SetParameter(key, value);
}

// ── Layer B OVObject 수신 끝점 ───────────────────────────────────────────────
void SWEAcquisitionAssistant::SetParameter(
    const std::string& key, const Variant& value)
{
    // "SWEAcqAssist.State"
    if (key == "SWEAcqAssist.State")
    {
        // Variant → int 언박싱 (타입 불일치 시 GetInt() 내부에서 경고 로그)
        int state = value.GetInt();
        m_currentState = static_cast<AcqAssistState>(state);
        return;
    }

    if (key == "SWEAcqAssist.RoiX")
    {
        m_roiX = value.GetFloat();
        return;
    }

    // 알 수 없는 키: 묵음 무시 (assert 금지 — 렌더 스레드)
}

// ── prefix 매칭 규칙 ─────────────────────────────────────────────────────────
// "SWEAcqAssist.*"  → Gc.SWEAcqAssist  (SWEAcquisitionAssistant)
// "UGAPAcqAssist.*" → Gc.UGAPAcqAssist (UGAPAcquisitionAssistant)
// "LiverWarning.*"  → Gc.LiverWarning  (LiverWarning3Panel)
// 겹치는 prefix 없음 — 매핑은 1:1
```

## Key Points

- 디스패치 체인: `SetParameterValue` → `RouteParameter()` → prefix 매칭 → `OVObject::SetParameter()` — 총 3단계
- prefix는 팩토리 태그("Gc.SWEAcqAssist")의 "Gc." 뒤 부분과 일치해야 라우팅됨
- **수신자가 없으면 에러 없이 묵음 드롭** — 오타는 런타임에 잡히지 않으므로 키 이름을 상수로 관리할 것
- fire-and-forget 설계: 발신자가 렌더 스레드 지연을 유발하지 않도록 반환값·ACK 없음
- `SetParameter()` 내부에서 예외 throw 금지 — 렌더 스레드에서 호출되며 의료기기 안전 요구사항
