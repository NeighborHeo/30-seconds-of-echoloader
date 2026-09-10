---
title: "gipc-app 빌드 오류 TOP 5 — 원인과 수정"
category: workflow
tags: [build, msvc, linker, error, c4100, lnk2019]
difficulty: beginner
---

gipc-app에서 자주 보이는 MSVC 빌드·링크 오류 5가지와 확실한 수정 방법.

## Why

W4 + warnings-as-errors 환경에서 새 코드를 추가하면 기존에 안 보이던 오류가 터진다.
오류 메시지만 알면 수정은 1분 안에 끝난다.

## Pattern

**Error 1 — C4100: 미사용 형식 매개변수**

```
error C4100: 'param': unreferenced formal parameter
```

```cpp
// 원인: 함수 서명에 파라미터가 있지만 본문에서 쓰지 않음
// (virtual override, 콜백 시그니처 맞추기 위해 흔히 발생)

// 수정 A: void 캐스트 (이름 유지, 경고 억제)
void CFoo::OnEvent(int nEventId)
{
    (void)nEventId;
    DoSomething();
}

// 수정 B: 이름 제거 (파라미터 타입만 남김)
void CFoo::OnEvent(int /*nEventId*/)
{
    DoSomething();
}
```

---

**Error 2 — C2664: COM QueryInterface 변환 실패**

```
error C2664: 'HRESULT IUnknown::QueryInterface(REFIID,void **)':
cannot convert argument 2 from 'IFoo **' to 'void **'
```

```cpp
// 원인: QueryInterface 두 번째 인자는 void** 이어야 함
// IUnknown* 캐스트 없이 직접 넘기면 컴파일 실패

// 수정: IUnknown** 로 캐스트
IFoo* pFoo = nullptr;
pUnknown->QueryInterface(IID_IFoo, reinterpret_cast<void**>(&pFoo));

// 또는 CComPtr 사용 (내부적으로 캐스트 처리)
CComPtr<IFoo> spFoo;
pUnknown->QueryInterface(IID_PPV_ARGS(&spFoo));
```

---

**Error 3 — LNK2019: 외부 심볼 미해결**

```
error LNK2019: unresolved external symbol "public: void __cdecl CBar::DoWork(void)"
referenced in function "void __cdecl CFoo::Run(void)"
```

```cpp
// 원인: .lib 파일이 Additional Dependencies에 없거나
//       새 패키지를 사용하면서 .vcxproj 를 업데이트하지 않음

// 수정: gipc-app 의 경우 CppProjectBase.props 또는
// 해당 .vcxproj 의 AdditionalDependencies 에 추가
// <AdditionalDependencies>BarLib.lib;%(AdditionalDependencies)</AdditionalDependencies>

// 패키지 이름 확인: 누락된 심볼을 grep 으로 어느 .lib 에 있는지 찾기
// dumpbin /exports BarLib.lib | findstr "DoWork"
```

---

**Error 4 — C4996: deprecated 함수**

```
error C4996: 'strcpy': This function or variable may be unsafe.
Consider using strcpy_s instead.
```

```cpp
// 수정 A (권장): 보안 버전으로 교체
char szBuf[256];
strcpy_s(szBuf, sizeof(szBuf), pSrc);

// 수정 B (레거시 코드 대량 수정 불가할 때만 — 최후 수단):
// .vcxproj 또는 CppProjectBase.props 의 PreprocessorDefinitions 에 추가
// _CRT_SECURE_NO_WARNINGS
// ↑ 신규 코드에는 절대 쓰지 말 것. 레거시 래퍼 빌드에만 허용.

// 수정 C: std::string / std::array 로 교체 (신규 코드 권장)
std::string sName = pSrc;
```

---

**Error 5 — C1083: include 파일을 열 수 없음**

```
fatal error C1083: Cannot open include file: 'ScCommon/ScLog.h': No such file or directory
```

```
; 원인: 새 패키지 헤더를 추가했지만 Additional Include Directories 미등록

; 수정 위치: CppProjectBase.props (공통) 또는 해당 .vcxproj
; <AdditionalIncludeDirectories>
;   $(SolutionDir)\..\ScCommon\include;%(AdditionalIncludeDirectories)
; </AdditionalIncludeDirectories>

; 패키지 include 경로 규칙: $(SolutionDir)\..\<PackageName>\include
; 경로 추가 후 반드시 "모두 다시 빌드" — 증분 빌드는 헤더 경로 변경을 감지 못할 수 있음
```

## Key Points

- C4100은 `(void)param;` 또는 `/*param*/` 주석으로 즉시 해결 — 가장 자주 나오는 오류
- COM `QueryInterface`는 항상 `IID_PPV_ARGS` 매크로를 써서 void** 캐스트를 자동화하라
- LNK2019가 나오면 먼저 `.lib` 누락을 의심하고, `dumpbin /exports`로 심볼 소재 확인
- C4996은 신규 코드에서는 `_s` 버전으로, 레거시 대량 빌드에서는 `.props` 억제 — 혼용 금지
- C1083 후 증분 빌드가 잘못된 결과를 내면 `/clean` 후 전체 빌드로 초기화
