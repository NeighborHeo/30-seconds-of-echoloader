---
title: "DICOM Structured Report(SR)와 심장 측정값 저장"
category: domain
tags: [dicom, sr, structured-report, measurement, echoworksheet, tags]
difficulty: intermediate
---

EchoWorksheet가 심장 측정값을 저장할 때 DICOM SR 트리로 인코딩한다. 코드/단위 불일치는 규제 지적 사항이다.

## Why

초음파 측정값은 단순 숫자가 아니다.
"좌심실 박출률 65%"에는 측정 방법(Simpson's Biplane), 단위(%), 코딩 표준(SNOMED/UCUM), 타임스탬프가 함께 있어야 병원 PACS에서 의미를 해석할 수 있다.
DICOM SR(Structured Report)는 이 정보를 트리 구조로 표준화한다.

## Pattern

```
DICOM SR 트리 구조 (심장 박출률 예시)

ROOT Container (CONTAINER)
│
├─ ConceptNameCodeSequence: "Echocardiography Report" (SNOMED: 433871000124107)
│
└─ Content Items
    │
    ├─ [CONTAINER] "Left Ventricle"
    │   ├─ ConceptNameCode: "Left Ventricle" (SNOMED: 87878005)
    │   │
    │   ├─ [NUM] "Left Ventricular Ejection Fraction"
    │   │   ├─ ConceptNameCode : (SNOMED: 250908004)
    │   │   ├─ MeasuredValue   : 65
    │   │   ├─ MeasurementUnit : % (UCUM: %)
    │   │   └─ MeasurementMethod: Simpson's Biplane (SNOMED: 418531000)
    │   │
    │   ├─ [NUM] "LV End-Diastolic Volume"
    │   │   ├─ MeasuredValue   : 120
    │   │   └─ MeasurementUnit : mL (UCUM: mL)
    │   │
    │   └─ [NUM] "LV End-Systolic Volume"
    │       ├─ MeasuredValue   : 42
    │       └─ MeasurementUnit : mL
    │
    └─ [DATE] "Study Date" : 2026-09-04
```

```
DICOM 태그 매핑 (핵심 태그)

Tag            VR    이름
----------------------------------------------------------------------
(0040,A730)   SQ    ContentSequence       — SR 내용 트리
(0040,A010)   CS    RelationshipType      — CONTAINS / HAS OBS CONTEXT
(0040,A040)   CS    ValueType             — NUM / TEXT / CODE / CONTAINER
(0040,A043)   SQ    ConceptNameCodeSeq    — 측정 항목 이름 (코드)
(0040,A300)   SQ    MeasuredValueSeq      — 숫자값 + 단위
(0040,08EA)   SQ    MeasurementUnitsCodeSeq — UCUM 단위 코드
(0040,A168)   SQ    ConceptCodeSeq        — 코드 값 (메서드 등)
```

```cpp
// gipc-app 측정 결과 → DICOM SR 변환 패턴 (의사코드)
void EchoWorksheet::SaveMeasurementToDicom(const LvMeasurement& meas)
{
    DicomSrItem efItem;
    efItem.SetValueType(DicomSrItem::NUM);
    // SNOMED 코드 정확히 지정 — 오타나 임의 코드 사용 금지
    efItem.SetConceptName("250908004", "SRT", "Left Ventricular Ejection Fraction");
    efItem.SetNumericValue(meas.m_fEjectionFraction);
    // UCUM 단위 — "%" 아니라 UCUM 코드 "%" 그대로 사용
    efItem.SetUnit("%", "UCUM", "%");
    efItem.SetMethod("418531000", "SRT", "Simpson's Biplane Method");

    m_srRoot.AddContentItem(efItem);
    m_srRoot.Finalize();  // SOPClassUID = Enhanced SR (1.2.840.10008.5.1.4.1.1.88.22)
}
```

## Key Points

- **코드 불일치 = 규제 발견**: SNOMED 코드를 임의 번호로 대체하거나, 단위를 "cc"(UCUM 아님) 대신 "mL"로 써야 할 자리에 틀리면 FDA/CE 심사에서 Non-Conformance
- UCUM 단위 표준 사용: mL, cm, m/s, mmHg — 자유 텍스트 단위 금지
- SR SOPClassUID는 측정 유형에 따라 다름: Comprehensive SR, Enhanced SR, Basic Diagnostic Imaging Report — 잘못된 클래스로 저장하면 PACS가 파싱 거부
- `EchoWorksheet`가 SR을 생성한 후 DICOM 스토어로 전송할 때 네트워크 오류가 발생해도 로컬 캐시가 남아있어야 함 (데이터 유실 = 환자 안전 이슈)
- 측정 방법(Simpson's Biplane, Area-Length 등)이 SR에 명시되지 않으면 수신 시스템이 EF 값의 임상 의미를 판단할 수 없음 — 방법 코드 필수
