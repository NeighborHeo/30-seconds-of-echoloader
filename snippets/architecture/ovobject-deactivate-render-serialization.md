---
title: "OnDeactivate와 Render() 직렬화: IsValid()가 충분한 이유"
category: architecture
tags: [gc, ovobject, render, deactivate, thread-safety, IsValid, GcUdtHandle]
difficulty: advanced
---

GrandCentral 프레임워크가 `OnDeactivate()`와 `Render()`를 직렬화하는 방식, 그리고 `IsValid()`가 여전히 필요한 이유.

## 언제 이 아티클을 보나

"`OnDeactivate()` 호출 중에 `Render()`가 동시에 실행될 수 있나?"라는 질문이 생겼을 때.
또는 자신의 OVObject 멤버에는 뮤텍스 없이 접근하면서, 다른 OVObject 핸들에는 `IsValid()`를 쓰는 이유를 설명해야 할 때.

## GrandCentral 프레임워크 직렬화 보증

GC 프레임워크는 OVObject 생명주기 콜백과 Render() 사이를 다음 순서로 직렬화한다.

```
1. OnActivate() 완료
        ↓
2. Render() 호출 시작 (프레임마다)
        ↓
3. OnDeactivate() 요청 발생
        ↓
4. 진행 중인 Render() 완료 대기
        ↓
5. OnDeactivate() 실행
        ↓
6. 핸들이 Invalid로 전환
```

즉, **자기 자신의 OVObject에 한해** `OnDeactivate()`와 `Render()`는 동시에 실행되지 않는다.
뮤텍스 없이도 자기 멤버 변수에 접근하는 코드가 안전한 이유다.

## IsValid()가 여전히 필요한 경우

직렬화 보증은 "자기 자신"의 콜백에만 적용된다. 다른 OVObject의 핸들을 Render() 안에서 사용하면, 상대방 OVObject가 그 순간 `OnDeactivate()` 중일 수 있다.

```cpp
void SWEAcquisitionAssistant::Render()
{
    // ✅ 자기 멤버 접근 — OnDeactivate와 직렬화됨, 뮤텍스 불필요
    float score = m_score.load();  // m_score는 자기 멤버
    DrawScoreBar(score);

    // ✅ IsValid() 필수 — 다른 OVObject 핸들
    // LiverAIProcessor는 별도 OVObject — Render() 도중 deactivate될 수 있음
    auto handle = GcUdtRegistry::Get<LiverAIProcessor>("Gc.LiverAI");
    if (!handle.IsValid())  // ← 이 검사는 여전히 개발자 책임
        return;
    handle->DrawAiOverlay();
}
```

`IsValid()`는 "핸들이 현재 유효한가"를 확인한다. `Get<T>()`로 핸들을 얻은 직후 상대방이 deactivate되더라도, `IsValid()` 시점에 이미 Invalid로 전환되어 있으면 안전하게 리턴한다.

## SetParameter ↔ Render 간 공유 변수

프레임워크가 직렬화하지 않는 또 다른 쌍이 있다: `SetParameter()`와 `Render()`.
`SetParameter()`는 GC 파라미터 버스 스레드에서, `Render()`는 렌더 스레드에서 호출된다.

```cpp
// ❌ 보호 없이 공유하면 데이터 레이스
void SWEAcquisitionAssistant::SetParameter(
    const std::string& key, const Variant& value) override
{
    if (key == "SWEAcqAssist.Score")
        m_score = value.AsFloat();  // 렌더 스레드와 동시 접근 가능
}

void SWEAcquisitionAssistant::Render()
{
    float score = m_score;  // ← 레이스
}

// ✅ atomic 또는 ScCommon::Mutex로 보호 (개발자 책임)
void SWEAcquisitionAssistant::SetParameter(
    const std::string& key, const Variant& value) override
{
    if (key == "SWEAcqAssist.Score")
        m_score.store(value.AsFloat());  // std::atomic<float>
}

void SWEAcquisitionAssistant::Render()
{
    float score = m_score.load();  // 안전
}
```

## 책임 분류표

| 상황 | 보호 수단 | 책임 |
|------|-----------|------|
| 자기 OVObject의 멤버 변수 (Render ↔ OnDeactivate) | 프레임워크가 직렬화 | 프레임워크 |
| SetParameter ↔ Render 간 공유 변수 | `std::atomic` 또는 `ScCommon::Mutex` | 개발자 |
| 다른 OVObject 핸들 | `IsValid()` 검사 | 개발자 |
| 다른 OVObject의 내부 상태 직접 접근 | 직접 접근 불가 — GC 파라미터 버스 경유 | 아키텍처 설계 |

## Key Points

- 프레임워크는 **자기 자신의 `OnDeactivate()` ↔ `Render()` 쌍**만 직렬화한다 — 다른 OVObject와의 관계는 보증하지 않는다
- `GcUdtRegistry::Get<T>()`로 얻은 핸들은 항상 `IsValid()` 후 사용한다 — 핸들 획득과 사용 사이에 상대방이 deactivate될 수 있다
- `SetParameter()` ↔ `Render()` 사이의 공유 변수는 프레임워크 직렬화 범위 밖이다 — `std::atomic` 또는 `ScCommon::Mutex`로 보호해야 한다
- 다른 OVObject의 내부 상태가 필요하면 직접 접근하지 않고 GC 파라미터 버스를 경유한다 — 레이어 분리 원칙
- `OnActivate()` 완료 전에는 `Render()`가 호출되지 않는다 — 초기화 완료 여부를 별도 플래그로 관리할 필요 없다
