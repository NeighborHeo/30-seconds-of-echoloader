---
title: "COM 인터페이스 패턴: IUnknown, Apartment, HRESULT"
category: cpp-patterns
tags: [com, atl, hresult, apartment, iunknown, threading]
difficulty: advanced
---

gipc-app의 COM 컴포넌트는 주로 ATL 기반이며 Apartment 스레딩 모델을 사용한다. GC 파라미터 버스의 바깥 경계(원격 스트리밍 등)에서 나타난다.

## 핵심 COM 패턴

```cpp
// ATL COM 클래스 선언 (Apartment 스레딩)
[
    coclass,
    threading("apartment"),   // _ATL_APARTMENT_THREADED
    uuid("...")
]
class ATL_NO_VTABLE CAcqAssistServer
    : public CComObjectRootEx<CComSingleThreadModel>
    , public CComCoClass<CAcqAssistServer, &CLSID_AcqAssistServer>
    , public IAcqAssistServer          // 자체 인터페이스
    , public ISupportErrorInfo         // COM 에러 전파
{
public:
    DECLARE_REGISTRY_RESOURCEID(IDR_ACQASSISTSERVER)
    DECLARE_NOT_AGGREGATABLE(CAcqAssistServer)
    BEGIN_COM_MAP(CAcqAssistServer)
        COM_INTERFACE_ENTRY(IAcqAssistServer)
        COM_INTERFACE_ENTRY(ISupportErrorInfo)
    END_COM_MAP()
};
```

## HRESULT 처리 (COM 경계)

```cpp
// COM 메서드는 bool 대신 HRESULT 반환
HRESULT CAcqAssistServer::AuthenticateUser(BSTR userName, VARIANT_BOOL* pResult)
{
    if (userName == nullptr || pResult == nullptr)
        return E_POINTER;

    std::string name = _bstr_t(userName);
    std::string errMsg;

    // 내부 로직은 bool+out-param 패턴 유지
    if (!m_authManager.Authenticate(name, errMsg))
    {
        ScLogError("STREAMING", "Auth failed: %s", errMsg.c_str());
        *pResult = VARIANT_FALSE;
        return S_OK;  // COM 메서드는 성공 반환 — 논리 실패는 out-param으로
    }

    *pResult = VARIANT_TRUE;
    return S_OK;
}

// COM 경계 밖에서 호출 (일반 C++ → COM)
CComPtr<IAcqAssistServer> pServer;
HRESULT hr = CoCreateInstance(CLSID_AcqAssistServer, nullptr,
                               CLSCTX_LOCAL_SERVER, IID_IAcqAssistServer,
                               reinterpret_cast<void**>(&pServer));
if (FAILED(hr))
{
    ScLogError("COM", "CoCreateInstance 실패: 0x%08X", hr);
    return false;
}
```

## Apartment 스레딩 규칙

```cpp
// Apartment = COM 객체가 생성된 스레드에서만 직접 호출 가능
// 다른 스레드에서 호출 → COM 런타임이 마샬링(marshaling) 처리

// 올바른 패턴: UI 스레드에서 COM 초기화
HRESULT hr = CoInitializeEx(nullptr, COINIT_APARTMENTTHREADED);
// GEOComm::InitFromMainUIThread() 참고 — 이 이유로 UI 친화도 있음

// 잘못된 패턴: 백그라운드 스레드에서 Apartment COM 객체 직접 호출
std::thread([&]() {
    pServer->DoWork();  // ❌ 마샬링 없이 다른 Apartment에서 호출
}).join();

// 올바른 패턴: 마샬링 또는 GIT(Global Interface Table) 사용
IGlobalInterfaceTable* pGIT = nullptr;
CoCreateInstance(CLSID_StdGlobalInterfaceTable, ...);
// 스레드 간 인터페이스 공유
```

## CComPtr vs 원시 포인터

```cpp
// ✅ CComPtr<T>: RAII, AddRef/Release 자동
CComPtr<IAcqAssistServer> pServer;  // 소멸 시 Release() 자동

// ❌ 원시 COM 포인터: 신규 코드에서 금지
IAcqAssistServer* pServer = nullptr;  // Release() 수동, 누출 위험
// RBUG123456: 레거시 코드 — 수정 시 CComPtr 교체 대상

// ✅ _bstr_t / _variant_t: BSTR/VARIANT RAII
_bstr_t bstrName(L"UserName");  // SysFreeString 자동
_variant_t vtValue(42);          // VariantClear 자동
```

## Key Points

- COM 경계(원격 스트리밍, 외부 플러그인)에서만 COM 사용 — 내부 로직은 일반 C++
- `FAILED(hr)` / `SUCCEEDED(hr)` 매크로 사용 — 숫자 직접 비교 금지 (`hr != S_OK` 는 틀림)
- COM 메서드에서 예외 던지기 금지 — `HRESULT` 로 에러를 전달하고 내부에서 `ScLogError`
- `_ATL_APARTMENT_THREADED` 컴포넌트는 반드시 같은 스레드(주로 UI 스레드)에서 접근
- `CComPtr` 은 COM 경계에서의 `std::unique_ptr` — 신규 COM 코드에서 항상 사용
