---
title: "RAII 패턴: ScopedLock, GcUdtHandle, 프레임 버퍼 소유권"
category: patterns
tags: [raii, ScopedLock, GcUdtHandle, mutex, ownership, ScCommon, resource-management]
difficulty: beginner
---

gipc-app에서 자원 해제는 소멸자에 맡긴다 — `ScCommon::ScopedLock`, `GcUdtHandle`, 프레임 버퍼 세 가지 RAII 패턴으로 raw `unlock()`/`delete` 호출을 제거한다.

## Why

GE 코딩 표준은 raw `mutex.Unlock()`, raw `delete`, raw 핸들 해제를 금지한다. 이유: 예외, 조기 `return`, 에러 분기에서 해제가 빠지면 데드락과 메모리 누수가 필드에서 재현하기 어려운 형태로 발생한다. RAII는 스코프 종료 시 소멸자가 반드시 호출된다는 C++ 보장을 이용해 이 위험을 문법적으로 차단한다.

## Pattern

### 패턴 1: ScCommon::ScopedLock (뮤텍스)

```cpp
// === Bad: raw unlock — 조기 return에서 데드락 ===
void AcqAssistManager::UpdateStateBad(EAcqState eNewState)
{
    m_mutex.Lock();

    if (eNewState == m_eCurrentState)
    {
        // 여기서 return하면 Unlock()을 건너뜀 → 데드락
        return;
    }

    m_eCurrentState = eNewState;
    m_mutex.Unlock();  // 모든 경로에서 직접 호출해야 함
}

// === Good: ScopedLock — 어느 경로로 나가도 해제 보장 ===
void AcqAssistManager::UpdateState(EAcqState eNewState)
{
    ScCommon::ScopedLock lock(m_mutex);  // 생성자에서 Lock()

    if (eNewState == m_eCurrentState)
        return;  // 소멸자에서 자동 Unlock() — 데드락 없음

    m_eCurrentState = eNewState;
    NotifyStateChanged(eNewState);
    // 함수 종료 시 lock 소멸자 → Unlock()
}

// 락 범위를 좁혀야 할 때: 중괄호로 스코프 명시
void AcqAssistManager::OnNewFrameWithNarrowLock(const AcqFrame& frame)
{
    EAcqState eStateCopy;
    {
        ScCommon::ScopedLock lock(m_mutex);
        eStateCopy = m_eCurrentState;  // 복사만 락 안에서
    }  // 락 해제

    // 오래 걸리는 처리는 락 밖에서
    ProcessFrameForState(frame, eStateCopy);
}
```

### 패턴 2: GcUdtHandle 자동 해제

```cpp
// === Bad: raw 핸들 — 예외 또는 조기 return 시 누수 ===
void ProcessUDTBad()
{
    GcUdtHandle hUdt = GcUdtOpen("SWE_Channel");
    if (hUdt == GC_INVALID_HANDLE)
        return;

    bool bSuccess = GcUdtSend(hUdt, GetFrameData());
    if (!bSuccess)
    {
        // GcUdtClose(hUdt) 잊으면 핸들 누수
        return;
    }

    GcUdtClose(hUdt);
}

// === Good: RAII 핸들 래퍼 ===
class ScopedUdtHandle
{
public:
    explicit ScopedUdtHandle(const char* pszChannel)
        : m_hUdt(GcUdtOpen(pszChannel))
    {}

    ~ScopedUdtHandle()
    {
        if (m_hUdt != GC_INVALID_HANDLE)
            GcUdtClose(m_hUdt);  // 항상 해제
    }

    bool IsValid() const { return m_hUdt != GC_INVALID_HANDLE; }
    GcUdtHandle Get()    { return m_hUdt; }

private:
    // 복사 금지 (핸들 이중 해제 방지)
    ScopedUdtHandle(const ScopedUdtHandle&);
    ScopedUdtHandle& operator=(const ScopedUdtHandle&);

    GcUdtHandle m_hUdt;
};

void ProcessUDT()
{
    ScopedUdtHandle udtHandle("SWE_Channel");
    if (!udtHandle.IsValid())
    {
        ScCommon::Logger::Error("Failed to open UDT channel");
        return;  // 소멸자에서 GcUdtClose 자동 호출 (no-op, invalid handle)
    }

    GcUdtSend(udtHandle.Get(), GetFrameData());
    // 함수 종료 시 핸들 자동 해제
}
```

### 패턴 3: 프레임 버퍼 소유권 (RAII + 이전)

```cpp
// 프레임 버퍼의 소유권을 명시적으로 표현하는 RAII 래퍼
class ScopedFrameBuffer
{
public:
    explicit ScopedFrameBuffer(int nWidth, int nHeight)
        : m_pBuffer(new unsigned char[nWidth * nHeight * 4])
        , m_nSize(nWidth * nHeight * 4)
    {}

    ~ScopedFrameBuffer()
    {
        delete[] m_pBuffer;
        m_pBuffer = NULL;
    }

    unsigned char* Get()      { return m_pBuffer; }
    int            Size() const { return m_nSize; }

    // 소유권 이전 (C++11 move 대신 명시적 release 패턴)
    unsigned char* Release()
    {
        unsigned char* pTemp = m_pBuffer;
        m_pBuffer = NULL;
        m_nSize = 0;
        return pTemp;  // 호출자가 delete[] 책임
    }

private:
    ScopedFrameBuffer(const ScopedFrameBuffer&);
    ScopedFrameBuffer& operator=(const ScopedFrameBuffer&);

    unsigned char* m_pBuffer;
    int            m_nSize;
};

void SWEAcqAssist::ProcessAndPublishFrame(int nWidth, int nHeight)
{
    ScopedFrameBuffer frameBuffer(nWidth, nHeight);

    FillSWEData(frameBuffer.Get(), frameBuffer.Size());

    // 처리 성공 시에만 소유권 이전, 실패 시 자동 해제
    if (ValidateFrame(frameBuffer.Get()))
    {
        unsigned char* pOwned = frameBuffer.Release();
        PublishFrame(pOwned);  // 수신 측이 delete[] 책임
    }
    // frameBuffer 소멸자: Release()가 호출됐으면 no-op, 아니면 해제
}
```

## Key Points

- **GE 코딩 표준**: raw `mutex.Unlock()`, raw `delete`/`delete[]`, raw 핸들 해제 직접 호출 금지 — 모든 자원 해제는 소멸자에 위임. 코드 리뷰에서 즉시 지적 항목.
- **`ScCommon::ScopedLock` vs `std::lock_guard`**: gipc-app은 `std::mutex`가 아닌 `ScCommon::Mutex`를 쓰므로 `std::lock_guard` 사용 불가. `ScCommon::ScopedLock`이 그 역할을 한다.
- **복사 생성자/대입 연산자 금지**: RAII 래퍼는 반드시 복사를 막아야 한다 (이중 해제 방지). C++11 `= delete`보다 MSVC 호환 방식인 `private` 선언으로 막는다.
- **`Release()` 패턴**: C++11 `std::move`/`unique_ptr`이 있으면 이상적이지만, 레거시 코드베이스에서는 명시적 `Release()` + 주석으로 소유권 이전을 표시하라. 소유권 이전 후 원본 래퍼의 포인터를 `NULL`로 설정하는 것 필수.
- **락 범위 최소화**: `ScopedLock`의 스코프가 함수 전체를 덮으면 경합이 늘어난다. 상태 읽기만 락 안에서 하고, 오래 걸리는 처리는 락 밖에서 하는 "복사 후 해제" 관용구를 활용하라.
