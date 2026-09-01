---
title: "리팩토링 PR 제출 전 최종 체크리스트"
category: workflow
tags: [PR, checklist, refactoring, code-review, C++]
difficulty: intermediate
---

PR 열기 전 이 체크리스트를 순서대로 통과해야 한다.

## 명명 / 구조

- [ ] 클래스·함수: PascalCase / `GetFoo()`, `SetFoo()` 형식
- [ ] 멤버변수 `m_*`, 정적 멤버 `s_*`, 전역 `g_*`
- [ ] 모든 새 헤더에 `#pragma once`
- [ ] GE CONFIDENTIAL 블록 삽입 (자동생성·3rd-party 제외)

## 메모리 / 스레딩

- [ ] 소유권 이전 시 `unique_ptr` 우선, 원시 `new` 금지
- [ ] 스레드 친화도 변경 시 주석 추가 (`// runs on AcqThread`)
- [ ] 공유 리소스 접근 시 락 전략 명시

## 에러 처리

- [ ] 반환 형식: `bool` + out-param (예외 미사용)
- [ ] 실패 경로에 `ScLogError("MODULE_KEY", "상세 메시지")` 추가
- [ ] null 포인터 / 범위 초과 입력 모두 방어

## 추적성

- [ ] 변경 지점에 `// Phase X.Y — 설명` 마커
- [ ] `§` 번호로 설계문서 섹션 참조
- [ ] RBUG / FBUG / CR 번호를 코드 주석 및 Phase 문서에 기록
- [ ] PR 커밋 메시지에 `[X.Y-NNN]` 접두사

## 컴파일

- [ ] MSVC `/W4` 경고 0개
- [ ] `/permissive-` 모드 통과
- [ ] C++11/14 상한 준수 (C++17 기능 사용 금지)
- [ ] 가상함수 재정의에 `override` 명시
- [ ] 변경 안 되는 값에 `const` 적용

## 테스트

- [ ] `TEST_F(XxxFixture, Require_that_...)` 형식 테스트 추가
- [ ] 파일 위치: `EchoXxx/UnitTest/TestMyFeature.cpp`
- [ ] 새 로직 분기마다 대응 테스트 존재
- [ ] `EXPECT_NO_THROW` 대신 반환값 직접 검증

## 의료기기 안전

- [ ] 모든 포인터 역참조 전 null 체크
- [ ] `switch` 문에 모든 열거값 또는 `default` 처리
- [ ] `FatalErrorHandler` / `WatchdogCB` 경로에 영향 없음 확인
- [ ] 보안 이벤트 변경 시 `AUDIT_LOG` / `SECURITY_ALERT` 추가
