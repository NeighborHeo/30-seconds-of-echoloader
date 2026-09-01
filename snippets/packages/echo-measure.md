---
title: "EchoMeasure — 측정 도구"
category: packages
tags: [measurement, SWE, UGAP, kPa, facade, decoupling]
difficulty: intermediate
---

SWE(kPa, m/s)와 UGAP(dB/cm/MHz) 측정값을 계산하고 표시하는 패키지. GcViewer의 AA 클래스와는 **Manager facade**로만 연결되어 직접 결합이 없다.

## 역할

- SWE 탄성도(kPa, m/s) 계산 및 결과 포매팅
- UGAP 감쇠 계수(dB/cm/MHz) 계산 및 결과 포매팅
- 측정 세션 관리(시작/종료/저장)
- Manager facade를 통해 AA 구현체와 완전히 디커플됨

## 위치

```
src/packages/EchoMeasure/
├── EchoMeasureManager.h/.cpp   ← Manager facade (AA 인터페이스만 참조)
├── SWEMeasurement.h/.cpp       ← SWE kPa/m/s 계산
├── UGAPMeasurement.h/.cpp      ← UGAP dB/cm/MHz 계산
└── MeasurementDisplay.h/.cpp   ← 측정값 UI 포매팅
```

## 핵심 클래스

| 클래스 | 역할 |
|---|---|
| `EchoMeasureManager` | 측정 세션 조율, AA와의 유일한 접촉점 |
| `SWEMeasurement` | Shear Wave 탄성도 수치 계산 |
| `UGAPMeasurement` | 초음파 감쇠 파라미터 수치 계산 |
| `MeasurementDisplay` | 수치 → 표시 문자열 변환 |

## 의존 관계

```
EchoMeasure
  → IAcqAssistManager  (EchoScanner 인터페이스, 직접 GcViewer import 없음)
  → EchoConfig         (단위·포맷 설정)
  ← EchoWorksheet      (측정 결과 소비)
  ← EchoDecisionSupport (임상 판단 입력)
```

## 주의사항

- GcViewer AA 구현체(`SWEAcquisitionAssistant` 등)를 **직접 include 금지** — facade 패턴 깨짐
- 단위 혼용 버그 위험: kPa ↔ m/s 변환 상수는 `SWEMeasurement` 내부에만 두고 중복 정의 방지
- UGAP dB/cm/MHz는 주파수 정규화가 포함된 값 — 단순 dB와 혼동 주의
