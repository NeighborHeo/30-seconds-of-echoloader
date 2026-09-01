---
title: "ESMain → 서브패키지 버그 트레이싱"
category: workflow
tags: [debugging, logging, ESMain, OVObject, crash]
difficulty: advanced
---

모듈 키로 로그 필터링 → 호출 사이트 grep → Layer 경계 확인 → 크래시 덤프 순서로 추적한다.

## 절차

1. **로그 필터링**: `ScLogInfo` / `ScLogError` 첫 인자(모듈 키) 기준
   ```
   모듈 키 예: "SHAPEMI", "STREAMING", "SWEMGR", "UGAPMGR"
   ```

2. **호출 사이트 grep** (`ESMain.cpp`)
   ```bash
   grep -n "Manager::Instance(HandlerType" ESMain.cpp
   # HandlerType::SWE, HandlerType::UGAP 분기 확인
   ```

3. **Layer 경계 이해**
   - Layer A: `EchoScanner/Manager` — 상위 오케스트레이션
   - Layer B: `GcViewer/OVObject` — 데이터 뷰 객체
   - 경계 넘을 때 `dynamic_cast` 실패가 자주 원인

4. **`dynamic_cast` 실패 찾기** (`ObjectViewerImpl.cpp` 패턴)
   ```cpp
   if (auto* p = dynamic_cast<GcMyClass*>(pObj))
   { /* 정상 */ }
   else
   { ScLogError("VIEWER", "unexpected OVObject type"); }
   ```

5. **크래시/와치독 추적**
   - `FatalErrorHandler` 콜스택 확인
   - `WatchdogCB` 타임아웃 경로 grep
   - 덤프 경로: `UpdatePreviousBootCrashDump` 함수에서 경로 추출

## 체크리스트

- [ ] 모듈 키로 로그 범위 좁히기
- [ ] `ESMain.cpp` `HandlerType` 호출 사이트 확인
- [ ] Layer A/B 경계에서 `dynamic_cast` 반환값 검사
- [ ] `ObjectViewerImpl.cpp` null 분기 로그 확인
- [ ] 크래시 시 `UpdatePreviousBootCrashDump` 경로로 덤프 수집
- [ ] `FatalErrorHandler` / `WatchdogCB` 콜스택 분석
