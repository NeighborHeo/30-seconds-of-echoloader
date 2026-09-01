---
title: "Include Order Convention"
category: cpp
tags: [conventions, includes, header, build]
difficulty: beginner
---

gipc-app의 `#include` 순서는 `StdAfx.h` → 프로젝트 헤더 → ScCommon/mcd → STL. 순서가 뒤집히면 빌드가 깨진다.

## Pattern

```cpp
/*
-GE CONFIDENTIAL-
...
*/
#pragma once

// 1. Precompiled header — 항상 첫 번째
#include "StdAfx.h"

// 2. 로컬 프로젝트 헤더 — 같은 패키지/모듈
#include "AcquisitionAssistantBase.h"
#include "LiverWarning3Panel.h"
#include "SWEROIProcessor.h"

// 3. GE 공통 라이브러리 — <꺾쇠 괄호>
#include <ScCommon/Thread.h>
#include <ScCommon/LogShim.h>
#include <mcd/DBLogger.h>
#include <ScLogsDatabase/ILogSimple.h>

// 4. STL / 표준 라이브러리 — 마지막
#include <vector>
#include <string>
#include <memory>
#include <mutex>
```

## 헤더 가드 혼용

```cpp
// 68%: pragma once (신규 코드 표준)
#pragma once

// 32%: 구식 include guard (레거시)
#ifndef _ACQUISITION_ASSISTANT_BASE_H_
#define _ACQUISITION_ASSISTANT_BASE_H_
// ...
#endif // _ACQUISITION_ASSISTANT_BASE_H_
```

## Key Points

- `StdAfx.h` 를 첫 줄에 빠트리면 precompiled header 미스로 빌드 시간이 폭발적으로 증가
- `#pragma once` 와 `#ifndef` 가드 혼용됨 — 기존 파일은 기존 스타일 유지, 신규는 `#pragma once`
- `<ScCommon/...>` 과 `<mcd/...>` 는 GE 공통 인프라 — 절대 경로가 아닌 include path 에 등록된 꺾쇠 괄호 형식
- STL 은 항상 마지막 — MFC/Windows 헤더보다 뒤에 와야 충돌 없음
- `.clang-format` 없음 → include 정렬은 컨벤션 문서 + 코드리뷰에만 의존
