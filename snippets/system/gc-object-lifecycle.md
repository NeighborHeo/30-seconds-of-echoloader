---
title: "GrandCentral 객체 수명주기 다이어그램"
category: system
tags: [gc-framework, udt, lifecycle, ovobject, activation]
difficulty: advanced
---

OVObject가 GrandCentral에서 생성되고 소멸되는 전체 흐름. 이걸 모르면 핸들 사용 시 댕글링 포인터를 만든다.

## 수명주기 상태 다이어그램

```
                    [앱 시작]
                        │
                        ▼
              ┌─────────────────┐
              │  팩토리 등록     │  ← OVObject.cpp: "Gc.SWEAcqAssist" 태그 등록
              │  (정적 초기화)   │     ObjectViewerImpl이 태그→타입 매핑 보유
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  Inactive       │  ← 객체 인스턴스 없음
              │  (모드 비활성)   │     GcUdtHandle<T>.IsValid() == false
              └────────┬────────┘
                       │ 모드 진입 (SWE ON / UGAP ON)
                       ▼
              ┌─────────────────┐
              │  OnActivate()   │  ← 프레임워크 호출
              │                 │     내부 상태 초기화, 리소스 획득
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  Active         │  ← Render() 루프 실행 중
              │                 │     SetParameter() 수신 중
              │  ┌───────────┐  │     GcUdtHandle<T>.IsValid() == true
              │  │ Render()  │◀─┼──── 렌더링 루프 (프레임마다)
              │  └───────────┘  │
              │  ┌───────────┐  │
              │  │SetParam() │◀─┼──── GC 파라미터 버스 수신
              │  └───────────┘  │
              └────────┬────────┘
                       │ 모드 종료 (SWE OFF / 프로브 분리)
                       ▼
              ┌─────────────────┐
              │  OnDeactivate() │  ← 프레임워크 호출
              │                 │     리소스 해제, 핸들 Release()
              └────────┬────────┘
                       │
                       ▼
              ┌─────────────────┐
              │  Inactive       │  ← 다시 비활성 상태
              └─────────────────┘
```

## 핸들 접근 안전 패턴

```cpp
// ✅ 올바른 핸들 사용 패턴
void SomeOVObject::Render()
{
    // 매번 IsValid() 확인 — OnDeactivate 후에 Invalid
    auto handle = GcUdtRegistry::Get<SWEAcquisitionAssistant>("Gc.SWEAcqAssist");
    if (!handle.IsValid())
        return;  // 조용히 건너뜀 — 크래시 대신

    handle->DoSomething();
    // handle 소멸 → 자동 참조 해제 (RAII)
}

// ❌ 잘못된 패턴: 핸들을 멤버로 캐시
class BadOVObject : public OVObject
{
    GcUdtHandle<SWEAcquisitionAssistant> m_cachedHandle;  // ❌

    void OnActivate() override
    {
        m_cachedHandle = GcUdtRegistry::Get<SWEAcquisitionAssistant>("Gc.SWEAcqAssist");
        // 다른 객체가 OnDeactivate 되면 m_cachedHandle이 댕글링
    }
};
```

## 프레임워크 vs 개발자 책임 경계

```
프레임워크가 보장:               개발자가 해야 할 것:
  UDT 노드 생성/소멸              IsValid() 확인 후 접근
  OnActivate/OnDeactivate 호출   OnDeactivate에서 핸들 Release()
  렌더링 루프 스케줄링            Render()에서 무거운 연산 금지
  파라미터 라우팅 스레드 안전      내부 공유 상태에 Mutex 사용
  팩토리 등록 관리               "Gc." 접두사 오타 방지
```

## Key Points

- `OnActivate` → `Active` → `OnDeactivate` 는 프레임워크가 호출 — 개발자가 직접 부르면 안 됨
- `GcUdtHandle<T>`는 비소유 참조 — `Release()` 안 해도 소멸자에서 자동 해제
- `IsValid()` 검사를 빠뜨리면 모드 전환 순간에 접근 위반(AV) 발생 → 임상 중 크래시
- `Render()`는 프레임마다 호출 → 할당/파싱 등 무거운 작업은 `OnActivate`에서
- 핸들을 멤버로 캐시하지 말 것 — 타 객체의 `OnDeactivate`로 무효화될 수 있음
