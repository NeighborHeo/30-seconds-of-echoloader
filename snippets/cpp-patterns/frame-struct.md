---
title: "Frame 구조체 — 초음파 이미지 데이터 컨테이너"
category: cpp-patterns
tags: [frame, acquisition, image-data, swe, ugap, roi-processor]
difficulty: intermediate
---

`Frame`은 한 acquisition cycle의 원시 IQ/B-mode 데이터와 메타데이터를 담는 값 타입 구조체다.

## Why

초음파 처리 파이프라인(acquisition → ROI processing → rendering)은 매 프레임 데이터를 레이어 간에 전달한다.  
Frame은 픽셀 버퍼, 타임스탬프, 해상도, 깊이 정보를 하나로 묶어 단일 단위로 처리·동결(freeze)·복사할 수 있게 한다.  
SWE는 단일 채널, UGAP는 2채널 Frame을 사용하므로 멤버 구성이 다르다 — 두 모드를 같은 Frame으로 합칠 수 없다.

## Pattern

```cpp
// --- Frame 구조체 (개념적 스케치, 실제 멤버명은 코드베이스 참조) ---
struct Frame
{
    std::vector<uint8_t> pixelBuffer;   // B-mode raw byte array
    uint32_t             width;
    uint32_t             height;
    float                depth_mm;      // 영상 깊이 (mm)
    uint64_t             timestamp;     // acquisition timestamp (μs)
    int                  frameNumber;
};

// --- SWE: 단일 채널 ---
class SWEAcqAssistObject : public OVObject
{
    Frame m_currentFrame_BS;            // B-mode single channel
};

// --- UGAP: 2채널 ---
class UGAPAcqAssistObject : public OVObject
{
    Frame m_currentFrame_BS;            // B-mode channel
    Frame m_currentFrame_BS_UGAP;       // UGAP-specific channel
};

// --- 전달: 항상 const& ---
// Frame은 복사 가능하지만 pixelBuffer가 크므로 const& 로 전달
void ROIProcessor::Process(const Frame& frame, std::string& errorMsg)
{
    if (frame.pixelBuffer.empty())
    {
        errorMsg = "ROIProcessor: empty frame";
        return;
    }
    // ... 알고리즘 처리
    ScLogInfo("ROIProcessor", "Processing frame #%d (%ux%u, %.1fmm)",
              frame.frameNumber, frame.width, frame.height, frame.depth_mm);
}

// --- Freeze: 마지막 프레임 잠금 ---
void SWEAcqAssistObject::OnFreeze()
{
    // m_currentFrame_BS는 이미 마지막 수신 프레임 상태로 유지
    // Freeze 이후 새 Frame 수신 시 m_currentFrame_BS 갱신 금지
    m_isFrozen = true;
}

void SWEAcqAssistObject::OnNewFrame(const Frame& newFrame)
{
    if (m_isFrozen)
        return;                         // 동결 중엔 갱신 안 함
    m_currentFrame_BS = newFrame;       // 복사 — 의도적
    m_pRoiProcessor->Process(m_currentFrame_BS, m_lastError);
}
```

## Key Points

- Frame은 값 타입이므로 대입 시 pixelBuffer 전체가 복사된다 — 파이프라인 내에서는 `const Frame&`로 전달.
- SWE는 `m_currentFrame_BS` 단일 멤버, UGAP는 `m_currentFrame_BS` + `m_currentFrame_BS_UGAP` 2개 — 두 모드는 구조가 다르다.
- ROIProcessor는 `const Frame&`를 받아 처리 결과를 `ProcessResult`(또는 bool + errorMsg)로 반환 — Frame을 수정하지 않는다.
- Freeze 상태에서는 `m_currentFrame_BS` 갱신을 명시적으로 막아야 한다 — OnNewFrame 진입부에서 guard 처리.
- `depth_mm`은 렌더링 스케일 계산과 ROI 좌표 변환에 모두 쓰이므로 Frame에 반드시 포함되어야 한다.
