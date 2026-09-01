---
title: "GcUdtHandle 생명주기: 생성·유효성·해제"
category: udt
tags: [GcUdtHandle, lifetime, RAII, validity, framework]
difficulty: advanced
---

`GcUdtHandle`은 프레임워크가 소유한 UDT 노드로의 참조다. 핸들이 살아 있어도 노드는 프레임워크에 의해 독립적으로 삭제될 수 있다.

## 생명주기 흐름

```
[생성]                         [사용]                    [해제]
   │                              │                          │
   ▼                              ▼                          ▼
GcFramework::CreateObject()   handle.IsValid()?         handle.Release()
   │                          ├── true → 사용 가능      또는 스코프 종료
   │                          └── false → 노드 소멸됨   (RAII 소멸자)
   ▼
GcUdtHandle<T> handle        ← 프레임워크가 소유,
                                핸들은 참조만 보유
```

## 핵심 패턴

```cpp
class SWEAcquisitionAssistant : public OVObject
{
public:
    // 1. 초기화: 팩토리를 통해 핸들 획득
    bool Init(const std::string& presetName, std::string& errorMsg)
    {
        m_roiHandle = GetFramework()->AcquireUdt<SWEROIProcessor>(
            "Gc.SWEROIProcessor");
        if (!m_roiHandle.IsValid())
        {
            errorMsg = "SWEROIProcessor UDT 획득 실패";
            return false;
        }
        return true;
    }

    // 2. 사용: 항상 IsValid() 선행
    void Update()
    {
        if (!m_roiHandle.IsValid())      // ← 생략 금지
        {
            ScLogError("SWEAA", "ROI handle invalid — UDT may have been destroyed");
            return;
        }
        m_roiHandle->ProcessFrame(m_currentFrame_BS);
    }

    // 3. 해제: RAII 또는 명시적 Release()
    void Shutdown()
    {
        m_roiHandle.Release();           // 프레임워크에 참조 반환
        // m_roiHandle.IsValid() == false 이후
    }

private:
    GcUdtHandle<SWEROIProcessor> m_roiHandle;
    // 소멸자에서 자동 Release() — RAII
};
```

## 무효 핸들이 발생하는 시점

```cpp
// 시나리오 1: 모드 전환 — UGAP → SWE 전환 시 UGAP UDT 소멸
GcUdtHandle<UGAPROIProcessor> m_ugapHandle;
// 모드 전환 후:
// m_ugapHandle.IsValid() → false
// m_ugapHandle->Process() → 크래시 (무효 역참조)

// 시나리오 2: 프레임워크 셧다운 순서
// EchoLoader 종료 시 UDT 그래프가 ObjectViewer보다 먼저 해제될 수 있음
// → Shutdown()에서 IsValid() 재확인 필요

// 시나리오 3: OVObject::OnDeactivate() 콜백
void SWEAcquisitionAssistant::OnDeactivate() override
{
    // 비활성화 시 보유 핸들 정리
    m_roiHandle.Release();
    m_liverAiHandle.Release();
}
```

## Key Points

- `GcUdtHandle`은 **비소유(non-owning)** 참조다 — 노드 소유자는 항상 GC 프레임워크
- `IsValid()` 없는 역참조는 프레임워크 내부 크래시 → 의료기기에서 치명적
- 핸들이 유효해도 내부 노드 상태가 일관성 없을 수 있음 — 도메인 레벨 검증 별도 필요
- `GcUdtHandle` 자체를 `std::shared_ptr` 에 넣는 패턴 사용 금지 — 이중 소유권 혼란
- 모드 전환 경로(`SWE ↔ UGAP`)는 핸들 무효화 hotspot — 해당 코드 변경 시 반드시 검토
