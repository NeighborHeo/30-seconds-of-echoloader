---
title: "UDT 객체 vs GC 파라미터: 언제 어느 채널을 쓰나"
category: udt
tags: [udt, gc-params, communication, layer-boundary, design]
difficulty: advanced
---

gipc-app에는 두 가지 통신 채널이 공존한다. **언제 어느 쪽을 써야 하는가**가 Layer A/B 구분보다 더 중요한 판단이다.

## 두 채널 비교

```
                  GC 파라미터 버스              UDT 객체 직접 접근
                  ─────────────────────         ────────────────────────
형태              SetParameterValue(key, val)   handle->Method()
통신 방향         단방향 (주로 A→B)             양방향
상태 보유         없음 (stateless 메시지)        있음 (객체 멤버 변수)
타입 안전성       문자열 키 → 런타임 파싱        C++ 타입 → 컴파일 타임
레이어 제약       레이어 경계 넘기 가능          같은 레이어 내부에서만
오타 결과         묵음 실패 (무시됨)             컴파일 에러
성능              직렬화/파싱 오버헤드           직접 함수 호출
```

## 올바른 선택 기준

```cpp
// ✅ GC 파라미터 버스: 레이어 A → 레이어 B (경계 넘기)
// EchoScanner(Layer A)에서 GcViewer(Layer B)로 상태 전달
ESMain::SetParameterValue("SWEAcqAssist.RoiX", roiX);
ESMain::SetParameterValue("SWEAcqAssist.State",
    static_cast<int>(AcqAssistState::Freeze));

// ✅ UDT 핸들: 같은 레이어 내 직접 호출
// Layer B 내부 — SWEAcquisitionAssistant가 ROIProcessor 호출
if (m_roiHandle.IsValid())
    m_roiHandle->ProcessFrame(m_currentFrame_BS);

// ✅ GC 파라미터 버스: 느슨한 결합 (수신자가 누군지 모를 때)
// 브로드캐스트성 상태 변경
ESMain::SetParameterValue("GlobalFreeze", true);

// ✅ UDT 핸들: 타입이 보장된 1:1 직접 조작
// 특정 컴포넌트를 직접 제어할 때
m_liverAiHandle->SetInputFrame(frame);
result = m_liverAiHandle->RunInference();
```

## 혼용 실수 사례

```cpp
// ❌ 잘못된 패턴: Layer A에서 UDT 핸들로 Layer B 직접 접근
// AcquisitionAssistant.cpp(Layer A)에서 GcViewer UDT에 핸들을 보유
// → 레이어 경계 위반, 순환 의존 발생
class SWEAcqAssistHandler {  // Layer A
    GcUdtHandle<SWEAcquisitionAssistant> m_viewerHandle;  // ❌ Layer B 객체
};

// ❌ 잘못된 패턴: 파라미터 버스로 고빈도 데이터 스트림 전달
// 프레임마다 픽셀 데이터를 GC 파라미터로 넘기면 직렬화 오버헤드 폭발
for (auto& pixel : frameData) {
    ESMain::SetParameterValue("Pixel", pixel);  // ❌ 프레임 단위 고빈도
}
```

## Key Points

- **레이어 경계** = GC 파라미터. **같은 레이어 내** = UDT 핸들
- 파라미터 키 오타는 컴파일 타임에 잡히지 않음 → 상수 정의 또는 집중 관리 파일 권장
- 고빈도 데이터(프레임 버퍼, 실시간 메트릭)는 파라미터 버스 부적합 — 공유 메모리나 콜백 고려
- `AutoPos` 키(`"SWEAcqAssistToggleContinuous"`)는 파라미터처럼 보이지만 **명령(command)** 의미 — 상태가 아닌 이벤트를 파라미터로 보내는 특수 패턴
- 두 채널 모두 **외부 컨트랙트** — 키 이름이나 UDT 인터페이스 변경 시 Director sign-off 필수
