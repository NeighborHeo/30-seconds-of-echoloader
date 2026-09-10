---
title: "gipc-app 실패 모드 지도: 무엇이 왜 깨지는가"
category: architecture
tags: [debugging, failure-modes, crash, av, threading, build, gc-params]
difficulty: advanced
---

gipc-app에서 반복해서 나타나는 실패 모드는 6가지 패턴으로 수렴한다. 증상만 보면 원인이 제각각처럼 보이지만, 각각 뿌리 원인이 명확하다.

## Why / 왜 알아야 하나

26년 코드베이스에서 버그를 디버깅할 때 가장 큰 시간 낭비는 증상에서 원인으로 거슬러 올라가는 과정이다. 이 문서는 그 역방향 매핑을 제공한다. 증상을 보면 어디를 먼저 확인해야 하는지 알 수 있다.

## 실패 모드 상세

### 1. IsValid() 누락 → Access Violation

**언제 발생하나:** 한 OVObject의 `OnDeactivate()` 도중 다른 OVObject가 들고 있던 `GcUdtHandle`이 무효화된다. 핸들을 들고 있던 쪽이 `IsValid()` 확인 없이 역참조하면 AV가 발생한다.

```cpp
// 잘못된 패턴 — OnDeactivate 경쟁에서 크래시
void SWEAcquisitionAssistant::Render()
{
    m_roiHandle.DoSomething();  // ← handle이 이미 무효화되었을 수 있음
}

// 올바른 패턴
void SWEAcquisitionAssistant::Render()
{
    if (!m_roiHandle.IsValid())
        return;
    m_roiHandle.DoSomething();
}
```

**증상:** 모드 전환 직후 간헐적 AV 크래시. 항상 재현되지 않아 "그냥 타이밍 문제"로 오인된다.

**진단:** WinDbg에서 크래시 주소가 `GcUdtHandle::operator->` 또는 역참조 위치를 가리킨다. 콜 스택에 `OnDeactivate` 또는 모드 전환 경로가 보인다.

**수정:** 모든 핸들 사용 전 `if (!handle.IsValid()) return;`. 이 패턴은 코드베이스 전반의 OVObject 구현에서 표준이다.

---

### 2. 게이팅 술어 누락 → 잘못된 모드 동작

**언제 발생하나:** `ESMain.cpp`에 새 호출 사이트를 추가할 때 기존 11개 사이트가 모두 가지고 있는 게이팅 조건(`InElasto()` 또는 `IsInUGAPMode()`)을 빠뜨린다.

```cpp
// ESMain.cpp — 기존 올바른 패턴 (11개 사이트 모두 이 형태)
if (InElasto())
    AcquisitionAssistant::Manager::Instance(HandlerType::SWE)
        .UpdateAcqAssistState(...);

if (IsInUGAPMode())
    AcquisitionAssistant::Manager::Instance(HandlerType::UGAP)
        .UpdateAcqAssistState(...);

// 잘못된 패턴 — 게이팅 없이 직접 호출
AcquisitionAssistant::Manager::Instance(HandlerType::SWE)
    .UpdateAcqAssistState(...);  // ← SWE 아닌 모드에서도 실행됨
```

**증상:** SWE가 아닌 스캔 모드에서 SWE 로직이 실행되어 임상 데이터가 오염된다. 개발 환경 디버거로는 발견이 어렵고, 임상 QA에서 뒤늦게 발견된다.

**진단:** `ESMain.cpp`에서 해당 호출 사이트 전후로 `InElasto()` / `IsInUGAPMode()` 조건이 있는지 확인한다. SWE와 UGAP의 게이팅 술어는 반드시 다르다 — 같다면 잘못된 것이다.

**수정:** `ESMain.cpp`에 새 호출 사이트를 추가할 때는 기존 11개 패턴을 정확히 따른다. 게이팅 없는 호출은 코드리뷰에서 즉시 차단한다.

---

### 3. 레이어 역방향 #include → 빌드 순환

