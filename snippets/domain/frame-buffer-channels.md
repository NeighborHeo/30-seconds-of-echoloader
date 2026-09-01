---
title: "Frame Buffer Channels: SWE vs UGAP"
category: domain
tags: [frame-buffer, swe, ugap, acquisition, domain-difference]
difficulty: intermediate
---

SWE는 단일 프레임 버퍼(`m_currentFrame_BS`), UGAP는 2채널 버퍼(`m_currentFrame_BS` + `m_currentFrame_BS_UGAP`)를 사용한다. 채널 수가 다른 것 자체가 도메인 차이를 반영한다.

## Structure

```cpp
// SWE — 단일 채널: 전단파(shear wave) 데이터만
class SWEAcquisitionAssistant : public AcquisitionAssistantBase
{
    Frame m_currentFrame_BS;   // B-mode + Shear wave 합성 프레임
    // SWEROIProcessor 가 이 프레임에서 stiffness map 추출
};

// UGAP — 2채널: B-mode 기반 + UGAP 전용 감쇠 데이터
class UGAPAcquisitionAssistant : public AcquisitionAssistantBase
{
    Frame m_currentFrame_BS;         // 채널 1: 표준 B-mode 기반
    Frame m_currentFrame_BS_UGAP;    // 채널 2: UGAP 감쇠 계수 전용

    // ACResults 가 두 채널을 모두 읽어 감쇠(dB/cm/MHz) 계산
};
```

## 왜 2채널인가?

```
SWE 측정 원리:
  Push pulse → 전단파 발생 → 속도 추적 → stiffness
  (단일 물리량: 전단파 속도)

UGAP 측정 원리:
  참조 주파수 f1 반향 vs. 고주파 f2 반향 → 감쇠율(dB/cm/MHz) 계산
  (두 주파수 채널의 비교가 필요)
```

## Key Points

- `m_currentFrame_BS_UGAP` 는 UGAP 전용 — SWE 클래스에 추가하면 안 됨
- ROI 결과 타입도 다름: `SWEROIProcessor::ProcessResult` (탄성) vs `ACResults` (감쇠)
- 두 채널이 동기화되지 않으면 UGAP 측정값이 오염됨 — 2D/CM 뷰어 메타데이터 이전 후 검증 필수
- 이 채널 차이가 **SWE와 UGAP를 절대 하나의 클래스로 합칠 수 없는** 핵심 이유 중 하나
