---
title: "GrandCentral(Gc) 프레임워크 전체 그림"
category: architecture
tags: [grandcentral, gc-framework, udt, parameter-bus, ovobject, architecture]
difficulty: advanced
---

**GrandCentral(Gc)**은 gipc-app의 객체 관리·메시징 중추다. UDT 객체 그래프 + GC 파라미터 버스 두 계층으로 구성된다.

## 전체 구조

```
┌─────────────────────────────────────────────────────────────────┐
│                     GrandCentral (Gc) 프레임워크                 │
│                                                                  │
│  ┌─────────────────────────────┐                                 │
│  │       UDT 객체 그래프        │  ← 상태를 가진 C++ 객체들        │
│  │                             │                                 │
│  │  [Gc.SWEAcqAssist]         │  ← SWEAcquisitionAssistant      │
│  │  [Gc.UGAPAcqAssist]        │  ← UGAPAcquisitionAssistant     │
│  │  [Gc.LiverAI]              │  ← LiverAIProcessor             │
│  │  ... (~30개 OVObject)       │                                 │
│  │                             │                                 │
│  │  GcUdtHandle<T> ─────────▶ │  (핸들로 접근)                  │
│  └─────────────────────────────┘                                 │
│                    │                                             │
│                    │ OVObject::SetParameter()                    │
│                    ▼                                             │
│  ┌─────────────────────────────┐                                 │
│  │      GC 파라미터 버스        │  ← 문자열 키/값 메시지           │
│  │                             │                                 │
│  │  SetParameterValue(k, v)   │                                 │
│  │  GetParameterValue(k)      │                                 │
│  │                             │                                 │
│  └─────────────────────────────┘                                 │
│                    ▲                                             │
└────────────────────│────────────────────────────────────────────┘
                     │
             ESMain.cpp (Layer A 진입점)
             11개 호출 사이트에서 파라미터 발행
```

## 프레임워크 진입/탈출 지점

```cpp
// 진입 1: 팩토리로 UDT 노드 생성
// OVObject.cpp:222 — "Gc.SWEAcqAssist" 태그 등록
// ObjectViewerImpl.cpp — 런타임 인스턴스화

// 진입 2: GC 파라미터 수신
void SWEAcquisitionAssistant::SetParameter(
    const std::string& key, const Variant& value) override
{
    // Layer A → Layer B 메시지 수신
}

// 탈출: AutoPos 키 역방향 전달
std::string key = handler->ResolveAutoPosKey(state);
if (!key.empty())
    ESMain::ProcessAutoPos(key);  // Layer B → Layer A
```

## 프레임워크가 관리하는 것 vs 개발자가 관리하는 것

```
프레임워크 관리:            개발자 관리:
  UDT 노드 수명              OVObject 내부 로직
  팩토리 등록/인스턴스화      GcUdtHandle IsValid() 확인
  파라미터 라우팅             파라미터 키 문자열 관리
  렌더링 루프 스케줄링         Render() 구현
  OnActivate/OnDeactivate     핸들 정리 (Release)
  스레드 안전 디스패치         내부 뮤텍스
```

## 알려진 제약과 설계 이유

```
제약: GC 파라미터 키는 문자열 — 타입 안전성 없음
이유: Layer A(C++11 정적 바이너리)와 Layer B(플러그인 가능 렌더러)를
     동일 프로세스 내 느슨하게 결합하기 위함. COM 버전 지옥 없이
     레이어 독립 업데이트 가능.

제약: OVObject는 OVObject에만 상속 가능
이유: Gc 팩토리가 OVObject* 타입으로 관리. 임의 클래스는 UDT 그래프
     진입 불가. → Composition으로 해결 (LiverWarning3Panel 추출)

제약: ~30개 OVObject 중 warning 인디케이터 보유 클래스 0개
이유: warning 로직이 AcquisitionAssistantBase에 묶여 있었음
     → LiverWarning3Panel 추출로 해결 중
```

## Key Points

- GrandCentral = **객체 그래프(UDT)** + **메시지 버스(파라미터)** 두 개의 독립 계층
- 두 계층은 협력 관계 — 파라미터 버스가 UDT 노드의 `SetParameter()`를 호출
- `"Gc."` 접두어가 붙은 모든 문자열은 GrandCentral 네임스페이스 — 임의 변경 금지
- 프레임워크의 스레드 안전 디스패치를 믿되, UDT 핸들의 `IsValid()` 는 여전히 개발자 책임
- 이 프레임워크를 이해하면 gipc-app의 모든 레이어 간 통신이 설명됨
