---
title: "새 OVObject 파생 클래스 추가"
category: workflow
tags: [OVObject, factory, COM, MFC, architecture]
difficulty: advanced
---

`OVObject` 상속 → 팩토리 등록 → `ObjectViewerImpl` 캐스트 → Manager 연결 순서로 진행한다.

## 절차

1. **헤더 작성** (`GcViewer/GcMyClass.h`)
   - `#pragma once` 첫 줄
   - GE CONFIDENTIAL 블록 (아래 `ge-confidential-header.md` 참고)
   - `class GcMyClass : public OVObject` — PascalCase, `Gc` 접두사
   - 순수 가상함수 `GetTag()`, `GetData()` 등 필수 구현 선언
   - 멤버변수 접두사 `m_`, 정적 `s_`

2. **팩토리 등록** (`OVObject.cpp` 222, 225번째 줄 패턴)
   ```cpp
   // OVObject.cpp ~222
   else if (tag == "Gc.MyClass")
       pObj = new GcMyClass();
   ```

3. **동적 캐스트 추가** (`ObjectViewerImpl.cpp` 2063, 2098번째 줄 패턴)
   ```cpp
   if (auto* p = dynamic_cast<GcMyClass*>(pObj))
   {
       // GcMyClass 전용 처리
   }
   ```

4. **Manager 연결** (`ESMain.cpp`)
   - 기존 `Manager::Instance(HandlerType::SWE)` 호출 사이트 옆에 추가
   - 외부 컨트랙트(공개 인터페이스, COM IDL) 변경 금지

## 예시

```cpp
// GcMyClass.h
#pragma once
/* -GE CONFIDENTIAL- ... */
#include "OVObject.h"

class GcMyClass : public OVObject
{
public:
    const char* GetTag() const override { return "Gc.MyClass"; }
private:
    int m_value = 0;
};
```

## 체크리스트

- [ ] `#pragma once` + GE CONFIDENTIAL 헤더
- [ ] PascalCase 클래스명, `m_` 멤버, `s_` 정적
- [ ] `OVObject.cpp` 팩토리 분기 추가
- [ ] `ObjectViewerImpl.cpp` `dynamic_cast` 분기 추가
- [ ] ESMain.cpp Manager 호출 사이트 확인
- [ ] 외부 COM/공개 인터페이스 변경 없음
- [ ] `/W4` 경고 없이 컴파일 통과
