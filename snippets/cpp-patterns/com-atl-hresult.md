---
title: "COM/ATL HRESULT 에러 처리 패턴"
category: cpp-patterns
tags: [COM, ATL, HRESULT, error-handling]
difficulty: intermediate
---

COM 인터페이스 경계에서만 HRESULT를 사용하고, 내부 코드는 bool로 처리한다.

## Why

COM 표준은 인터페이스 메서드가 HRESULT를 반환하도록 강제한다. 그러나 내부 구현 로직까지 HRESULT로 오염시키면 코드가 복잡해지고 예외-안전 추론이 어려워진다. gipc-app은 `PackageComWrap`이 COM 경계를 담당하고, 그 안쪽은 bool+out-param 방식을 쓴다.

## Pattern

```cpp
// ATL COM 클래스 선언 — 아파트 스레딩 모델 명시
[
    coclass,
    threading(apartment),   // _ATL_APARTMENT_THREADED
    ...
]
class ATL_NO_VTABLE CMyComObject : public IMyInterface {
public:
    // COM 인터페이스 경계: HRESULT 반환 필수
    STDMETHODIMP DoWork(BSTR input, VARIANT* pResult) {
        bool ok = InternalDoWork(input, pResult);  // 내부는 bool
        return ok ? S_OK : E_FAIL;
    }

private:
    // 내부 구현: bool + out-param 패턴
    bool InternalDoWork(BSTR input, VARIANT* pResult);
};

// 호출 측 — SUCCEEDED/FAILED 매크로 사용
HRESULT hr = pObj->DoWork(bstrInput, &varResult);
if (FAILED(hr)) {
    // 에러 처리: HRESULT를 직접 전파하거나 로그
    return hr;  // COM 경계 안에서는 hr 전파
}

// _com_error 래핑 — 예외가 허용된 맥락에서만
try {
    HRESULT hr2 = pObj->AnotherMethod();
    if (FAILED(hr2)) _com_issue_error(hr2);
} catch (_com_error& e) {
    LogError(e.ErrorMessage());
    return E_UNEXPECTED;
}

// PackageComWrap 패턴: COM 래퍼가 경계 역할
class PackageComWrap {
public:
    // 외부(COM): HRESULT 노출
    HRESULT Initialize(BSTR config);

    // 내부: bool 위임
private:
    bool InitializeImpl(const std::wstring& config);
};
```

## Key Points

- `_ATL_APARTMENT_THREADED`는 COM 클래스 선언부에 명시한다 — STA 스레딩 모델
- `SUCCEEDED(hr)` / `FAILED(hr)` 매크로만 사용. `hr == S_OK` 직접 비교 금지 (S_FALSE 등 무시하는 버그)
- COM 인터페이스 경계에서만 HRESULT — 내부 헬퍼는 `bool` + out-param
- `_com_error` 예외는 예외가 허용된 맥락에서만 사용. COM 콜백 안에서 예외가 경계를 넘으면 UB
- `PackageComWrap`이 COM 경계 역할. 래퍼 밖에서 raw COM 포인터를 다루지 않는다
- `E_FAIL`은 최후 수단. 가능하면 의미 있는 HRESULT(`E_INVALIDARG`, `E_OUTOFMEMORY` 등) 반환
