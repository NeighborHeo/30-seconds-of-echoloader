---
title: "왜 GC 파라미터 버스인가: 설계 결정의 이유"
category: architecture
tags: [gc-params, architecture, design-rationale, layer-boundary, decoupling]
difficulty: advanced
---

GC 파라미터 버스는 Layer A(EchoScanner)와 Layer B(GcViewer) 사이에서 유일하게 허용된 통신 수단이다. 이것이 그냥 관행이 아니라 26년에 걸쳐 검증된 설계 결정임을 이해해야 올바르게 쓸 수 있다.

## Why / 왜 알아야 하나

GC 파라미터 버스를 "그냥 있는 것"으로 다루면 두 가지 실수가 생긴다. 첫째, 편하다는 이유로 직접 `#include`를 넣어 레이어를 우회하는 것. 둘째, 버스를 통하되 문자열 키를 부주의하게 다뤄 사일런트 실패를 심는 것. 이 문서는 그 결정의 근거를 설명한다.

## 거절된 대안들

### 대안 1: 직접 함수 호출 (레이어 간 #include)

```
// 시나리오 A: ESMain이 GcViewer를 직접 호출
#include "GcViewer/ObjectViewer/SWEAcquisitionAssistant.h"
// 결과: EchoScanner → GcViewer 의존성 추가

// 시나리오 B: GcViewer가 EchoScanner를 직접 호출
#include "EchoScanner/AcquisitionAssistant/AcquisitionAssistant.h"
// 결과: GcViewer → EchoScanner 의존성 추가
```

두 시나리오 모두 A → B → A 순환 빌드를 만든다. `gipc-app`의 패키지 수는 수백 개에 달하고, 하나의 순환이 MSBuild 전체 빌드를 멈춘다. 팀 전체가 블로킹된다.

```
ESMain.cpp (Layer A)
    │
    │  #include "SWEAcquisitionAssistant.h"  ← 금지
    ▼
GcViewer (Layer B)
    │
    │  #include "AcquisitionAssistant.h"     ← 금지
    ▼
ESMain.cpp (Layer A)  ← 순환!
```

비용 외에도, Layer B가 Layer A의 내부를 알게 되는 순간 Layer A를 리팩토링할 때마다 Layer B도 깨진다. 두 레이어가 "독립적으로 업데이트 가능"하다는 전제가 무너진다.

### 대안 2: COM 이벤트/인터페이스

`gipc-app` 코드베이스는 COM/ATL을 광범위하게 사용한다. COM 이벤트로 레이어 간 통신하는 방법이 논리적으로 보일 수 있다. 두 가지 이유로 거절되었다.

**첫째, COM 인터페이스는 버전이 고정된다.** GUID를 레지스트리와 코드 두 곳에 하드코딩해야 하며, 인터페이스를 변경하면 새 GUID + 모든 구현체 업데이트가 강제된다. `SWEAcqAssistHandler`에 파라미터 하나를 추가하는 일이 COM 버전 협의 절차로 번진다.

**둘째, 이미 복잡한 코드베이스를 더 복잡하게 만든다.** COM reference counting + `QueryInterface()` 패턴은 26년 레거시에 쌓이면 디버깅이 극도로 어려워진다.

### 대안 3: 공유 메모리 / 전역 변수

간단해 보이지만, 소유권 문제와 스레드 안전 문제 두 가지가 동시에 발생한다.

| 문제 | 결과 |
|------|------|
| 소유권 불명확 | double-free, use-after-free |
| 비 원자적 접근 | 렌더 스레드와 획득 스레드 간 레이스 컨디션 |
| 초기화 순서 불명확 | SIOF (Static Initialization Order Fiasco) |

## 왜 문자열 키/값 버스인가

GC 파라미터 버스가 선택된 이유는 그것이 위 세 대안의 비용 없이 세 가지 요구사항을 동시에 만족하기 때문이다.

```
요구사항 1: Layer A와 Layer B가 같은 프로세스 내에서 독립적으로 업데이트될 것
요구사항 2: 새 소비자(OVObject)를 추가해도 기존 코드가 바뀌지 않을 것
요구사항 3: Layer A 로직을 Layer B 없이 단위 테스트할 수 있을 것
```

**요구사항 1 충족:** 문자열 키/값 교환은 바이너리 의존성이 없다. `SWEAcquisitionAssistant`의 내부가 바뀌어도 `ESMain.cpp`는 재컴파일할 필요가 없다.

