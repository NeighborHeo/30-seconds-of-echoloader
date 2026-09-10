---
title: "크래시 덤프 분석 (WinDbg + gipc-app)"
category: workflow
tags: [debugging, windbg, minidump, crash, symbols]
difficulty: intermediate
---

gipc-app이 임상 환경에서 크래시했을 때 .dmp 파일로 원인을 찾는 절차.

## Why

Release 빌드는 최적화·인라이닝으로 스택이 망가진다. .pdb 없이는 주소만 보인다.
WinDbg + 매칭 .pdb 없이는 크래시 원인을 특정할 수 없다.

## Pattern

```windbg
; 1. WinDbg 실행 후 덤프 열기
File > Open Crash Dump > gipc_app_20260903_143201.dmp

; 2. 심볼 경로 설정 (빌드 서버의 .pdb 경로 + MS 퍼블릭 서버)
.sympath srv*C:\symbols*\\build-server\symbols;srv*C:\symbols*https://msdl.microsoft.com/download/symbols
.reload /f

; 3. 자동 분석 실행 — 가장 먼저
!analyze -v

; 4. 출력에서 핵심 필드 확인
; FAULTING_MODULE: gipc_render.dll
; FAULTING_IP:     gipc_render!CRenderPipeline::Render+0x3a8
; EXCEPTION_CODE:  0xC0000005 (Access Violation, read of address 0x00000000)

; 5. 스택 전체 출력
k
; 또는 모든 스레드 스택 한꺼번에
~*kb

; 6. 특정 스레드로 이동 (렌더 스레드가 의심될 때)
~2s
k

; 7. 로컬 변수 / 레지스터 확인
dv          ; 현재 프레임 지역 변수
r           ; 레지스터 덤프
dt ovHandle ; OVObjectHandle 구조체 출력

; 8. RBUG/FBUG 연결
; FAULTING_IP 주소를 빌드 서버 맵 파일(.map)에서 검색
; 또는 !lmi gipc_render 로 모듈 타임스탬프 확인 → 빌드 번호 → Jira 검색
!lmi gipc_render
```

**Release 빌드와 .pdb 매칭 주의사항:**

```
; 심볼이 로드됐는지 확인
lm v m gipc_render
; "private pdb symbols" 또는 "export symbols" 여부 표시
; export symbols = .pdb 없음 → 함수 이름만, 라인 번호 없음

; .pdb 경로가 잘못됐을 때
.symfix+ C:\symbols
.reload /f gipc_render.dll
```

**gipc-app 3대 크래시 패턴:**

```cpp
// 패턴 1: IsValid() 누락 — null OVObject 역참조
// !analyze 출력: Access Violation at 0x00000000
// 스택에 COVObjectHandle::operator-> 가 보임
COVObjectHandle<CSWEAcqAssist> hAssist = GetAssist();
// 수정 전: hAssist->DoSomething();
// 수정 후:
if (!hAssist.IsValid()) return;
hAssist->DoSomething();

// 패턴 2: 렌더 스레드 스택 오버플로
// !analyze: EXCEPTION_CODE 0xC00000FD (Stack Overflow)
// 스택에 CRenderPipeline::Render 가 수십 번 반복
// → Render() 안의 재귀 호출 경로 확인
// EXCEPTION_PARAMETER: 스택 포인터 값이 스택 하단 근처

// 패턴 3: COM 객체 조기 해제
// !analyze: Access Violation in combase.dll
// 스택에 IUnknown::Release 이후 vtable 접근 흔적
// → AddRef/Release 불균형, 스마트 포인터 수명 확인
CComPtr<IFoo> pFoo = GetFoo();
// 수정 전: pFoo.Release(); Use(pFoo);  // UAF
// 수정 후: Use(pFoo); (Release는 CComPtr 소멸자에 맡김)
```

## Key Points

- `.sympath` 설정 후 반드시 `.reload /f` 실행해야 심볼이 적용된다
- `!analyze -v` 출력의 `FAULTING_IP` + `.map` 파일로 소스 라인 역추적 가능
- Release 빌드 .pdb는 빌드 서버에서 **빌드 번호 일치** 파일을 사용해야 한다 — 재빌드하면 주소가 달라짐
- 렌더 스레드 스택 오버플로는 `~*kb` 로 전 스레드 스택을 확인해야 발견됨
- RBUG 연결: `!lmi <module>` 의 타임스탬프 → 빌드 로그 → 커밋 → Jira 티켓
