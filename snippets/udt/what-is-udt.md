---
title: "UDT란 무엇인가: GrandCentral의 객체 노드"
category: udt
tags: [udt, grandcentral, gc-framework, object-model, architecture]
difficulty: intermediate
---

gipc-app의 **UDT(User-Defined Type)**는 GrandCentral(Gc) 프레임워크가 관리하는 typed 객체 노드다. `GcUdtHandle`은 이 노드로 향하는 관리형 참조다.

## 개념 계층

```
GrandCentral (Gc) 프레임워크
│
├── GC Parameter Bus        ← 문자열 키/값 메시지 (stateless)
│   └── SetParameterValue("SWEAcqAssist.RoiX", 42.0f)
│
└── UDT Object Graph        ← 프레임워크가 소유하는 typed 객체 트리 (stateful)
    ├── UDT Node (Gc.SWEAcqAssist)   ← SWEAcquisitionAssistant
    ├── UDT Node (Gc.UGAPAcqAssist)  ← UGAPAcquisitionAssistant
    └── UDT Node (...)               ← ~30개 OVObject 서브클래스

                GcUdtHandle<T>
                     │
                     └──▶ UDT Node  (프레임워크 소유)
                          (C++ 객체 + 타입 메타데이터 + 팩토리 태그)
```

## UDT 노드의 구성

```cpp
// 팩토리 태그가 UDT 타입을 식별 (OVObject.cpp에 등록)
// "Gc.SWEAcqAssist"  → SWEAcquisitionAssistant 인스턴스화
// "Gc.UGAPAcqAssist" → UGAPAcquisitionAssistant 인스턴스화

// UDT 노드는 세 가지를 포함:
// 1. C++ 클래스 인스턴스 (SWEAcquisitionAssistant 등)
// 2. 타입 식별자 ("Gc.SWEAcqAssist")
// 3. GC 파라미터 수신 인터페이스 (SetParameter / ChangeState)

// 노드 생성: 팩토리 태그로 요청 → 프레임워크가 인스턴스화
GcUdtHandle<SWEAcquisitionAssistant> handle =
    GcFramework::CreateObject("Gc.SWEAcqAssist");
//                             ^^^^^^^^^^^^^^^^
//                             OVObject.cpp:222에 등록된 팩토리 태그
```

## GC 파라미터 vs UDT 객체

```
GC 파라미터 버스:                UDT 객체:
  키-값 문자열 메시지               상태를 가진 C++ 객체
  stateless                        stateful (멤버 변수 보유)
  any → any 브로드캐스트            handle 보유자 → 특정 노드
  Layer A → Layer B 단방향 주         양방향 (handle로 직접 호출 가능)
  GC param 이름으로 라우팅           C++ 타입으로 직접 접근

  SetParameterValue("key", val)    handle->DoWork()
                                   handle->GetState()
```

## Key Points

- "UDT"의 "U"는 User-Defined (GE 프레임워크 위에서 정의한 타입)
- "Gc"는 GrandCentral — GE의 내부 객체 관리/메시징 프레임워크
- UDT 노드의 수명은 프레임워크가 관리 — `new`/`delete` 직접 호출 금지
- 팩토리 태그("Gc.SWEAcqAssist")가 타입 시스템의 핵심 — 오타 = 인스턴스화 실패 (묵음)
- GC 파라미터 버스와 UDT 객체는 **협력하지, 대체하지 않는다** — 레이어 간 통신은 파라미터, 레이어 내 직접 호출은 핸들
