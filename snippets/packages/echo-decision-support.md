---
title: "EchoDecisionSupport — 임상 의사결정 지원"
category: packages
tags: [clinical, LiverAI, warning, guideline, SWE, UGAP, pipeline]
difficulty: advanced
---

SWE/UGAP 측정 결과를 임상 가이드라인과 대조해 경고 및 권고를 생성하는 패키지. LiverAI Processor 파이프라인의 상위 소비자.

## 역할

- SWE(kPa) / UGAP(dB/cm/MHz) 측정값을 임상 기준값과 비교
- 간 섬유화 단계(F0–F4) 판정 및 경고 생성
- `ProbeContactQualityIndicator` — 프로브 접촉 품질 실시간 평가
- GcViewer의 `LiverWarning3Panel`에 경고 데이터 공급

## 위치

```
src/packages/EchoDecisionSupport/
├── EchoDecisionSupportBase.h/.cpp     ← 공통 판정 로직
├── EchoDecisionSupportPanel.h/.cpp    ← 경고 패널 UI 바인딩
├── ProbeContactQualityIndicator.h/.cpp← 프로브 접촉 품질 평가
└── LiverAIProcessorConsumer.h/.cpp    ← LiverAI 파이프라인 소비 인터페이스
```

## 핵심 클래스

| 클래스 | 역할 |
|---|---|
| `EchoDecisionSupportBase` | 임상 기준값 비교, 경고 레벨 결정 |
| `EchoDecisionSupportPanel` | 경고를 GcViewer `LiverWarning3Panel`에 바인딩 |
| `ProbeContactQualityIndicator` | 측정 신뢰도 품질 지표 계산 |
| `LiverAIProcessorConsumer` | LiverAI Processor 출력 소비 인터페이스 |

## 의존 관계

```
EchoDecisionSupport
  → EchoMeasure          (SWE/UGAP 수치 소비)
  → GcViewer/ObjectViewer (LiverWarning3Panel, AcquisitionAssistantBase)
  → LiverAI Processor    (AI 추론 결과 소비)
  ← EchoWorksheet        (경고 결과를 리포트에 포함)
```

## LiverAI 파이프라인 소비 구조

```
LiverAI Processor
  → EchoDecisionSupportBase    (판정)
  → EchoDecisionSupportPanel   (UI)
  → ProbeContactQualityIndicator (품질)
  ↑ 이 3개만 LiverAIProcessor를 직접 소비
```

## 주의사항

- 임상 기준값(kPa 임계값 등)은 가이드라인 버전에 따라 변경됨 — 소스 주석에 근거 가이드라인 명시
- `ProbeContactQualityIndicator`는 실시간 경로 — 블로킹 연산 금지
- GcViewer `LiverWarning3Panel` API 변경 시 `EchoDecisionSupportPanel` 동시 수정 필요
