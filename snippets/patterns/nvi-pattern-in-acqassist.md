---
title: "NVI 패턴: AcquisitionAssistantBase의 공개 비가상 인터페이스"
category: patterns
tags: [nvi, template-method, AcquisitionAssistantBase, SWE, UGAP, medical-device]
difficulty: intermediate
---

`OnFrame()`은 public non-virtual, `DoOnFrame()`은 private virtual — NVI(Non-Virtual Interface) 패턴으로 서브클래스가 pre/post 조건을 우회하지 못하게 강제한다.

## Why

의료기기 소프트웨어에서 public pure virtual 인터페이스의 문제: 서브클래스가 로깅, 모드 검사, 타이밍 계측을 건너뛸 수 있다. 한 서브클래스가 조건을 누락하면 필드 장비에서 재현 불가능한 오동작이 된다. NVI는 "공통 외골격(skeleton)은 베이스가 소유, 변화 지점만 서브클래스에 위임" 원칙으로 이 위험을 제거한다.

## Pattern

```cpp
// === 베이스 클래스: NVI 외골격 ===
class AcquisitionAssistantBase
{
public:
    // public, non-virtual — 모든 호출자는 여기로 들어온다
    void OnFrame(const AcqFrame& frame)
    {
        // pre-condition: 공통 검사 (서브클래스가 우회 불가)
        if (!IsOperatingModeValid())
        {
            ScCommon::Logger::Warning(
                "AcquisitionAssistantBase::OnFrame - invalid mode, skipping");
            return;
        }

        ScCommon::ScopedTimer timer("OnFrame");  // 타이밍 계측

        // 서브클래스 로직 위임
        DoOnFrame(frame);

        // post-condition: 공통 후처리 (서브클래스가 우회 불가)
        m_nFrameCount++;
        NotifyFrameProcessed(frame);
    }

    // 마찬가지로 NVI: 서브클래스별 초기화 진입점
    void Initialize(const AcqConfig& config)
    {
        ValidateConfig(config);      // 공통 검증
        m_config = config;
        DoInitialize(config);        // 서브클래스 고유 초기화
        LogInitialized();            // 공통 후처리
    }

protected:
    // private virtual이 이상적이나, 서브클래스가 다시 위임해야 할 경우
    // protected virtual 허용 — 단, 직접 호출은 금지
    virtual void DoOnFrame(const AcqFrame& frame) = 0;
    virtual void DoInitialize(const AcqConfig& config) = 0;

private:
    bool IsOperatingModeValid() const;
    void NotifyFrameProcessed(const AcqFrame& frame);
    void ValidateConfig(const AcqConfig& config);
    void LogInitialized();

    int          m_nFrameCount;
    AcqConfig    m_config;
};

// === SWE 서브클래스: 변화 지점만 구현 ===
class SWEAcqAssist : public AcquisitionAssistantBase
{
protected:
    virtual void DoOnFrame(const AcqFrame& frame) OVERRIDE
    {
        // SWE 전용 로직만 — pre/post는 베이스가 보장
        ProcessSWEVelocityData(frame);
        UpdateSWEColorMap(frame);
    }

    virtual void DoInitialize(const AcqConfig& config) OVERRIDE
    {
        m_pSWEProcessor = new SWEProcessor(config.GetSWEParams());
    }

private:
    void ProcessSWEVelocityData(const AcqFrame& frame);
    void UpdateSWEColorMap(const AcqFrame& frame);

    SWEProcessor* m_pSWEProcessor;
};

// === UGAP 서브클래스: 동일 외골격, 다른 내부 ===
class UGAPAcqAssist : public AcquisitionAssistantBase
{
protected:
    virtual void DoOnFrame(const AcqFrame& frame) OVERRIDE
    {
        // UGAP 전용 로직 — 로깅/모드검사는 베이스가 이미 처리
        ProcessUGAPShearWave(frame);
    }

    virtual void DoInitialize(const AcqConfig& config) OVERRIDE
    {
        m_pUGAPEngine = new UGAPEngine(config.GetUGAPParams());
    }

private:
    void ProcessUGAPShearWave(const AcqFrame& frame);

    UGAPEngine* m_pUGAPEngine;
};

// === 호출 측 (ESMain) ===
// 호출자는 OnFrame()만 호출 — 내부가 SWE인지 UGAP인지 모른다
void ESMain::ProcessNewFrame(const AcqFrame& frame)
{
    m_pAcqAssist->OnFrame(frame);  // NVI 진입점
}
```

## Key Points

- **우회 불가능한 pre/post 조건**: `DoOnFrame()`이 `private`(또는 `protected`)이므로 외부 코드가 직접 호출하여 검사를 건너뛸 수 없다 — 의료기기 IEC 62304 트레이서빌리티 요구사항 대응.
- **공통 계측 단일화**: 로깅, 타이머, 프레임 카운터가 베이스에만 존재 — 서브클래스 추가 시 자동 적용, 누락 불가.
- **C++ `OVERRIDE` 키워드**: MSVC gipc-app 코드베이스에서 `override` 대신 매크로 `OVERRIDE` 사용 — 오타로 인한 새 가상 함수 실수 방지.
- **순수 가상 공개 인터페이스와의 차이**: `virtual void OnFrame(...) = 0;` 스타일이면 서브클래스가 로깅을 깜빡할 수 있다. NVI는 그 가능성을 문법적으로 차단한다.
- **테스트 용이성**: 테스트에서 `DoOnFrame()`만 mock으로 대체하면 pre/post 경로 모두 검증 가능 — 베이스 테스트와 서브클래스 테스트 명확히 분리된다.
