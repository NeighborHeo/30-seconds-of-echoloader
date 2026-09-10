---
title: "프로브 종류별 동작 분기: 코드 어디서 갈리는가"
category: domain
tags: [probe, preset, esMain, acqassist, branching, debugging]
difficulty: intermediate
---

"Probe A는 되고 Probe B는 안 된다" 버그를 받았을 때, 코드 분기점이 어디에 있는지 계층별로 찾는 법.

## 언제 이 아티클을 보나

- 특정 프로브에서만 기능이 동작하지 않는 버그를 받았다
- 프로브 교체 후 상태가 올바르게 초기화되지 않는 것 같다
- 프로브별로 알고리즘 파라미터가 어디서 달라지는지 모르겠다

## Level 1: 프리셋 파일 (최상위 분기)

프로브별 동작 차이의 가장 흔한 원인. EchoConfig가 프로브 ID를 읽어 해당 프로브의 프리셋 XML을 로딩한다. 같은 기능이 프로브마다 다른 파라미터 값으로 동작한다.

```bash
# 프로브 ID별 프리셋 파일 위치
# EchoConfig가 관리하는 프리셋 파일 탐색
find . -name "*.xml" -path "*/preset*" | head -20
grep -rn "probe\|Probe" src/EchoConfig/ --include="*.h" | grep -i "id\|type\|name"
```

두 프로브의 프리셋 XML을 diff하면 파라미터 차이가 바로 드러난다. 런타임 분기보다 프리셋 차이가 원인인 경우가 더 많다.

## Level 2: ESMain의 프로브 의존 조건

프리셋이 동일해도 ESMain에서 프로브 타입별로 런타임 분기가 있을 수 있다.

```bash
# 프로브 타입에 따른 분기
grep -n "probe\|Probe" src/EchoScanner/ESMain.cpp | grep -i "type\|id\|model"

# 프로브 교체 이벤트 핸들러
grep -n "OnProbe\|ProbeChanged\|ProbeConnected" src/EchoScanner/ESMain.cpp
```

`OnProbeConnected()` 핸들러에서 Probe B 전용 초기화가 누락된 경우, 핫스왑(재시작 없이 교체) 시에만 증상이 나타나고 재시작 후에는 정상처럼 보인다.

## Level 3: AcquisitionAssistant의 프로브별 파라미터

Handler 레벨에서 프로브 정보를 받아 SCD 임계값, ROI 크기, 알고리즘 파라미터를 분기하는 경우가 있다.

```bash
# Handler가 프로브 정보를 어떻게 받는가
grep -rn "probe\|Probe" src/EchoScanner/AcquisitionAssistant/ --include="*.h"

# 프로브별 SCD 임계값, ROI 크기, 알고리즘 파라미터가 다를 수 있음
```

## 진단 절차

```
"Probe A는 되고 Probe B는 안 된다" 버그 접근법:

1. 프리셋 파일 차이 확인:
   두 프로브의 프리셋 XML을 diff → 다른 파라미터가 원인일 가능성 높음

2. ESMain의 프로브 조건 분기 확인:
   grep -n "ProbeModel\|ProbeType" src/EchoScanner/ESMain.cpp

3. Handler 내 프로브 파라미터 확인:
   해당 기능의 Handler에서 프로브 의존 파라미터 조회 위치

4. 프로브 교체 이벤트 흐름 추적:
   OnProbeConnected() → ESMain → Manager → Handler 흐름에서
   Probe B 특정 초기화가 누락됐는지 확인

5. 재현 방법:
   같은 기기에서 Probe A/B 교체 후 재테스트
   → 프리셋 로딩 문제라면 재시작 후에도 동일
   → 런타임 분기 문제라면 핫스왑(재시작 없이 교체)에서만 발생
```

## Key Points

- 프로브별 동작 차이의 가장 흔한 원인은 런타임 코드 분기가 아니라 프리셋 XML 파라미터 차이다 — 코드를 보기 전에 프리셋 diff를 먼저 한다
- "재시작하면 된다" 재현 패턴은 `OnProbeConnected()` 초기화 누락을 강하게 시사한다 — 프리셋 문제는 재시작 후에도 동일하게 나타난다
- ESMain의 프로브 조건 분기(`ProbeModel`, `ProbeType` 기반)와 Handler의 파라미터 분기는 별개의 레이어다 — 증상이 "기능 자체가 안 됨"이면 ESMain, "기능은 동작하나 수치가 다름"이면 Handler 레이어를 먼저 본다
- `OnProbeConnected()` 핸들러에서 새 프로브 타입에 대한 초기화 분기가 누락되면, 기존 프로브 상태가 그대로 유지된 채 새 프로브 프레임을 처리하게 된다
- EchoConfig의 프로브 ID → 프리셋 매핑이 깨지면 해당 프로브에서 모든 파라미터가 기본값으로 동작한다 — 로그에서 "preset not found" 또는 기본값 로딩 메시지를 확인한다