**언제 발생하나:** Layer B(`GcViewer`)에서 Layer A(`EchoScanner`) 헤더를 직접 포함하거나, Layer A에서 Layer B 헤더를 직접 포함할 때 발생한다.

```
// 위반 예시 — GcViewer 코드에서
#include "../../EchoScanner/AcquisitionAssistant/AcquisitionAssistant.h"
//                          ^^^^^^^^^^^^
//                          Layer A 헤더를 Layer B가 직접 포함
```

**증상:** 링커 에러 LNK2005(심볼 중복 정의) 또는 MSBuild 순환 빌드 실패. CI가 없으므로 팀 전체 빌드 환경이 깨진 채 시간이 낭비된다.

**진단:**
```
grep -r "include.*EchoScanner" src/GcViewer/
grep -r "include.*GcViewer"   src/packages/EchoScanner/
```

**수정:** 레이어 간 통신은 GC 파라미터 버스로만 한다. 직접 포함이 필요하다고 느껴진다면 파라미터 키로 표현할 방법을 먼저 찾는다.

---

### 4. GC 파라미터 키 오타 → 사일런트 드롭

**언제 발생하나:** `SetParameterValue()` 호출에서 문자열 키를 오타냈을 때. 버스는 알 수 없는 키를 조용히 무시한다. 에러 로그도 없다.

```cpp
// Layer A — 오타 발생
ESMain::SetParameterValue("SWEAcqAssist.Stat",   // ← "State" 아니라 "Stat"
    static_cast<int>(AcqAssistState::Freeze));

// Layer B — 이 키는 절대 수신되지 않음. 무반응.
void SWEAcquisitionAssistant::SetParameter(
    const std::string& key, const Variant& value) override
{
    if (key == "SWEAcqAssist.State")  // ← "Stat"과 매칭 안 됨
        m_currentState = ...;
}
```

**증상:** OVObject가 아무 반응을 하지 않는다. 로그에도 에러가 없다. 기능이 그냥 동작하지 않는다.

**진단:** `SetParameterValue` 호출 측 로그와 `SetParameter` 수신 측 로그를 비교한다. 발행은 되었는데 수신이 없다면 키 불일치다.

**수정:** 문자열 리터럴을 `constexpr const char*` 상수로 정의하고 공유한다. 오타를 컴파일 타임에 잡는다:

```cpp
namespace GcKeys {
    constexpr const char* SWEAcqAssistState = "SWEAcqAssist.State";
}
// 이제 "SWEAcqAssist.Stat" 오타는 컴파일 에러로 드러남
```

---

### 5. 렌더 스레드에서 블로킹 → 프레임 드롭

**언제 발생하나:** `Render()` 내에서 파일 I/O(`ScLogsDatabase` 쓰기), 네트워크 호출, 또는 `ScCommon::Mutex` 장기 대기가 발생할 때. 렌더 루프는 프레임 예산(약 16ms at 60fps) 안에 완료되어야 한다.

```cpp
// 잘못된 패턴 — Render()에서 블로킹 작업
void SWEAcquisitionAssistant::Render() override
{
    // ...렌더링...
    ScLogsDatabase::Write(m_logData);  // ← 디스크 쓰기, 수십 ms 가능
}

// 올바른 패턴 — 작업을 별도 스레드로 오프로드
void SWEAcquisitionAssistant::Render() override
{
    // ...렌더링...
    m_logThread.Post([data = m_logData] {
        ScLogsDatabase::Write(data);   // ← 렌더 스레드 바깥에서 실행
    });
}
```

**증상:** 임상 스캔 중 화면이 끊긴다. 60fps에서 30fps 또는 그 이하로 떨어진다. 환자가 있는 임상 환경에서 발생하면 진단 신뢰도에 영향을 준다.

**진단:** `ScCommon::Timer`로 `Render()` 실행 시간을 측정한다. 16ms를 초과하는 프레임을 로그로 남긴다:
```cpp
auto t0 = ScCommon::Timer::Now();
// ... Render 본문 ...
auto elapsed = ScCommon::Timer::ElapsedMs(t0);
if (elapsed > 16)
    ScLogInfo("RENDER_PERF", "Render exceeded budget: %dms", elapsed);
```

