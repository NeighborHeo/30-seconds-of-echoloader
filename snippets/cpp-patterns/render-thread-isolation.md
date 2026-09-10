---
title: "렌더 스레드 격리: 블로킹 작업 오프로드"
category: cpp-patterns
tags: [render-thread, threading, async, double-buffer, ai-inference]
difficulty: advanced
---

OVObject::Render()는 GC 렌더 스레드에서 실행된다. 파일 I/O, 네트워크, 무거운 연산 한 번이 전체 OVObject의 프레임을 멈춘다.

## Why

GC 렌더 스레드는 모든 OVObject를 순차 호출한다. `ScLogsDatabase` 쓰기(DB = 디스크 I/O), AI 추론, 파라미터 파일 로드 등 블로킹 작업을 Render() 안에서 호출하면 그 시간만큼 전체 렌더 파이프라인이 정지한다. 패턴: **비동기로 킥오프 → atomic/더블버퍼로 결과를 Render()에 전달.**

## Pattern

```cpp
// GE naming conventions:
// PascalCase classes, m_camelCase members, PascalCase methods
// ScCommon::ScopedLock (not std::lock_guard)
// ScCommon::Mutex (not std::mutex)
// NULL not nullptr
// No auto for non-trivial types

// ---- 패턴 1: 비동기 AI 추론 (SetParameter → Render) ----
class AiOvObject : public OVObject
{
public:
    AiOvObject() : m_bResultReady(false), m_fConfidence(0.0f) {}

    // SetParameter()에서 추론 킥오프 — Render()가 아님
    void SetParameter(const GCParam& param) override
    {
        if (param.GetKey() == m_strTriggerKey)
        {
            // 별도 스레드에서 AI 추론 실행
            ScCommon::Thread::Run([this]()
            {
                float fResult = RunAiInference(GetCurrentFrame());
                // atomic swap으로 결과 게시
                m_fConfidence.store(fResult, std::memory_order_release);
                m_bResultReady.store(true, std::memory_order_release);
            });
        }
    }

    // Render()는 atomic에서 읽기만 — 블로킹 없음
    void Render(ScCommon::IRenderContext* pCtx) override
    {
        if (m_bResultReady.load(std::memory_order_acquire))
        {
            float fConf = m_fConfidence.load(std::memory_order_acquire);
            DrawConfidenceOverlay(pCtx, fConf);
        }
    }

private:
    std::atomic<bool>  m_bResultReady;
    std::atomic<float> m_fConfidence;
    std::string        m_strTriggerKey; // OnActivate()에서 캐시
};

// ---- 패턴 2: 더블 버퍼 (컴퓨트 쓰기 / 렌더 읽기) ----
struct AnalysisResult
{
    float fValues[256]; // 계산 결과
    int   nCount;
};

class DoubleBufferedOvObject : public OVObject
{
public:
    DoubleBufferedOvObject()
        : m_pFrontBuffer(&m_Buffers[0])
        , m_pBackBuffer(&m_Buffers[1])
    {}

    void OnNewData(const RawData& data)
    {
        // 컴퓨트 스레드: 백버퍼에 씀
        ComputeAnalysis(data, m_pBackBuffer);

        // 포인터 atomic swap — 렌더 스레드는 항상 완성된 프론트를 읽음
        AnalysisResult* pTemp = m_pFrontBuffer.exchange(
            m_pBackBuffer, std::memory_order_acq_rel);
        m_pBackBuffer = pTemp;
    }

    void Render(ScCommon::IRenderContext* pCtx) override
    {
        // 블로킹 없음 — 항상 가장 최근의 완성된 버퍼를 읽음
        AnalysisResult* pFront = m_pFrontBuffer.load(std::memory_order_acquire);
        DrawAnalysis(pCtx, pFront);
    }

private:
    AnalysisResult          m_Buffers[2];
    std::atomic<AnalysisResult*> m_pFrontBuffer;
    AnalysisResult*         m_pBackBuffer; // 컴퓨트 스레드만 접근
};

// ---- 안티패턴: Render()에서 DB 쓰기 (절대 금지) ----
void Render_Bad(ScCommon::IRenderContext* pCtx)
{
    // ScLogsDatabase::Write() = 디스크 I/O = 프레임 드롭
    // ScLogsDatabase::Write("event", GetCurrentFrame());  // 금지!
}
```

## Key Points

- Render()에서 금지: 파일/DB I/O(`ScLogsDatabase`), 네트워크, 무거운 연산, Mutex 경합
- 비동기 킥오프는 `SetParameter()` 또는 취득 콜백에서, 결과 소비는 Render()에서 atomic으로만 한다
- 더블 버퍼 패턴: 컴퓨트가 백버퍼에 쓴 뒤 `atomic::exchange`로 포인터를 교체하면 Render()는 항상 완전한 데이터를 읽는다
- `std::atomic<float>`는 MSVC x86/x64에서 lock-free가 보장된다(`is_lock_free()` 확인 권장)
- AI 추론처럼 수십~수백ms가 걸리는 작업은 반드시 별도 스레드로 분리하고 결과만 atomic으로 게시한다
