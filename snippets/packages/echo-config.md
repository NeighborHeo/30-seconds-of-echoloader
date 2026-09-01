---
title: "EchoConfig / EchoConfigManager — 설정 & 프리셋 관리"
category: packages
tags: [config, preset, parameterization, UGAP, SWE]
difficulty: intermediate
---

애플리케이션 전체 설정과 모드별 프리셋 문자열을 관리. `Init(presetName)` 파라미터화의 근거이며, `"ShearTrackAlg"` 같은 프리셋 키가 여기서 출처를 갖는다.

## 역할

- 프리셋 파일(XML/INI) 로드 및 파싱
- 모드별 설정 키(`"ShearTrackAlg"`, `"UGAPMode"` 등) 제공
- `Init(presetName)` 호출로 컨텍스트별 파라미터 스위칭
- 런타임 설정 변경(프로브 교체, 모드 전환) 브로드캐스트

## 위치

```
src/packages/EchoConfig/
├── EchoConfigManager.h/.cpp   ← 싱글턴 설정 관리자
├── PresetLoader.h/.cpp        ← 프리셋 파일 파싱
├── ConfigKeys.h               ← 프리셋 키 문자열 상수
└── presets/
    ├── SWE_default.xml
    └── UGAP_default.xml
```

## 핵심 클래스

| 클래스 | 역할 |
|---|---|
| `EchoConfigManager` | 설정 CRUD, 구독자 알림 |
| `PresetLoader` | XML/INI → 내부 설정 맵 변환 |
| `ConfigKeys` | 프리셋 키 문자열 상수 정의 |

## 의존 관계

```
EchoConfig
  → (파일시스템, XML 파서)
  ← EchoScanner   (획득 파라미터 조회)
  ← GcViewer      (렌더링 파라미터 조회)
  ← EchoMeasure   (단위·포맷 설정)
  ← EchoLoader    (최초 Init)
```

## 주의사항

- UGAP 프리셋은 SWE와 키 네임스페이스가 겹칠 수 있음 — `ConfigKeys.h`에서 접두사 구분 확인
- 프리셋 파일 누락 시 조용히 기본값으로 폴백하는 경우 있음 — 검증 로그 반드시 확인
- `Init(presetName)` 호출 전에 파라미터를 읽으면 이전 프리셋 값이 반환됨 — 호출 순서 의존성 주의