**요구사항 2 충족:** 새 OVObject가 `"SWEAcqAssist.State"` 키를 구독하는 것만으로 기존 코드를 전혀 수정하지 않고 통합된다. `ESMain.cpp`의 11개 호출 사이트는 변하지 않는다.

**요구사항 3 충족:** Layer A (`AcquisitionAssistant.cpp`, `SWEAcqAssistHandler`)는 Layer B 헤더를 하나도 포함하지 않는다. `TestSWEAlgorithm/`, `TestUGAPAlgorithm/` 단위 테스트가 GcViewer 없이 독립 실행되는 이유다.

## 실제 코드 흐름

```
ESMain.cpp (Layer A)
│
│  // 모드 전환 시 파라미터 발행
│  ESMain::SetParameterValue("SWEAcqAssist.State",
│      static_cast<int>(AcqAssistState::Freeze));
│  ESMain::SetParameterValue("SWEAcqAssist.RoiX", roiX);
│
▼
GC Parameter Bus (런타임 라우팅, 문자열 키 매칭)
│
▼  SetParameter() 호출
SWEAcquisitionAssistant (Layer B)
│
│  void SetParameter(const std::string& key, const Variant& value) override
│  {
│      if (key == "SWEAcqAssist.State")
│          m_currentState = static_cast<AcqAssistState>(value.AsInt());
│      else if (key == "SWEAcqAssist.RoiX")
│          m_roiX = value.AsFloat();
│  }
│
│  // 역방향: AutoPos 키 반환
▼
SWEAcqAssistHandler::ResolveAutoPosKey()
    → "SWEAcqAssistToggleContinuous" 등 문자열 키 반환
    → ESMain::ProcessAutoPos()가 수신
```

## 비용 솔직하게 인정하기

버스가 완벽한 해법은 아니다. 아래 비용을 알고 써야 한다.

| 장점 | 단점 |
|------|------|
| 레이어 독립 업데이트 가능 | 키 타입 안전성 없음 (컴파일 타임 오류 없음) |
| 새 구독자 추가 = OVObject 등록만으로 끝 | 키 오타 = 사일런트 드롭 (로그에 에러 없음) |
| 스레드 안전 라우팅 내장 | 디스패치 오버헤드 (미미하지만 존재) |
| Layer A 단위 테스트 용이 | 파라미터 계약이 문서화 안 되면 암묵적 |

키 오타 문제는 `constexpr`로 상수화하면 완화된다:

```cpp
// 권장: namespace에 키 상수 정의
namespace GcKeys {
    constexpr const char* SWEAcqAssistState = "SWEAcqAssist.State";
    constexpr const char* SWEAcqAssistRoiX  = "SWEAcqAssist.RoiX";
}
// "SWEAcqAssist.Stat" 오타가 링크 에러로 드러남
```

## 이 설계를 존중해야 하는 이유

26년 동안 이 계약이 Layer A와 Layer B의 독립 진화를 가능하게 했다. EchoScanner 획득 로직이 업데이트될 때 GcViewer 렌더러는 그 변경을 몰라도 된다. 반대도 마찬가지다.

이것을 우회하는 순간의 시나리오:

```
1. 개발자 X가 편의상 GcViewer 헤더를 EchoScanner에 #include
2. 다음 빌드: LNK2005 또는 순환 빌드 실패
3. CI 없음 → 팀 전체 빌드 환경이 깨진 채 시간 소비
4. 원인 추적: grep 필요, 수정: include 제거 + 파라미터 버스로 교체
5. 낭비된 시간 = 수 시간 ~ 하루
```

## Key Points

- GC 파라미터 버스는 편의 패턴이 아니라 레이어 간 바이너리 독립성을 보장하는 **아키텍처 계약**이다.
- 직접 `#include` 대안은 50개 패키지 빌드 순환을 만든다 — 이론이 아니라 실제로 팀을 블로킹한다.
- 문자열 키의 타입 안전성 부재는 `constexpr const char*` 상수화로 완화한다 — 오타를 링크 타임에 잡는다.
- Layer A(`AcquisitionAssistant.cpp`)가 Layer B 헤더를 하나도 포함하지 않는 것이 단위 테스트 독립성의 전제 조건이다.
- 버스를 우회하는 코드를 보면 즉시 리뷰에서 차단하라 — 26년간 유지된 경계를 한 줄이 무너뜨린다.
