---
title: "내 OVObject가 Active인지 확인하는 법"
category: workflow
tags: [ovobject, debugging, grandcentral, objectviewer, render]
difficulty: intermediate
---

`Render()` 또는 `SetParameter()`가 전혀 실행되지 않을 때, OVObject가 실제로 Active 상태인지 단계별로 검증하는 법.

## 언제 이 아티클을 보나

- `Render()` 안에 로직을 넣었는데 화면에 아무 변화가 없다
- `SetParameter()`가 호출되는지 확인할 수 없다
- `OnActivate()` → `Render()` 흐름이 실제로 타는지 의심스럽다

## Method 1: 로그로 확인 (가장 빠름)

`OnActivate()`와 `Render()` 양쪽에 임시 로그를 심는다. `OnActivate()` 로그가 없으면 등록 문제다.

```cpp
void MyOVObject::OnActivate()
{
    ScLogInfo("MYOBJ", "OnActivate() called — 이 로그가 없으면 Active 아님");
    // ... 초기화
}

void MyOVObject::Render()
{
    // 임시 — 커밋 전 제거
    static int s_renderCount = 0;
    if (++s_renderCount <= 5)  // 처음 5프레임만
        ScLogInfo("MYOBJ", "Render() 호출됨 — count=%d", s_renderCount);
}
```

`OnActivate()` 로그가 보이면 등록은 정상. `Render()` 로그가 없으면 GrandCentral 렌더 루프 등록 문제.

## Method 2: 팩토리 등록 확인

태그 오타가 가장 흔한 원인이다. 등록 태그와 참조 태그가 정확히 일치해야 한다.

```bash
# OVObject.cpp에서 태그 등록 확인
grep -n "Gc.MyObject\|MyOVObject" src/GcViewer/ObjectViewer/OVObject.cpp

# 태그 오타가 흔한 원인:
# "Gc.MyObject" (등록) vs "Gc.MyObj" (다른 곳에서 참조) → 불일치
```

## Method 3: ObjectViewerImpl에서 인스턴스 생성 확인

`ObjectViewerImpl.cpp`의 `dynamic_cast` 체인에 새 OVObject가 추가됐는지 확인한다.

```bash
# dynamic_cast 체인에 추가됐는지
grep -n "MyOVObject" src/GcViewer/ObjectViewer/ObjectViewerImpl.cpp
```

결과가 없으면 ObjectViewerImpl이 인스턴스를 생성하지 않는다. 추가 필요.

## Method 4: 활성화 조건 추적

코드 등록이 모두 맞아도 GrandCentral이 해당 모드에서 OVObject를 활성화하지 않으면 `OnActivate()`는 불리지 않는다.

```
OVObject가 Inactive인 일반적 이유:
  1. 팩토리 태그 등록 누락/오타 → OVObject.cpp 확인
  2. ObjectViewerImpl의 dynamic_cast 분기 누락 → ObjectViewerImpl.cpp 확인
  3. GrandCentral이 이 모드에서 해당 OVObject를 활성화하지 않음
     → 어떤 조건에서 OnActivate가 불리는지 확인 (모드 전환 시점)
  4. ESMain.cpp에서 GC 태그를 요청하지 않음 (태그 기반 on-demand 활성화 시)
```

## 진단 흐름

```
내 Render()가 안 불린다
  ↓
OnActivate() 로그 있나?
  NO → 팩토리 등록 확인 (OVObject.cpp, ObjectViewerImpl.cpp)
  YES → Render()에 임시 로그 추가
         Render() 로그 없나?
           → GrandCentral 렌더 루프에 등록됐는지 확인
           → 해당 모드 진입 후 활성화되는지 확인
```

## Key Points

- `OnActivate()` 로그 유무가 "등록 문제"와 "런타임 활성화 문제"를 가르는 첫 번째 기준이다
- 팩토리 태그는 `OVObject.cpp`(등록)와 참조 측 모두 동일한 문자열이어야 한다 — 오타는 무음 실패(silent failure)
- `ObjectViewerImpl.cpp`의 `dynamic_cast` 체인 누락은 인스턴스 자체가 생성되지 않아 `OnActivate()`가 영원히 불리지 않는다
- GrandCentral이 특정 모드에서만 OVObject를 활성화하는 경우, 모드 진입 타이밍과 `OnActivate()` 호출 시점을 로그로 교차 확인한다
- 임시 `ScLogInfo` 로그는 커밋 전 반드시 제거한다 — `static` 카운터 패턴으로 로그 폭발을 방지하되 운영 코드에 남기지 않는다
