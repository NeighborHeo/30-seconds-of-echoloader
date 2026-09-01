# 30 Seconds of EchoLoader

> **gipc-app** 코드베이스를 30초 안에 이해하는 레퍼런스 카드 모음

GE Healthcare 초음파 진단기기(Vivid 시리즈) 소프트웨어 `gipc-app`에 대한 개발 지식, 도메인 지식, 아키텍처 지식을 **30초 안에 읽을 수 있는 아티클** 형식으로 정리했습니다. [30 seconds of code](https://30secondsofcode.org) 형식에서 영감을 받았습니다.

---

## 📦 카테고리

| 카테고리 | 수 | 설명 |
|---------|-----|------|
| [C++ 패턴](snippets/cpp/) | 15 | gipc-app에서 실제로 쓰이는 C++ 관용구·컨벤션 |
| [고급 C++ 패턴](snippets/cpp-patterns/) | 8 | COM/ATL, MFC, 싱글톤, 팩토리 등 심화 패턴 |
| [도메인 지식](snippets/domain/) | 16 | 초음파 의료 영상, SWE, UGAP, 간 측정 개념 |
| [아키텍처](snippets/architecture/) | 15 | 패키지 구조, 설계 패턴, 레이어 분리 전략 |
| [패키지 가이드](snippets/packages/) | 10 | Echo* 패키지별 역할, 의존 관계, 주의사항 |
| [워크플로](snippets/workflow/) | 8 | 개발 절차, PR 체크리스트, 디버깅, 보안 감사 |

**총 72개 아티클**

---

## 🗂 C++ 패턴 (15)

| 아티클 | 핵심 개념 |
|--------|----------|
| [bool + out-param 에러 처리](snippets/cpp/bool-out-param-error-handling.md) | `bool Fn(std::string& errorMsg)` |
| [스마트 포인터 소유권](snippets/cpp/smart-pointer-ownership.md) | unique_ptr / shared_ptr / GcUdtHandle |
| [명명 규칙](snippets/cpp/member-naming-conventions.md) | m_, s_, PascalCase, C접두어 |
| [헤더 가드 패턴](snippets/cpp/header-guard-patterns.md) | #pragma once vs #ifndef |
| [C++11/14 천장](snippets/cpp/cpp11-14-ceiling.md) | 허용/금지 기능 목록 |
| [ScLog 모듈 키 로깅](snippets/cpp/sclog-module-key-logging.md) | ScLogInfo("KEY", ...) |
| [null 체크 + 조기 반환](snippets/cpp/null-check-early-return.md) | 방어적 코딩 패턴 |
| [티켓 ID 주석](snippets/cpp/ticket-id-comments.md) | RBUG / FBUG / CR- 추적성 |
| [Phase 마커 추적성](snippets/cpp/phase-marker-traceability.md) | 리팩토링 Phase 주석 |
| [W4 경고 → 에러](snippets/cpp/w4-warnings-as-errors.md) | MSVC 23개 에러 승격 |
| [GoogleTest 명명 규칙](snippets/cpp/gtest-naming-convention.md) | `Require_that_*` + `*Fixture` |
| [스레딩 패턴](snippets/cpp/threading-patterns.md) | lock_guard, unique_lock, ScCommon::Thread |
| [MFC 다이얼로그 클래스 패턴](snippets/cpp/mfc-dialog-pattern.md) | `C*` 접두어, m_b*, m_n* |
| [Include 순서 컨벤션](snippets/cpp/include-order.md) | StdAfx → 로컬 → ScCommon → STL |
| [빌드 매크로: _DEBUG vs NDEBUG](snippets/cpp/build-macros-debug-ndebug.md) | DEBUG_NEW, EnableFastChecks |

---

## 🔬 고급 C++ 패턴 (8)

| 아티클 | 핵심 개념 |
|--------|----------|
| [const 정확성 패턴](snippets/cpp-patterns/const-correctness.md) | const 멤버 함수, const& 매개변수 |
| [COM/ATL HRESULT 처리](snippets/cpp-patterns/com-atl-hresult.md) | COM 인터페이스 에러 처리 |
| [GcUdtHandle 사용법](snippets/cpp-patterns/gc-udt-handle.md) | UDT 비소유 참조 핸들 |
| [MFC 다이얼로그 클래스](snippets/cpp-patterns/mfc-dialog-class.md) | CDialog 서브클래스 패턴 |
| [OVObject 팩토리 등록](snippets/cpp-patterns/ovobject-factory-registration.md) | Gc.* 태그 등록 방법 |
| [ScCommon Thread 래퍼](snippets/cpp-patterns/sccommon-thread-wrapper.md) | GE 스레드 풀 래퍼 |
| [싱글톤 Manager::Instance()](snippets/cpp-patterns/singleton-manager-instance.md) | 스레드 안전 싱글톤 |
| [상태머신 enum + switch](snippets/cpp-patterns/state-machine-enum-switch.md) | AcqAssistState 패턴 |

---

## 🏥 도메인 지식 (16)

| 아티클 | 핵심 개념 |
|--------|----------|
| [SWE - 전단파 탄성 초음파](snippets/domain/swe-shear-wave-elastography.md) | 간 섬유화 비침습 측정 |
| [UGAP - 초음파 감쇠 파라미터](snippets/domain/ugap-ultrasound-guided-attenuation-parameter.md) | 지방간 비침습 측정 |
| [간 경고 3종](snippets/domain/liver-warning-triad.md) | LargeSCD + PoorProbe + ObliqueCapsule |
| [Acquisition Assistant 상태 머신](snippets/domain/acq-assist-state-machine.md) | AcqAssistState 5개 상태 |
| [Acquisition Assistant Phase 생명주기](snippets/domain/acq-assist-phase-lifecycle.md) | Phase enum vs ScanState |
| [AutoPos 키 해석](snippets/domain/autopos-key-resolution.md) | SWE vs UGAP 키 차이 |
| [ROI Processor 개념](snippets/domain/roi-processor-concept.md) | SWEROIProcessor vs ACResults |
| [프로브 접촉 품질 지시자](snippets/domain/probe-contact-quality-indicator.md) | ProbeContactQualityIndicator |
| [LiverAI Processor](snippets/domain/liver-ai-processor.md) | AI 모델 파이프라인 |
| [피부-캡슐 거리 (SCD)](snippets/domain/skin-to-capsule-distance.md) | SCD 측정 및 경고 조건 |
| [프레임 버퍼 채널: SWE vs UGAP](snippets/domain/frame-buffer-channels.md) | 단일 채널 vs 2채널 구조 |
| [SWE: Freeze vs Continuous 모드](snippets/domain/swe-freeze-continuous-mode.md) | AutoPos 키 분기 이유 |
| [UGAP: Measure 모드와 분석 키](snippets/domain/ugap-measure-mode.md) | UGAPAcqAssistAnalyze 발행 조건 |
| [워치독 & 치명 오류 핸들러](snippets/domain/watchdog-crash-handler.md) | FatalErrorHandler + WatchdogCB |
| [감사 로그 사용자 이벤트](snippets/domain/audit-log-user-event.md) | AuditLogUserEvent + FDA 규정 |
| [기울기 각도 검증 (PoorProbe 기하)](snippets/domain/tilt-angle-validation.md) | m_leftTiltAngle UGAP 검증 |

---

## 🏗 아키텍처 (15)

| 아티클 | 핵심 개념 |
|--------|----------|
| [Layer A vs Layer B](snippets/architecture/layer-a-vs-layer-b.md) | EchoScanner vs GcViewer 분리 |
| [Manager-Factory-Handler 패턴](snippets/architecture/manager-factory-handler-pattern.md) | 3-파트 AA 패턴 |
| [OVObject 상속](snippets/architecture/ovobject-inheritance.md) | 렌더링 기반 클래스 |
| [ESMain 진입점](snippets/architecture/esmain-as-system-hub.md) | 11개 호출 사이트 |
| [Echo* 패키지 분류](snippets/architecture/echo-package-taxonomy.md) | ~50개 패키지 역할 |
| [Composition: LiverWarning3Panel 추출](snippets/architecture/composition-over-inheritance-liverwarning3panel.md) | 상속 → 컴포지션 |
| [Push-down vs Hoisting 전략](snippets/architecture/pushdown-vs-hoisting-strategy.md) | 리팩토링 방향 결정 |
| [SWE와 UGAP는 합칠 수 없다](snippets/architecture/swe-vs-ugap-cannot-merge.md) | 도메인 차이 5가지 |
| [AcqAssistState enum 중복](snippets/architecture/acqassiststate-enum-duplication.md) | 두 레이어의 enum 동기화 문제 |
| [외부 컨트랙트 - 절대 건드리지 말 것](snippets/architecture/external-contracts-dont-touch.md) | 4개 load-bearing 지점 |
| [GC 파라미터 버스: Layer A↔B 브리지](snippets/architecture/gc-parameter-bus.md) | 문자열 키/값 메시지 버스 |
| [P0b: Warning 단위 테스트 전략](snippets/architecture/p0b-unit-test-strategy.md) | P2+P4 진입 전 필수 테스트 |
| [P2+P4 전 Director 확인 질문 5가지](snippets/architecture/director-signoff-questions.md) | 열린 질문 + 의사결정 매트릭스 |
| [analysis/ 디렉토리와 설계문서 §참조](snippets/architecture/analysis-design-docs.md) | §참조 코드-문서 연결 체계 |
| [푸시다운 사전 조건](snippets/architecture/pushdown-preconditions.md) | Warning 결합도별 타이밍 분류 |

---

## 📦 패키지 가이드 (10)

| 아티클 | 핵심 개념 |
|--------|----------|
| [EchoLoader](snippets/packages/echo-loader.md) | 부팅 진입점, 초기화 순서 |
| [EchoScanner](snippets/packages/echo-scanner.md) | 획득 로직 메인 패키지 |
| [EchoRoot](snippets/packages/echo-root.md) | 시스템 루트, 워치독 |
| [EchoMeasure](snippets/packages/echo-measure.md) | 측정값 관리 |
| [EchoWorksheet](snippets/packages/echo-worksheet.md) | 임상 보고서 |
| [EchoFrontPanel](snippets/packages/echo-frontpanel.md) | 하드웨어 패널 입력 |
| [EchoConfig](snippets/packages/echo-config.md) | 설정/프리셋 관리 |
| [EchoSysMon](snippets/packages/echo-sysmon.md) | 시스템 모니터링 |
| [EchoDecisionSupport](snippets/packages/echo-decision-support.md) | 임상 의사결정 지원 |
| [GcViewer/ObjectViewer](snippets/packages/gc-viewer-object-viewer.md) | 렌더링 오브젝트 계층 |

---

## ⚙️ 워크플로 (8)

| 아티클 | 핵심 개념 |
|--------|----------|
| [GE CONFIDENTIAL 헤더 추가](snippets/workflow/ge-confidential-header.md) | 신규 파일 체크리스트 |
| [리팩토링 PR 체크리스트](snippets/workflow/refactoring-pr-checklist.md) | 추적성 + 빌드 + 테스트 |
| [새 OVObject 추가 절차](snippets/workflow/add-new-ovobject.md) | 팩토리 태그 등록 순서 |
| [버그 추적 흐름](snippets/workflow/bug-trace-flow.md) | RBUG → 코드 → 로그 역추적 |
| [설계문서 Phase 업데이트](snippets/workflow/design-doc-phase-update.md) | analysis/ 문서 §갱신 절차 |
| [보안 감사 로그](snippets/workflow/security-audit-log.md) | AuditLogUserEvent 작성 규칙 |
| [Debug vs Release 빌드 차이](snippets/workflow/debug-vs-release-build.md) | RuntimeChecks, DEBUG_NEW |
| [GoogleTest Fixture 작성](snippets/workflow/write-googletest-fixture.md) | Require_that_* 명명 + 배치 |

---

## 📖 읽기 순서 추천

### 1단계: 전체 그림 (15분)
1. [Layer A vs Layer B](snippets/architecture/layer-a-vs-layer-b.md)
2. [GC 파라미터 버스](snippets/architecture/gc-parameter-bus.md)
3. [Echo* 패키지 분류](snippets/architecture/echo-package-taxonomy.md)

### 2단계: 도메인 (20분)
4. [SWE 개요](snippets/domain/swe-shear-wave-elastography.md)
5. [UGAP 개요](snippets/domain/ugap-ultrasound-guided-attenuation-parameter.md)
6. [간 경고 3종](snippets/domain/liver-warning-triad.md)
7. [상태 머신](snippets/domain/acq-assist-state-machine.md)

### 3단계: 코딩 규칙 (10분)
8. [명명 규칙](snippets/cpp/member-naming-conventions.md)
9. [에러 처리](snippets/cpp/bool-out-param-error-handling.md)
10. [티켓 ID 주석](snippets/cpp/ticket-id-comments.md)

### 4단계: 진행 중인 리팩토링 (15분)
11. [LiverWarning3Panel 추출](snippets/architecture/composition-over-inheritance-liverwarning3panel.md)
12. [Push-down 전략](snippets/architecture/pushdown-vs-hoisting-strategy.md)
13. [P0b 테스트 전략](snippets/architecture/p0b-unit-test-strategy.md)
14. [Director 확인 질문](snippets/architecture/director-signoff-questions.md)

---

## 🔖 컨텍스트

- **제품**: GE Healthcare Vivid 시리즈 초음파 진단기기
- **코드베이스**: `gipc-app` (~4.9GB, ~40k C++ 파일)
- **기술 스택**: C++11/14, MSVC, MFC, COM, ATL, GoogleTest
- **규제 환경**: FDA, IEC 62304 (의료기기 소프트웨어)
- **레거시 기간**: 26년 (GE + Vingmed 인수)

---

*inspired by [30 seconds of code](https://github.com/30-seconds/30-seconds-of-code)*
