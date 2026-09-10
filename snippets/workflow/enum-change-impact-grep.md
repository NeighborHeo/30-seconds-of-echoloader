---
title: "AcqAssistState에 새 enum 값 추가할 때 영향 범위 grep"
category: workflow
tags: [enum, grep, esmain, state, switch, layer-a, layer-b, msvc]
difficulty: intermediate
---

`AcqAssistState` 또는 레이어 간 공유 상태 enum에 값을 추가할 때 빠짐없이 업데이트할 위치를 찾는 방법.

## 언제 이 아티클을 보나

- 새 획득 상태를 enum에 추가하는 티켓을 받았을 때
- PR 리뷰 중 enum 값이 일부 switch에서 처리되지 않는지 확인할 때
- MSVC C4061 경고가 발생해 switch 누락 위치를 추적할 때

## 5단계 grep 순서

```bash
# 1단계: enum이 몇 개 헤더에 정의되어 있나? (Layer A/B 동기화 필요)
grep -rn "enum.*AcqAssistState\|AcqAssistState {" src/ --include="*.h"
# → 두 곳 이상 있으면: 모두 동기화 필요

# 2단계: switch문 중 새 case가 빠질 수 있는 곳
grep -rn "switch.*AcqAssistState\|switch.*m_state\|switch.*state)" src/ --include="*.cpp"
# → 각 switch에 새 case 추가 여부 확인

# 3단계: if-else 체인 (switch 아닌 형태)
grep -rn "AcqAssistState::\|== Active\|== Idle\|== Acquiring" src/ --include="*.cpp"
# → 새 enum 값 처리 누락 여부

# 4단계: GC 파라미터로 int 캐스트되는 위치
grep -rn "static_cast<int>.*state\|SetParameterValue.*state" src/ --include="*.cpp"
# → Layer B가 수신하는 측도 새 값 처리 필요

# 5단계: GoogleTest 중 하드코딩된 state 값
grep -rn "AcqAssistState::" src/ --include="*.cpp" | grep -i "test\|fixture\|mock"
# → 테스트가 특정 enum 값에 의존하면 업데이트 필요
```

## 추가 후 완료 체크리스트

```
새 AcqAssistState::NewState 추가 체크리스트:
[ ] Layer A 헤더의 enum 정의에 추가
[ ] Layer B 헤더의 동기화된 enum 정의에 추가 (존재한다면)
[ ] ESMain의 관련 switch/if에 case 추가
[ ] Manager/Handler의 switch/if에 case 추가
[ ] Layer B OVObject의 SetParameter switch에 case 추가
[ ] GC 파라미터로 int 전달 → 수신측 범위 검사 업데이트
[ ] 단위 테스트에 새 상태 커버리지 추가
```

## MSVC /W4가 잡는 것과 못 잡는 것

| 패턴 | /W4 경고 | 이유 |
|------|----------|------|
| `switch (state)` — 새 case 누락 | C4061 발생 | `default` 없는 switch에서 열거값 미처리 경고 |
| `switch (state)` — `default:` 있음 | 경고 없음 | default가 있으면 C4061 억제됨 |
| `if (state == Active) else if ...` | 경고 없음 | if-else 체인은 exhaustive 검사 없음 |
| Layer B int 수신측 범위 검사 | 경고 없음 | int로 캐스트된 이후 enum 정보 소실 |

switch에 `default:` 블록이 있으면 C4061이 억제된다. if-else 체인은 어떤 경우도 경고가 없다. **grep이 컴파일러보다 더 많은 위치를 찾는다.**

## 실전 팁: Layer A/B 동기화 확인

```bash
# Layer A 정의
grep -n "AcqAssistState" src/AcquisitionAssistant/AcqAssistState.h

# Layer B 미러 정의 (GcViewer 쪽)
grep -rn "AcqAssistState" src/GcViewer/ --include="*.h"

# 두 파일의 값 목록 비교 — 줄 수가 다르면 동기화 깨짐
```

## Key Points

- enum 정의가 Layer A/B 양쪽에 있으면 두 파일을 동시에 수정해야 한다.
- `switch` + `default:` 조합은 C4061을 억제하므로 grep으로 직접 확인해야 한다.
- if-else 체인은 컴파일러 경고가 전혀 없으므로 3단계 grep이 필수다.
- GC 파라미터로 int 캐스트되는 경로는 컴파일러가 추적 불가 — 4단계 grep 필수다.
- 테스트의 하드코딩된 enum 값은 새 값 추가 시 테스트 논리가 틀어지는 silent 버그 원인이 된다.
