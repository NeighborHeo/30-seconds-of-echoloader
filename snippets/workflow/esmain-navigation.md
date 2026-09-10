---
title: "ESMain.cpp 10,000줄 탐색법"
category: workflow
tags: [esmain, navigation, aa, acquisition-assistant, gating, manager]
difficulty: intermediate
---

ESMain.cpp(9,954줄)를 처음부터 읽지 않고, 작업에 필요한 구역만 정확히 찾아가는 방법.

## 언제 이 아티클을 보나

AA(AcquisitionAssistant) 관련 호출 사이트를 추가하거나, ESMain에서 특정 이벤트 핸들러를 수정해야 하는데 어디서 시작해야 할지 모를 때.

## ESMain.cpp 구역 지도

```
라인 범위        | 내용
-----------------|------------------------------------------
1    ~ 200      | #include, 전역 선언, 전처리 매크로
200  ~ 1000     | 초기화/셧다운 함수들 (Initialize, Shutdown)
1000 ~ 3000     | 스캔 모드 전환, 프로브 이벤트 핸들러
3000 ~ 5000     | 사용자 입력 핸들러 (버튼, 노브, 터치)
5000 ~ 7000     | GC 파라미터 발행 함수들
7066 ~ 9954     | AA(AcquisitionAssistant) 호출 사이트 11개
9954            | 파일 끝
```

## 작업별 탐색 전략

```bash
# 1. 새 AA 호출 사이트 추가 위치 찾기
# → 7066~9954 범위의 기존 호출 사이트 옆에 추가
grep -n "Manager::Instance" src/EchoScanner/ESMain.cpp
# 첫 번째 결과 → 기준점으로 삼아 주변 코드 읽기

# 2. 특정 이벤트 핸들러 찾기
grep -n "void ESMain::" src/EchoScanner/ESMain.cpp
# 함수 목록 출력 → 관련 함수 라인 번호 확인

# 3. GC 파라미터 발행 위치 찾기 (5000~7000 구역)
grep -n "SetParameterValue" src/EchoScanner/ESMain.cpp
# 파라미터 키 이름으로 필터

# 4. 게이팅 술어 사용 위치 확인
grep -n "InElasto()\|IsInUGAPMode()" src/EchoScanner/ESMain.cpp | head -30
```

## ESMain 수정 시 반드시 지켜야 할 규칙

**규칙 1: 새 AA 관련 코드는 반드시 게이팅 후 추가**

```cpp
if (InElasto())       // SWE만 실행
if (IsInUGAPMode())   // UGAP만 실행
// 게이팅 없음 → 모든 모드에서 실행됨 (의도한 경우에만)
```

**규칙 2: ESMain이 직접 AA 로직을 갖지 않음**

```cpp
// ❌ ESMain에서 직접 로직
if (InElasto()) { m_sweState = newState; UpdateUI(); }

// ✅ Manager에 위임
if (InElasto()) Manager::Instance(HandlerType::SWE)->UpdateState(newState);
```

**규칙 3: 11개 기존 사이트 패턴을 먼저 읽고 따라 작성**

새 호출 사이트는 기존 사이트의 if-guard + `Manager::Instance()` 패턴을 그대로 복사하는 것에서 시작한다. 패턴이 다르면 코드 리뷰에서 반려된다.

## ESMain에서 절대 하지 말 것

| 금지 사항 | 이유 |
|---|---|
| 9,954줄 이상으로 늘리기 | 기능은 Manager로 이동하는 것이 원칙 |
| 게이팅 없이 AA 코드 추가 | 모든 모드에서 실행되어 다른 모드 깨짐 |
| `#include "GcViewer/..."` 추가 | Layer A → Layer B 역방향 의존 금지 |
| 새 전역 변수 추가 | Manager 내부 상태로 대신 |
| ESMain 내부에서 직접 상태 계산 | Manager/Handler에 위임 |

## VS에서 실용적인 탐색

```
Ctrl+G              → 라인 번호로 바로 이동 (7066부터 보기)
Ctrl+F              → 현재 파일에서 검색
Ctrl+]              → 심볼 정의로 이동 (Manager::Instance 정의로)
우클릭 → Find All References → 호출처 전체 목록
```

ESMain에서 `Manager::Instance`를 찾았으면, `Ctrl+]`로 해당 Manager 정의로 바로 이동한다. AA 로직의 실제 내용은 Manager/Handler에 있다.

## Key Points

- AA 관련 코드는 7066~9954 라인에만 있다. 새 호출 사이트도 이 구역에 추가.
- ESMain은 진입점 + 게이팅만 담당. 로직 자체는 `Manager::Instance()->메서드()` 호출로 위임.
- `InElasto()` / `IsInUGAPMode()` 게이팅 없이 AA 코드를 추가하면 반드시 다른 모드에서 버그가 발생한다.
- 기존 11개 호출 사이트 중 유사한 것을 먼저 찾아 패턴을 확인하고 따라 작성한다.
- `#include "GcViewer/..."` 는 레이어 위반이다. ESMain에서 GcViewer 헤더를 포함하는 코드가 보이면 기존 위반이므로 새로 추가하지 않는다.
