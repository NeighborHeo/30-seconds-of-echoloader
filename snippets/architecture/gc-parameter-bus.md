---
title: "GC Parameter Bus: The Layer A↔B Bridge"
category: architecture
tags: [gc-params, layer-boundary, messaging, decoupling, architecture]
difficulty: intermediate
---

레이어 A(EchoScanner)와 레이어 B(GcViewer) 사이의 유일한 통신 수단은 GC 파라미터 버스다. 문자열 키/값 쌍으로 교환하며, 직접 C++ 함수 호출은 없다.

## Structure

```cpp
// Layer A (EchoScanner) — 파라미터 쓰기
ESMain::SetParameterValue("SWEAcqAssist.RoiX", roiX);
ESMain::SetParameterValue("SWEAcqAssist.RoiY", roiY);
ESMain::SetParameterValue("SWEAcqAssist.State",
    static_cast<int>(AcqAssistState::Freeze));

// Layer B (GcViewer) — 파라미터 읽기
// OVObject 표준 인터페이스 통해 수신
void SWEAcquisitionAssistant::SetParameter(
    const std::string& key, const Variant& value) override
{
    if (key == "SWEAcqAssist.RoiX")
        m_roiX = value.AsFloat();
    else if (key == "SWEAcqAssist.State")
        m_currentState = static_cast<AcqAssistState>(value.AsInt());
}

// Layer B → Layer A 역방향 (AutoPos 키)
// ResolveAutoPosKey() 가 반환한 문자열을 ESMain이 수신
std::string key = m_handler->ResolveAutoPosKey(currentState);
if (!key.empty())
    ESMain::ProcessAutoPos(key);   // "SWEAcqAssistToggleContinuous" 등
```

## 버스 토폴로지

```
ESMain.cpp (Layer A)
    │
    │  SetParameterValue("SWEAcqAssist.*", value)
    ▼
GC Parameter Bus (문자열 키/변형 값)
    │
    ▼
OVObject::SetParameter() (Layer B)
    ├── SWEAcquisitionAssistant::SetParameter()
    └── UGAPAcquisitionAssistant::SetParameter()

역방향 (AutoPos):
SWEAcqAssistHandler::ResolveAutoPosKey() ──▶ ESMain::ProcessAutoPos()
```

## Key Points

- 버스는 **단방향 아님** — 파라미터(A→B)와 AutoPos 키(B→A) 두 방향 모두 존재
- 문자열 키 오타는 컴파일 타임에 잡히지 않음 → 묵음 실패 → 키 상수화 권장
- `SetParameter` / `ChangeState` 등의 OVObject 인터페이스는 **외부 컨트랙트** — 시그니처 변경 불가
- 레이어 분리 덕분에 레이어 B(렌더링)는 레이어 A(획득 로직) 변경 없이 독립 진화 가능
- UGAP preset 이름(`"ShearTrackAlg"`)도 이 버스를 타고 흐름 — Director 확인 없이 변경 금지
