---
title: "GE CONFIDENTIAL 라이선스 헤더 추가"
category: workflow
tags: [license, header, compliance, GE]
difficulty: beginner
---

모든 내부 소스 파일 최상단에 GE CONFIDENTIAL 블록을 의무 삽입한다.

## 절차

1. 파일을 새로 만들거나 수정할 때 **가장 먼저** 헤더 존재 여부 확인
2. 없으면 아래 형식을 `#pragma once` / `#include` 보다 **앞에** 삽입
3. `Keywords` 필드에 해당 파일의 기능 태그 기입
4. **예외**: MIDL 자동생성 파일, `resource.h`, 매우 오래된 `Drivers/` 코드, 3rd-party (원본 라이선스 유지)

## 예시

```cpp
/*
 -GE CONFIDENTIAL-
 Type: Source Code
 Sensitivity: Critical
 Keywords: SWE, Acquisition, Streaming

 Copyright (c) 2024, GE Healthcare Co.
 All Rights Reserved

 This document is GE Healthcare proprietary information.
 Unauthorized copying, distribution or disclosure of this
 document or its contents is strictly prohibited.
*/
#pragma once
#include "SomeHeader.h"
```

## 체크리스트

- [ ] 파일 최상단 (shebang 제외 첫 번째 블록)
- [ ] `Keywords:` 필드에 기능 태그 기입
- [ ] `#pragma once` 보다 앞에 위치
- [ ] MIDL / resource.h / 3rd-party 파일에는 삽입 금지
- [ ] 연도가 현재 저작권 연도와 맞는지 확인
