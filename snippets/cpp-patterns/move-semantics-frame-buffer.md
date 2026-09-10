---
title: "이동 의미론으로 프레임 버퍼 소유권 전달"
category: cpp-patterns
tags: [move-semantics, frame-buffer, rvalue-reference, ownership, performance]
difficulty: intermediate
---

대형 픽셀 배열을 복사하면 10 ms 패널티가 발생한다. `std::move()`로 소유권을 이전한다.

## Why

초음파 영상은 하드웨어 콜백에서 픽셀 데이터를 수신한다.
이 데이터를 처리 큐로 넘길 때 복사하면 한 프레임(예: 1280×1024 × 4 bytes = ~5 MB)마다
`memcpy` 비용이 발생하고 프레임레이트가 깨진다.
C++11 이동 의미론으로 포인터 교환(O(1))으로 대체한다.

## Pattern

```cpp
// Frame.h — 픽셀 데이터를 소유하는 프레임 구조체
#pragma once
#include <vector>
#include <cstdint>

struct Frame {
    std::vector<uint8_t> m_pixels;  // ~5 MB, 복사 비용 큼
    int m_nWidth  = 0;
    int m_nHeight = 0;
    double m_dTimestamp = 0.0;

    // 이동 생성자/대입 — std::vector가 이미 이동을 지원하므로 default로 충분
    Frame()                      = default;
    Frame(Frame&&)               = default;
    Frame& operator=(Frame&&)    = default;

    // 복사는 명시적으로 삭제 — 실수 방지
    Frame(const Frame&)            = delete;
    Frame& operator=(const Frame&) = delete;
};
```

```cpp
// HardwareCallback.cpp — 하드웨어 → 처리 큐 전달
void AcquisitionDriver::OnFrameReceived(Frame& rawFrame)
{
    // std::move: rawFrame의 내용을 ProcessingQueue로 이전 (복사 없음)
    // 이동 후 rawFrame은 "moved-from" 상태: 유효하지만 비어 있음
    m_processingQueue.Push(std::move(rawFrame));

    // 이동 후 rawFrame을 재사용하려면 반드시 재초기화
    rawFrame.m_pixels.clear();
    rawFrame.m_nWidth  = 0;
    rawFrame.m_nHeight = 0;
}

// ProcessingQueue.cpp — 큐에서 꺼내 처리기로 전달
void ProcessingQueue::DispatchFrame()
{
    Frame frame = std::move(m_queue.front());  // 큐에서 소유권 이전
    m_queue.pop();

    // ImageProcessor도 이동으로 받음 — 체인 전체에서 복사 0회
    m_imageProcessor.Process(std::move(frame));
}

// ImageProcessor — 최종 소비자
void ImageProcessor::Process(Frame frame)  // 값으로 받아 소유
{
    // frame은 이 함수가 소유 — 소멸 시 자동 해제
    ApplyScanConversion(frame.m_pixels.data(), frame.m_nWidth, frame.m_nHeight);
}
```

## Key Points

- **C++11 주의**: 보장 복사 생략(guaranteed copy elision)은 C++17부터. C++11/14에서는 반환값 최적화(NRVO)가 적용 안 될 수 있으므로 이동 생성자를 명시적으로 선언해야 함
- **moved-from 상태 규칙**: 이동 후 객체는 "유효하지만 비정의 상태". `std::vector`는 이동 후 비어 있음이 보장되지만, 커스텀 타입은 명시적 정리 필요
- `m_queue.front()` → `std::move` 없이 `pop()` 하면 복사 발생. 반드시 `move` 후 `pop`
- 복사 생성자를 `= delete`로 막으면 실수로 복사하는 코드가 컴파일 오류가 되어 조기 발견 가능
- ScCommon::ScopedLock으로 큐 접근 보호 필수: 하드웨어 콜백과 처리 스레드는 서로 다른 스레드에서 실행됨