**수정:** 블로킹 작업은 `ScCommon::Thread::Run()` 또는 비동기 큐로 오프로드한다.

---

### 6. 정적 초기화 순서 위반 → 시작 크래시

**언제 발생하나:** 한 DLL의 정적 변수가 다른 DLL의 정적 변수에 의존할 때. C++ 표준은 다른 번역 단위(특히 다른 DLL) 사이의 정적 초기화 순서를 보장하지 않는다.

```cpp
// 위험한 패턴 — 다른 DLL의 정적 객체에 의존하는 정적 변수
// DLL A
static AcquisitionAssistant::Manager s_manager;  // ← DLL B의 s_factory에 의존

// DLL B
static SomeFactory s_factory;   // ← DLL A보다 늦게 초기화될 수 있음
// 결과: s_manager 초기화 시점에 s_factory가 아직 null → 크래시
```

**증상:** 애플리케이션이 `main()` 진입 전에 크래시한다. WinDbg에서 DLL 로드 시점에 크래시 위치가 보인다. 스택 트레이스에 `_CRT_INIT` 또는 DLL 진입점이 보인다.

**진단:** WinDbg에서 `sxe ld:<dllname>` 로 DLL 로드 시 중단점을 건다. 크래시가 DLL 초기화 코드에서 발생함을 확인한다.

**수정:** Construct-on-first-use 패턴으로 정적 초기화를 함수 호출 시점으로 지연한다:

```cpp
// 안전한 패턴 — 첫 호출 시 초기화, 순서 문제 없음
AcquisitionAssistant::Manager& AcquisitionAssistant::Manager::Instance()
{
    static Manager s_instance;  // ← 함수 첫 호출 시 초기화됨
    return s_instance;
}
```

## 빠른 진단 룩업 테이블

| 증상 | 먼저 확인할 것 | 진단 도구 |
|------|----------------|-----------|
| 모드 전환 시 간헐적 AV 크래시 | `IsValid()` 누락 | WinDbg, 콜스택에서 `GcUdtHandle` 역참조 확인 |
| 기능이 무반응, 로그 에러 없음 | 게이팅 술어 누락 OR 키 오타 | 호출 사이트 `InElasto()`/`IsInUGAPMode()` 확인, 키 로그 비교 |
| 링커 에러 LNK2005 또는 순환 빌드 | 역방향 `#include` | `grep -r "include.*EchoScanner" src/GcViewer/` |
| 발행했는데 수신 없는 파라미터 | GC 키 오타 | `SetParameterValue` ↔ `SetParameter` 로그 비교 |
| 임상 스캔 중 화면 끊김 | `Render()`에서 블로킹 작업 | `ScCommon::Timer`로 렌더 프레임 시간 측정 |
| 시작 시 `main()` 전 크래시 | SIOF — 정적 초기화 순서 위반 | WinDbg `sxe ld:`, construct-on-first-use 적용 |

## Key Points

- `GcUdtHandle`을 역참조하기 전 `IsValid()` 확인은 선택이 아니라 **표준 패턴**이다 — OVObject 코드 전반에서 볼 수 있다.
- `ESMain.cpp` 새 호출 사이트는 반드시 기존 11개 사이트의 게이팅 술어(`InElasto()` / `IsInUGAPMode()`)를 그대로 따른다 — 없으면 임상 데이터 오염이다.
- GC 파라미터 키 오타는 CI나 컴파일러가 잡아주지 않는다 — `constexpr const char*` 상수화가 유일한 방어선이다.
- `Render()`는 16ms 예산을 가진다 — 그 안에서 블로킹 I/O나 뮤텍스 대기는 금지다.
- 정적 싱글톤은 전역 변수보다 construct-on-first-use 패턴이 안전하다 — `Manager::Instance()`가 그 예시다.
