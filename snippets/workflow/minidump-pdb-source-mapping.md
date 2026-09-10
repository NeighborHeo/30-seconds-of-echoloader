---
title: "현장 minidump → PDB → git 소스 연결하는 법"
category: workflow
tags: [debugging, windbg, minidump, crash, symbols, git, pdb]
difficulty: intermediate
---

현장 기기 .dmp 파일을 받았을 때 일치하는 소스 코드 버전을 찾아 라인 단위 분석까지 연결하는 절차.

## 언제 이 아티클을 보나

`crash-dump-analysis.md`의 WinDbg 명령어는 알지만, 덤프에 맞는 소스 버전을 찾는 법을 모를 때.
현장 기기는 최신 빌드가 아닐 수 있고, 재빌드하면 주소가 달라져서 PDB가 맞지 않는다.

## Step 1: 덤프에서 빌드 버전 확인

세 가지 방법 중 하나로 빌드 번호를 확인한다.

**방법 A: WinDbg — 모듈 버전 리소스**

```windbg
; 덤프 열기 후
lm                        ; 로드된 모듈 목록
lmi EchoLoader            ; EchoLoader.exe 상세 정보
; 출력 예:
;   Image name: EchoLoader.exe
;   Timestamp:  66A3F210 (Mon Jul 29 2026)
;   File version: 5.2.1.2314
```

**방법 B: 기기에서 직접 확인**

앱 메뉴 → 소프트웨어 버전 화면에서 `5.2.1.2314` 형식(Major.Minor.Patch.BuildNumber) 확인.

**방법 C: 타임스탬프 역산**

```windbg
; lmi 출력의 Timestamp 값을 UTC로 변환
.formats 66A3F210
; Date fields → 해당 날짜 빌드 번호를 빌드 서버 로그에서 검색
```

## Step 2: 빌드 번호 → git 커밋 매핑

빌드 시스템은 각 빌드에 git 커밋 해시를 두 곳에 삽입한다.

```windbg
; WinDbg에서 버전 리소스 전체 출력
!sxe ld EchoLoader.exe
g
lmi EchoLoader
; "Comments:" 또는 "ProductVersion:" 필드에 git 해시 포함
; 예: Comments: git=a3f8c12d BuildNum=2314
```

코드에서 확인하는 경우:

```cpp
// ScCommon::Version::GetBuildString() 반환값 형식
// "5.2.1.2314 (git: a3f8c12d)"
// 이 문자열은 앱 메뉴 버전 화면에도 표시됨

// EchoLoader.rc 에도 동일 해시가 기록됨
// FILEVERSION 5,2,1,2314
// VALUE "Comments", "git=a3f8c12d"
```

## Step 3: git 커밋 체크아웃

```bash
# 해시로 직접 체크아웃
git checkout a3f8c12d

# 릴리즈 태그가 있는 경우
git checkout v5.2.1.2314

# 체크아웃 후 소스 루트 경로를 기록해 둔다 — Step 5에서 사용
pwd
# 예: C:\src\gipc-app
```

## Step 4: PDB 파일 위치 지정

PDB는 빌드 서버 심볼 스토어에 보관된다. GUID 기반 경로 구조:

```
\\build-server\symbols\EchoLoader.pdb\<GUID>\EchoLoader.pdb
```

WinDbg에서 심볼 경로 추가:

```windbg
; 빌드 서버 심볼 스토어 + MS 퍼블릭 심볼 서버 병행
.sympath+ \\build-server\symbols
.sympath+ srv*C:\symbols*https://msdl.microsoft.com/download/symbols
.reload /f EchoLoader.exe

; 심볼 로드 상태 확인
lm v m EchoLoader
; "private pdb symbols" → PDB 정상 로드
; "export symbols"      → PDB 없음 (Step 실패 시나리오 참조)
```

## Step 5: 소스 연결 확인

```windbg
; 소스 루트 경로 설정 (Step 3에서 체크아웃한 경로)
.srcpath+ C:\src\gipc-app

; 자동 분석 실행
!analyze -v
; 성공 시 출력 예:
;   FAULTING_SOURCE_FILE: C:\src\gipc-app\src\EchoScanner\ESMain.cpp
;   FAULTING_SOURCE_LINE: 247

; 스택에서 소스 라인 확인
k
; #03 SWEAcquisitionAssistant::Render [ESMain.cpp @ 247]
```

## 실패 시나리오 → 원인

| 증상 | 원인 | 해결 |
|------|------|------|
| PDB가 없다 | 릴리즈 빌드 PDB를 심볼 스토어에 보관하지 않음 | 빌드 서버 post-build 정책 확인 — PDB 아카이브 단계 추가 필요 |
| 소스 라인이 안 나온다 | `.srcpath` 경로가 git 체크아웃 루트와 불일치 | `git checkout` 경로를 그대로 `.srcpath+`에 사용 |
| 함수명만 나오고 라인 없다 | PDB가 Export-only (Full PDB 아님) | 빌드 설정에서 `/Zi` (Full Debug Info) 확인 — `/Z7` 또는 `/Zd`이면 Full PDB 아님 |
| 버전이 아예 안 보인다 | `lm` 목록에 EchoLoader.exe가 없음 — 모듈 로드 실패 | `lm` 전체 출력으로 로드된 모듈 열거 후 올바른 모듈명 확인 |

## Key Points

- Release 빌드 PDB는 재빌드하면 GUID가 바뀐다 — **반드시 같은 빌드**의 PDB를 사용해야 한다
- `lmi EchoLoader`의 `Comments` 필드에 git 해시가 없으면, `.rc` 파일 빌드 규칙에 해시 삽입이 빠진 것이다
- `.sympath+`는 기존 경로를 유지하며 추가한다 — `.sympath`(플러스 없음)는 기존 경로를 **초기화**하므로 주의
- `!analyze -v` 후 소스 라인이 나오면 `kn` 으로 전체 스택 프레임 번호와 소스 위치를 함께 출력할 수 있다
- git 해시가 없는 구 빌드는 빌드 서버 로그의 날짜/타임스탬프로 커밋을 `git log --after --before`로 좁혀야 한다
