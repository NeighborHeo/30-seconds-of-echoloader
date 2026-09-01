---
title: "OVObject = UDT 노드의 구현체"
category: udt
tags: [udt, ovobject, gc-framework, factory, rendering]
difficulty: advanced
---

`OVObject`는 GrandCentral UDT 노드로 등록 가능한 렌더링 객체의 기반 클래스다. 팩토리 태그가 UDT 타입 ID가 된다.

## OVObject ↔ UDT 노드 매핑

```cpp
// OVObject.cpp — 팩토리 등록
// 태그 문자열 = UDT 타입 ID
REGISTER_OV_OBJECT("Gc.SWEAcqAssist",  SWEAcquisitionAssistant);   // line 222
REGISTER_OV_OBJECT("Gc.UGAPAcqAssist", UGAPAcquisitionAssistant);  // line 225
// 나머지 ~30개 OVObject 등록

// 런타임 생성
// ObjectViewerImpl.cpp에서:
auto handle = CreateUdtObject("Gc.SWEAcqAssist");
// → SWEAcquisitionAssistant 인스턴스화
// → GcUdtHandle<SWEAcquisitionAssistant> 반환
```

## OVObject 필수 인터페이스 (UDT 연동)

```cpp
class SWEAcquisitionAssistant : public OVObject
{
public:
    // ── UDT 프레임워크 필수 오버라이드 ──────────────────────────

    // GC 파라미터 수신 (Layer A → Layer B 채널)
    void SetParameter(const std::string& key,
                      const Variant& value) override;

    // 상태 변경 명령 수신
    void ChangeState(const std::string& stateName) override;

    // 렌더링 루프 진입점
    void Render(OVGraphics::Context& ctx) override;

    // 활성화/비활성화 (모드 전환 시 프레임워크 호출)
    void OnActivate() override;
    void OnDeactivate() override;

    // ── GcUdtHandle에서 접근 가능한 public API ──────────────────

    void ProcessFrame(const Frame& frame);
    SWEROIProcessor::ProcessResult GetLastResult() const;

private:
    // UDT 노드가 보유하는 상태 (프레임워크 수명 = 이 객체 수명)
    Frame                         m_currentFrame_BS;
    SWEROIProcessor               m_roiProcessor;
    LiverWarning3Panel            m_warningPanel;    // 추출 중인 컴포지션
    GcUdtHandle<LiverAIProcessor> m_liverAiHandle;  // 다른 UDT 참조
};
```

## dynamic_cast와 UDT 타입 확인

```cpp
// ObjectViewerImpl.cpp:2063, 2098 — 외부 컨트랙트
// UDT 핸들을 구체 타입으로 좁힐 때 dynamic_cast 사용
OVObject* pObj = /* 팩토리에서 얻은 OVObject */;

// SWE 전용 처리
auto* pSWE = dynamic_cast<SWEAcquisitionAssistant*>(pObj);
if (pSWE != nullptr)
    pSWE->SetSWESpecificParam(value);

// UGAP 전용 처리
auto* pUGAP = dynamic_cast<UGAPAcquisitionAssistant*>(pObj);
if (pUGAP != nullptr)
    pUGAP->SetUGAPSpecificParam(value);

// 이 두 줄은 외부 컨트랙트 — 클래스 이름 변경 또는 상속 구조 변경 시
// ObjectViewerImpl.cpp 의 이 위치도 함께 변경해야 함
```

## Key Points

- `REGISTER_OV_OBJECT("Gc.X", ClassName)` — 이 한 줄이 UDT 타입 등록의 전부
- 팩토리 태그("Gc.SWEAcqAssist")는 **외부 컨트랙트** — ESMain, ObjectViewerImpl이 이 문자열을 하드코딩
- `OnActivate()` / `OnDeactivate()` 는 모드 전환 시 호출 — 여기서 보유 핸들 정리 필수
- `dynamic_cast<SWEAcquisitionAssistant*>` 캐스트 위치(ObjectViewerImpl:2063/2098)는 절대 건드리지 않을 4개 지점 중 하나
- `OVObject::SetParameter()`가 GC 파라미터 버스의 수신 끝점 — Layer A와 Layer B를 잇는 유일한 C++ 진입점
