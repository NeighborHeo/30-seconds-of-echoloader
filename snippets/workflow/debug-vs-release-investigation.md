---
title: "Debug 빌드는 되고 Release 빌드는 안 되는 버그 조사법"
category: workflow
tags: [debug, release, msvc, ub, asan, optimization, volatile, assert]
difficulty: advanced
---

Debug에서만 재현되거나 Release에서만 터지는 버그를 조사할 때의 단계별 절차.

## 언제 이 아티클을 보나

- Debug 빌드는 정상 동작하는데 Release 빌드에서 잘못된 값이 나올 때
- Release 빌드에서 크래시가 나는데 Debug 빌드에서는 재현이 안 될 때
- `/O2` 최적화 이후 새로 발생한 버그를 추적할 때

## 원인 분류 (빈도순)

```
원인 분류                | 빈도 | 증상
------------------------|------|------
미정의 동작(UB)          | 높음 | Release에서만 잘못된 값
최적화로 날아간 변수      | 중간 | 항상 0 또는 쓰레기값
초기화 안 된 변수         | 중간 | Debug는 0으로 초기화, Release는 쓰레기
assert() 의존 사이드이펙트| 낮음 | Release에서 assert 제거 → 동작 없음
DEBUG_NEW / 메모리 검사   | 낮음 | Release에서 힙 손상 감지 안 됨
```

## 단계별 진단

### Step 1: UB 확인 — /fsanitize=address 빌드

```
Visual Studio → 프로젝트 속성 → C/C++ → Enable Address Sanitizer → Yes
Release 구성으로 빌드 → 실행 → ASan이 접근 위반 지점 출력
(단: ASan은 VS 2019 16.9+ 필요)
```

ASan이 접근 위반을 보고하면 → UB 원인. 보고하지 않으면 Step 2로.

### Step 2: 최적화 레벨 이분 탐색

```
Release 구성 → C/C++ → Optimization → /O1 로 낮춤
재현되면: /O2가 원인 → 해당 함수에 #pragma optimize("", off) 추가해 범위 좁히기
```

```cpp
// 특정 함수만 최적화 해제해 범위 좁히기
#pragma optimize("", off)
void SuspectFunction()
{
    // ...
}
#pragma optimize("", on)
```

`/O1`에서 재현 안 되면 → `/O2`가 UB를 다른 방식으로 표출하는 것. Step 1 재확인.

### Step 3: 초기화 안 된 변수 확인

Debug 힙은 `0xCDCDCDCD`로 초기화되어 NULL처럼 동작하는 경우가 있다. Release 힙은 임의값.

```bash
# 초기화값 없는 멤버 변수 검색
grep -n "m_[a-z][A-Za-z]*;" src/해당파일.h
```

```cpp
// ❌ 초기화 없음 — Release에서 임의값
class MyHandler
{
    int m_nCount;       // Debug: 0에 가까운 값, Release: 쓰레기
    bool m_bActive;     // Debug: false처럼 동작, Release: true처럼 동작
};

// ✅ 명시적 초기화
class MyHandler
{
    int m_nCount;
    bool m_bActive;

    MyHandler() : m_nCount(0), m_bActive(false) {}
};
```

### Step 4: assert 사이드이펙트 확인

```bash
grep -n "assert(" src/ --include="*.cpp" | grep -v "//"
```

```cpp
// ❌ assert 안에서 함수 호출 — Release에서 Init()이 실행되지 않음
assert(Init());

// ✅ 결과를 분리
bool bResult = Init();
assert(bResult);
// (또는 Release용 방어: if (!bResult) { ScLogError(...); return false; })
```

`assert(함수호출())` 패턴이 있으면 → Release에서 함수가 호출되지 않아 초기화 누락.

### Step 5: volatile 누락 확인 (다중 스레드 변수)

Release 최적화가 반복 읽기를 레지스터 캐시로 대체해 다른 스레드의 변경을 감지 못한다.

```cpp
// ❌ 최적화로 레지스터 캐시 → 렌더 스레드 변경 무시 가능
bool m_bRunning;

// ✅ 스레드 간 공유 변수는 atomic으로
#include <atomic>
std::atomic<bool> m_bRunning;

// 또는 ScCommon::Mutex로 보호
ScCommon::ScopedLock lock(m_mutex);
bool bRunning = m_bRunning;
```

## 빠른 진단 결정 트리

```
Release에서만 버그 발생
  ├─ ASan이 접근 위반 보고함 → UB (Step 1)
  ├─ /O1에서 재현 안 됨     → /O2 최적화 UB (Step 2)
  ├─ 멤버 초기화값 없음      → 초기화 누락 (Step 3)
  ├─ assert(함수()) 패턴     → assert 사이드이펙트 (Step 4)
  └─ 다중 스레드 변수         → volatile/atomic 누락 (Step 5)
```

## Key Points

- Debug 힙이 `0xCDCDCDCD`로 초기화하는 특성이 초기화 버그를 숨긴다 — Release에서 처음 드러난다.
- `assert(함수호출())` 패턴은 NDEBUG 정의 시 함수 호출 자체가 사라지므로 생성자·초기화 함수를 절대 assert 안에 넣지 않는다.
- ASan(`/fsanitize=address`)은 Release 구성에서도 빌드 가능하며 UB 위치를 정확히 출력한다.
- `#pragma optimize("", off/on)`으로 함수 단위로 최적화를 끄면 /O2 문제 범위를 빠르게 좁힐 수 있다.
- 스레드 간 공유 변수는 `std::atomic` 또는 `ScCommon::ScopedLock`으로 보호해야 Release 최적화의 레지스터 캐싱을 막을 수 있다.
