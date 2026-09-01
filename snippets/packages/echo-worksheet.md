---
title: "EchoWorksheet — 측정 결과 리포트 & 워크시트"
category: packages
tags: [report, worksheet, SWE, UGAP, facade, output]
difficulty: intermediate
---

SWE(kPa)와 UGAP(dB/cm/MHz) 측정 결과를 임상 보고서(워크시트)로 변환·출력하는 패키지. AA 구현체와는 Manager facade를 통해 결합이 차단된다.

## 역할

- 측정 결과(EchoMeasure 출력)를 임상 워크시트 형식으로 포매팅
- 인쇄·PDF 내보내기 및 DICOM SR 생성
- 경고/권고 사항(EchoDecisionSupport 출력) 리포트에 통합
- Manager facade가 AA 클래스와의 직접 결합을 차단

## 위치

```
src/packages/EchoWorksheet/
├── EchoWorksheetManager.h/.cpp  ← Manager facade, 측정/경고 집계
├── WorksheetRenderer.h/.cpp     ← 워크시트 UI 렌더링
├── WorksheetExporter.h/.cpp     ← 인쇄/PDF/DICOM SR 내보내기
└── WorksheetTemplate.h/.cpp     ← 리포트 템플릿 정의
```

## 핵심 클래스

| 클래스 | 역할 |
|---|---|
| `EchoWorksheetManager` | 측정 결과·경고 집계, AA와의 유일한 접촉점 |
| `WorksheetRenderer` | 임상 워크시트 화면 렌더링 |
| `WorksheetExporter` | 인쇄, PDF, DICOM SR 출력 |
| `WorksheetTemplate` | 리포트 레이아웃·섹션 정의 |

## 의존 관계

```
EchoWorksheet
  → EchoMeasure          (SWE kPa, UGAP dB/cm/MHz 수치)
  → EchoDecisionSupport  (경고·권고 데이터)
  → EchoConfig           (출력 형식 설정)
  ← (최종 출력 패키지 — 아무도 EchoWorksheet를 소비하지 않음)
```

## 데이터 흐름

```
EchoScanner → GcViewer(AA) → EchoMeasure
                                    ↓
EchoDecisionSupport ────────→ EchoWorksheet → 인쇄/PDF/DICOM
```

## 주의사항

- GcViewer AA 구현체를 **직접 include 금지** — `EchoWorksheetManager`가 유일한 facade 진입점
- DICOM SR 출력은 태그 매핑 오류가 임상 데이터 오류로 직결됨 — 변경 시 검증 필수
- SWE와 UGAP 결과가 동일 워크시트에 혼재할 수 있음 — 단위 레이블 명시 누락 주의
