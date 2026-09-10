---
title: "40k 파일 코드베이스에서 원하는 코드 찾기"
category: workflow
tags: [grep, search, navigation, debugging, gcviewer, echoscanner]
difficulty: beginner
---

버그 리포트나 기능 요청을 받았을 때, 40k 파일 중 관련 코드를 빠르게 찾기 위한 검색 전략 모음.

## 언제 이 아티클을 보나

"SWE 모드에서 경고가 안 뜬다"처럼 기능 이름만 있고, 어느 파일을 봐야 할지 모를 때. 또는 수정하려는 함수가 어디서 호출되는지 확인해야 할 때.

## 시나리오 1: 기능 이름으로 찾기 (버그 리포트에서 출발)

```bash
# "SWE 모드에서 경고가 안 뜬다" → 경고 관련 코드 찾기
grep -r "LargeS\|PoorProbe\|ObliqueCapsule" src/ --include="*.h" -l
grep -r "Warning" src/GcViewer/ObjectViewer/ --include="*.cpp" -l

# OVObject 태그로 찾기 (GC 프레임워크 진입점)
grep -r "Gc\.SWE\|Gc\.UGAP\|Gc\.Liver" src/ --include="*.cpp"
```

## 시나리오 2: GC 파라미터 키로 추적하기

```bash
# "SWEAcqAssist.State" 파라미터가 어디서 발행되고 어디서 수신되나?
grep -r "SWEAcqAssist\.State" src/ --include="*.cpp"
grep -r "SWEAcqAssist\.State" src/ --include="*.h"

# 패턴: SetParameterValue 발행처 → SetParameter 수신처
grep -r "SetParameterValue.*SWE" src/ --include="*.cpp"
grep -r "SetParameter.*SWE\|\"SWE" src/GcViewer/ --include="*.cpp"
```

GC 파라미터는 문자열 키 기반이므로, 키 이름 전체 또는 일부로 grep하면 발행처와 수신처를 모두 찾을 수 있다.

## 시나리오 3: ESMain.cpp에서 특정 기능 찾기 (10k줄 파일)

```bash
# AA 관련 코드만 볼 것 — 7066~9954 라인 범위
grep -n "Manager::Instance\|InElasto\|IsInUGAPMode" src/EchoScanner/ESMain.cpp

# 특정 이벤트 핸들러 찾기
grep -n "OnNewFrame\|HandleUserInput\|UpdateAcqAssist" src/EchoScanner/ESMain.cpp

# ESMain의 모든 멤버 함수 목록 (라인 번호 포함)
grep -n "void ESMain::\|bool ESMain::\|int ESMain::" src/EchoScanner/ESMain.cpp
```

## 시나리오 4: 클래스/함수 정의 찾기

```bash
# 선언(.h)과 정의(.cpp) 동시에
grep -rn "class SWEAcquisitionAssistant" src/ --include="*.h"
grep -rn "SWEAcquisitionAssistant::" src/ --include="*.cpp" | head -20

# 상속 체계 파악
grep -rn ": public OVObject\|: public AcquisitionAssistantBase" src/ --include="*.h"
```

## 시나리오 5: 역방향 의존 확인 (레이어 위반 검사)

```bash
# GcViewer가 EchoScanner를 포함하고 있나? (역방향 의존 확인)
grep -r "include.*EchoScanner" src/GcViewer/ --include="*.h"
grep -r "include.*EchoScanner" src/GcViewer/ --include="*.cpp"
# → 결과 있으면 즉시 제거 필요

# ScCommon이 상위 레이어를 포함하나?
grep -r "include.*EchoScanner\|include.*GcViewer" src/ScCommon/ --include="*.h"
```

역방향 의존이 발견되면 그 자체가 버그다. 수정 전에 팀에 알린다.

## 시나리오 6: 변경 영향 범위 파악 (내가 바꾸려는 함수를 누가 호출하나?)

```bash
# OnNewFrame을 바꾸려면 모든 호출처 확인
grep -rn "->OnNewFrame\|\.OnNewFrame" src/ --include="*.cpp"

# 특정 헤더를 포함하는 파일들 (파급 범위 파악)
grep -rn "include.*AcquisitionAssistantBase.h" src/ --include="*.cpp"

# 특정 함수의 모든 호출처
grep -rn "UpdateAcqAssist\b" src/ --include="*.cpp"
```

## 탐색 우선순위 표

| 찾으려는 것 | 먼저 볼 위치 |
|---|---|
| 기능의 레이어 경계 | `where-does-new-code-go.md` 결정 트리 |
| GC 파라미터 발행처 | `grep -r "SetParameterValue.*<키이름>"` |
| GC 파라미터 수신처 | `grep -r "SetParameter.*<키이름>"` src/GcViewer/ |
| OVObject 등록 위치 | OVObject.cpp 약 222번째 줄 |
| ESMain AA 호출 사이트 | ESMain.cpp 7066~9954 라인 |
| 상속 체계 | `grep -rn ": public OVObject"` src/ --include="*.h" |
| 새 기능 추가 위치 | `where-does-new-code-go.md` 결정 트리 |

## Key Points

- GC 파라미터 키 이름(문자열)으로 grep하면 발행처와 수신처를 동시에 찾을 수 있다.
- ESMain.cpp에서 AA 관련 코드는 7066~9954 라인에만 있다. 전체를 읽지 않아도 된다.
- `grep -l`(파일 목록만)로 먼저 범위를 좁히고, 해당 파일에서 세부 탐색한다.
- 역방향 의존(`GcViewer`가 `EchoScanner`를 include) 발견 시 수정 작업보다 우선 보고한다.
- 함수 수정 전에 반드시 `grep -rn "->함수명"` 으로 모든 호출처를 확인하고 영향 범위를 파악한다.
