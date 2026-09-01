---
title: "3개 로깅 시스템: ILogSimple / DBLogger / LogShim"
category: cpp-patterns
tags: [logging, ILogSimple, DBLogger, LogShim, ScLog, architecture]
difficulty: intermediate
---

gipc-app에는 3개의 로깅 시스템이 공존한다. 목적이 다르며 혼용하면 로그가 누락된다.

## 3개 시스템 비교

```
시스템          헤더                          매크로/API            용도
─────────────   ─────────────────────────     ─────────────────     ──────────────────
ILogSimple      <ScLogsDatabase/ILogSimple.h>  ScLogInfo / ScLogError  일반 진단 로그 (주력)
DBLogger        <mcd/DBLogger.h>              DBLogger::Log(...)    임상 측정값 DB 기록
LogShim         <ScCommon/LogShim.h>          ScLog*(통합 매크로)   ILogSimple 래퍼 (구형 코드)
```

## ILogSimple — 일반 진단 로그 (신규 코드 표준)

```cpp
#include <ScLogsDatabase/ILogSimple.h>

// 형식: ScLog<Level>("MODULE_KEY", "format", args...)
ScLogInfo ("SWEAA",      "Processing frame %d, roi=(%.1f, %.1f)",
           m_frameCount, m_roiX, m_roiY);

ScLogError("SWEAA",      "ROI handle invalid after mode switch");

ScLogWarn ("UGAPAA",     "Tilt exceeds threshold: %.1f deg", tiltMag);

// 모듈 키 규칙:
//   SWEAA   = SWE Acquisition Assistant
//   UGAPAA  = UGAP Acquisition Assistant
//   BOOTCHECK = 부팅 검사
//   STREAMING = 원격 스트리밍
//   PRESET    = 프리셋 로딩
```

## DBLogger — 임상 측정 데이터 기록

```cpp
#include <mcd/DBLogger.h>

// 임상 측정값을 영구 DB에 기록 — ScLogInfo 대신 DBLogger
// 용도: 환자 보고서, DICOM 내보내기, 감사 추적 (임상 데이터)
DBLogger::Log(DBLogger::Category::Measurement,
    "SWE_STIFFNESS",    // 측정 항목 키
    {{"value", 8.3f},   // kPa
     {"unit",  "kPa"},
     {"roi_x", m_roiX},
     {"roi_y", m_roiY},
     {"preset", m_presetName}});

// DBLogger를 ScLogInfo로 대체하면:
//  - 임상 측정 기록 누락 → FDA/IEC 62304 추적성 위반
//  - DICOM SR에 데이터 미포함 → 환자 보고서 불완전
```

## LogShim — 레거시 코드의 ILogSimple 래퍼

```cpp
#include <ScCommon/LogShim.h>

// 구형 코드가 LogShim을 통해 ILogSimple에 위임
// 신규 코드는 ILogSimple 직접 사용 권장
// 신규 코드에서 LogShim 추가 금지
ScLog("LEGACY_MODULE", ScLog::Error, "Something went wrong");
// → 내부적으로 ScLogError("LEGACY_MODULE", ...) 로 위임
```

## 선택 결정 트리

```
새 로그를 추가하려 할 때:

임상 측정 데이터인가?
  YES → DBLogger::Log()
  NO  →
    보안/감사 이벤트인가?
      YES → AuditLogUserEvent()
      NO  → ScLogInfo() / ScLogError()  ← 일반 케이스
             (모듈 키 필수!)
```

## Key Points

- **신규 코드에서 LogShim 사용 금지** — ILogSimple 직접 사용
- 모듈 키 없는 `ScLogInfo("", "...")` 은 필터링 불가 → 반드시 의미있는 키 지정
- DBLogger 누락은 임상 규정 위반 — 측정 결과 기록 코드 리뷰 시 DBLogger 사용 여부 확인
- 세 시스템 모두 영구 스토리지에 기록됨 — 디버그 전용 과다 로그는 스토리지 고려
- `ScLogError` 후 반드시 `return false` (또는 에러 전파) — 로그만 남기고 계속 진행하는 패턴 금지
