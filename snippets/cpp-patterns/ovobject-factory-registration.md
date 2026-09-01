---
title: "OVObject 팩토리 태그 등록 패턴"
category: cpp-patterns
tags: [factory, OVObject, registration, dynamic_cast]
difficulty: advanced
---

OVObject에 새 타입을 추가할 때는 태그 등록 → dynamic_cast → ESMain 연결의 3단계가 atomic하게 이루어져야 한다.

## Why

gipc-app의 OVObject 시스템은 문자열 태그로 객체 타입을 식별한다. 팩토리가 태그로 인스턴스를 생성하고, 뷰어가 `dynamic_cast`로 타입을 확인한다. 태그는 외부 컨트랙트(설정 파일, 직렬화 포맷)와 연결되므로 변경하면 하위 호환이 깨진다.

## Pattern

```cpp
// OVObject.cpp:222, 225 — 팩토리 태그 등록
// 반드시 이 파일의 등록 블록 안에 추가
REGISTER_OV_OBJECT("Gc.SWEAcqAssist",  SWEAcquisitionAssistant);
REGISTER_OV_OBJECT("Gc.UGAPAcqAssist", UGAPAcquisitionAssistant);

// 새 타입 추가 예시 (3단계 atomic)
// Step 1: OVObject.cpp — 태그 등록
REGISTER_OV_OBJECT("Gc.NewFeature", NewFeatureObject);

// Step 2: ObjectViewerImpl.cpp:2063, 2098 — dynamic_cast로 타입 확인
// ObjectViewerImpl.cpp
void ObjectViewerImpl::ProcessObject(OVObject* pObj) {
    // SWE 처리
    if (auto* pSwe = dynamic_cast<SWEAcquisitionAssistant*>(pObj)) {
        pSwe->HandleSweSpecific();
        return;
    }
    // UGAP 처리
    if (auto* pUgap = dynamic_cast<UGAPAcquisitionAssistant*>(pObj)) {
        pUgap->HandleUgapSpecific();
        return;
    }
    // Step 2 신규 추가
    if (auto* pNew = dynamic_cast<NewFeatureObject*>(pObj)) {
        pNew->HandleNewFeature();
        return;
    }
}

// Step 3: ESMain.cpp — 인스턴스 생성 및 연결
// ESMain.cpp
void ESMain::Initialize() {
    // 기존 패턴
    m_sweAssist  = OVObjectFactory::Create("Gc.SWEAcqAssist");
    m_ugapAssist = OVObjectFactory::Create("Gc.UGAPAcqAssist");
    // Step 3 신규 추가
    m_newFeature = OVObjectFactory::Create("Gc.NewFeature");
}
```

## Key Points

- 팩토리 태그(`"Gc.SWEAcqAssist"` 등)는 외부 컨트랙트 — **절대 변경 금지**. 변경하면 저장된 설정/직렬화 데이터가 로드 불가
- `dynamic_cast` 실패(`nullptr` 반환)는 조용히 무시하지 말 것 — 등록 누락 버그의 징후
- 3단계(태그 등록 + `dynamic_cast` + ESMain 연결)가 atomic: 하나라도 빠지면 런타임 오류
- 태그 문자열은 `"Gc."` 네임스페이스 접두사 유지
- `REGISTER_OV_OBJECT` 매크로 위치(`OVObject.cpp` 등록 블록)를 벗어나지 않는다 — 정적 초기화 순서 의존
- 기존 태그 목록은 `OVObject.cpp:222` 부근에서 확인
