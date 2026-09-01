---
title: "RAII 셧다운 순서: 초기화 역순 해제"
category: architecture
tags: [raii, shutdown, lifecycle, initialization, ovobject]
difficulty: intermediate
---

gipc-app의 패키지는 초기화 역순으로 해제된다. 순서가 틀리면 하위 레이어가 먼저 해제되어 댕글링 포인터가 발생한다.

## 초기화 / 셧다운 순서

```
초기화 (부팅):                     셧다운 (종료, 역순):
  1. EchoLoader       →              5. EchoLoader (마지막)
  2. EchoRoot         →              4. EchoRoot
  3. EchoConfig       →              3. EchoConfig
  4. EchoScanner      →              2. EchoScanner
  5. GcViewer/ObjectViewer  →        1. GcViewer/ObjectViewer (첫 번째)

이유: GcViewer가 EchoScanner에 의존
     → GcViewer 먼저 해제해야 EchoScanner 해제 안전
```

## OVObject 내부 해제 순서

```cpp
class SWEAcquisitionAssistant : public OVObject
{
public:
    // 프레임워크가 OnDeactivate() 먼저 호출 — 멤버 소멸자보다 먼저
    void OnDeactivate() override
    {
        // 1. 외부 UDT 참조 핸들 먼저 해제
        m_liverAiHandle.Release();
        m_roiHandle.Release();

        // 2. 렌더링 리소스 해제
        m_pRoiOverlay.Reset();
        m_pWarningTexture.Reset();

        // 3. 이후 멤버 소멸자가 선언 역순으로 자동 실행
    }

private:
    // 선언 순서 = 초기화 순서 = 소멸 역순
    Frame                          m_currentFrame_BS;     // 소멸 3번째
    SWEROIProcessor                m_roiProcessor;        // 소멸 2번째
    LiverWarning3Panel             m_warningPanel;        // 소멸 1번째 (역순)
    GcUdtHandle<LiverAIProcessor>  m_liverAiHandle;
    OVGraphics::Pointer<OVGraphics::Overlay> m_pRoiOverlay;
};
```

## 크래시 핸들러는 순서 밖

```cpp
// FatalErrorHandler — 정상 셧다운 순서와 무관하게 즉시 실행
// 이 안에서 멤버 변수 접근 금지 — 이미 소멸됐을 수 있음
void FatalErrorHandler(const char* module, const char* reason)
{
    ScLogError("FATAL", "[%s] %s — forcing shutdown", module, reason);
    std::terminate();
}
// RBUG159787: HW 부팅 후 재부팅 카운터 클리어
// → 정상 셧다운 완료 시점에만 수행 (FatalErrorHandler 안에서 금지)
```

## Key Points

- OVObject::OnDeactivate()는 멤버 소멸자보다 먼저 실행 — 핸들 해제 적기
- C++ 멤버 소멸 순서는 **선언 역순** — 헤더 선언 위치가 소멸 순서를 결정
- GcViewer(Layer B)는 EchoScanner(Layer A)보다 항상 먼저 해제
- 새 멤버 추가 시 의존 방향과 선언 위치를 함께 고려할 것
- 크래시 핸들러에서 멤버 변수 접근 금지 — 소멸 상태 불확정
