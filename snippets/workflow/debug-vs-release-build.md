---
title: "Debug vs Release 빌드 차이"
category: workflow
tags: [build, debug, release, MFC, MSVC]
difficulty: intermediate
---

`_DEBUG` / `NDEBUG` define 차이를 알고, `assert()` 도입 시 release 동작 변화를 반드시 확인한다.

## 절차

| 항목 | Debug | Release |
|------|-------|---------|
| `_DEBUG` define | O | X |
| `NDEBUG` define | X | O |
| `DEBUG_NEW` (MFC) | O | X |
| `RuntimeChecks /RTC1` | O (`EnableFastChecks`) | X |
| `assert()` | 발동 | 제거됨 |
| `/W4 /permissive-` | 양쪽 동일 | 양쪽 동일 |

- **`DEBUG_NEW`**: MFC `.cpp` 파일 상단의 `#ifdef _DEBUG / #define new DEBUG_NEW` 패턴으로 메모리 누수 위치 추적
- **`RuntimeChecks`**: `CppProjectBase.props`에서 Debug 전용으로 설정, Release에선 사라짐
- **production code에 `assert()` 거의 없는 이유**: Release 빌드에서 완전히 제거되므로 실제 오류 방어력 없음 — `if (!cond) return false;` 패턴 선호

## 예시

```cpp
// MFC Debug 메모리 추적 — .cpp 최상단
#ifdef _DEBUG
#define new DEBUG_NEW
#endif

// assert 대신 명시적 반환 (production 패턴)
bool MyClass::DoWork(Data* pData)
{
    if (!pData)          // Release에서도 동작
    {
        ScLogError("MYMOD", "DoWork: null pData");
        return false;
    }
    // ...
    return true;
}
```

## 체크리스트

- [ ] 새 `assert()` 도입 시 Release 동작 변화 검토
- [ ] MFC `DEBUG_NEW` 블록 `.cpp` 상단에 위치
- [ ] 실제 방어 로직은 `if (!cond) return false;` 패턴 사용
- [ ] `/W4 /permissive-` 양쪽 빌드 모두 통과
- [ ] `RuntimeChecks`는 `CppProjectBase.props` 에서만 관리
