# 30 Seconds of EchoLoader

> **gipc-app** 코드베이스를 30초 안에 이해하는 레퍼런스 카드 모음

GE Healthcare 초음파 진단기기(Vivid 시리즈) 소프트웨어 `gipc-app`에 대한 개발 지식, 도메인 지식, 아키텍처 지식을 **30초 안에 읽을 수 있는 아티클** 형식으로 정리했습니다. [30 seconds of code](https://30secondsofcode.org) 형식에서 영감을 받았습니다.

---

## 📦 카테고리

| 카테고리 | 수 | 설명 |
|---------|-----|------|
| [시스템 구조도](snippets/system/) | 6 | 리소스 맵, 스레드 모델, 버튼→렌더 흐름, 방송국 비유 |
| [UDT / GC 프레임워크](snippets/udt/) | 9 | UDT 핵심, GC 디스패치, OVObject 협력·합성 패턴 |
| [C++ 패턴](snippets/cpp/) | 17 | gipc-app에서 실제로 쓰이는 C++ 관용구·컨벤션 |
| [고급 C++ 패턴](snippets/cpp-patterns/) | 23 | PIMPL, 이동 시맨틱, 16ms 프레임 버짓, atomic, 오브젝트 풀 |
| [디자인 패턴](snippets/patterns/) | 6 | Observer, NVI, Strategy, Facade, CRTP, RAII |
| [아키텍처](snippets/architecture/) | 21 | 설계 철학, 실패 모드 지도, 26년 코드베이스 지형, 코드 배치 결정 |
| [도메인 지식](snippets/domain/) | 15 | 초음파 물리, SWE, UGAP, IEC 62304, DICOM |
| [패키지 가이드](snippets/packages/) | 11 | Echo* 패키지별 역할, 의존 관계, 주의사항 |
| [워크플로](snippets/workflow/) | 16 | 코드 탐색법, ESMain 탐색, GC 파라미터 개발, 커밋 전 안전망 |

**총 124개 아티클**

---

## 🗺 시스템 구조도 (6) — 여기부터 시작

| 아티클 | 핵심 개념 |
|--------|----------|
| [방송국 비유로 이해하는 gipc-app](snippets/system/mental-model-broadcast-system.md) | **가장 먼저** — 전체 구조를 비유로 즉시 이해 |
| [gipc-app 전체 리소스 구조도](snippets/system/gipc-app-resource-map.md) | 4개 레이어 + 데이터 흐름 다이어그램 |
| [스레드 모델: 어느 코드가 어느 스레드에](snippets/system/gipc-app-threading-model.md) | 4개 스레드 규칙, SetParameter() 스레드 위치 |
| [버튼 → 화면 렌더링까지 완전한 흐름](snippets/system/user-action-to-render-flow.md) | 사용자 입력 → GC 버스 → Render() 시퀀스 다이어그램 |
| [GrandCentral 객체 수명주기](snippets/system/gc-object-lifecycle.md) | OnActivate → Active → OnDeactivate 상태도 |
| [EchoLoader 시작 시퀀스](snippets/system/echoloader-startup-sequence.md) | DLL 로드 순서, Phase별 초기화, 셧다운 역순 |

---

## 🧩 UDT / GC 프레임워크 (9)

| 아티클 | 핵심 개념 |
|--------|----------|
| [UDT란 무엇인가](snippets/udt/what-is-udt.md) | GrandCentral UDT 개념 |
| [GcUdtHandle 수명주기](snippets/udt/gc-udt-handle-lifetime.md) | 비소유 핸들 패턴 |
| [UDT vs GC 파라미터](snippets/udt/udt-vs-gc-params.md) | 두 통신 채널 비교 |
| [OVObject as UDT 노드](snippets/udt/ovobject-as-udt-node.md) | 팩토리 등록과 UDT 그래프 |
| [GC 파라미터 버스 디스패치](snippets/udt/gc-parameter-dispatch.md) | prefix 매칭 → SetParameter() 3단계 체인 |
| [OVObject 간 간접 협력](snippets/udt/udt-multi-object-coordination.md) | 직접 포인터 금지 → 파라미터 버스 경유 |
| [OVObject 합성 패턴](snippets/udt/ovobject-composition-pattern.md) | 다중 상속 불가 → has-a 헬퍼 클래스 |
| [GC Variant 타입 디스패치](snippets/udt/gc-variant-type-dispatch.md) | tagged-union 수신측 안전 패턴 |
| [UDT 레지스트리 조회 안전 체인](snippets/udt/udt-registry-lookup.md) | Get() → IsValid() → RAII 해제 |

---

## 🎨 디자인 패턴 (6)

| 아티클 | 핵심 개념 |
|--------|----------|
| [Observer: GC 파라미터 버스](snippets/patterns/observer-gc-param-bus.md) | SetParameterValue publish / OVObject::SetParameter subscribe |
| [NVI: AcquisitionAssistant 계층](snippets/patterns/nvi-pattern-in-acqassist.md) | public OnFrame() → private virtual DoOnFrame() |
| [Strategy: HandlerType 선택](snippets/patterns/strategy-pattern-handler-type.md) | IAcqAssistHandler + SWE/UGAP 전략 교환 |
| [Facade: Manager 클래스](snippets/patterns/facade-pattern-manager.md) | ESMain이 내부 구조를 모르게 격리 |
| [CRTP 정적 다형성](snippets/patterns/crtp-static-polymorphism.md) | vtable 없는 Render() 핫 패스 |
| [RAII: ScCommon::ScopedLock](snippets/patterns/raii-scoped-lock-patterns.md) | ScopedLock / GcUdtHandle / FrameBuffer 3패턴 |

---

## 🗂 C++ 패턴 (17)

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
| [전방 선언 규칙](snippets/cpp/forward-declaration-rules.md) | .h에서 forward decl, .cpp에서 #include |
| [constexpr 컴파일 타임 상수](snippets/cpp/constexpr-compile-time-constants.md) | namespace GcKeys, C++11 단일-return 제약 |

---

## 🔬 고급 C++ 패턴 (23)

| 아티클 | 핵심 개념 |
|--------|----------|
| [PIMPL 이디엄](snippets/cpp-patterns/pimpl-idiom.md) | unique_ptr Impl, 컴파일 경계 축소 |
| [타입 소거: Variant](snippets/cpp-patterns/type-erasure-variant.md) | tagged-union, GC 파라미터 버스 타입 시스템 |
| [이동 시맨틱: 프레임 버퍼](snippets/cpp-patterns/move-semantics-frame-buffer.md) | std::move 소유권 이전, moved-from 상태 |
| [Const 정확성: Render 파이프라인](snippets/cpp-patterns/const-correctness-render.md) | Render() const, mutable 정당 사용 |
| [const 정확성 패턴](snippets/cpp-patterns/const-correctness.md) | const 멤버 함수, const& 매개변수 |
| [COM/ATL HRESULT 처리](snippets/cpp-patterns/com-atl-hresult.md) | COM 인터페이스 에러 처리 |
| [GcUdtHandle 사용법](snippets/cpp-patterns/gc-udt-handle.md) | UDT 비소유 참조 핸들 |
| [MFC 다이얼로그 클래스](snippets/cpp-patterns/mfc-dialog-class.md) | CDialog 서브클래스 패턴 |
| [OVObject 팩토리 등록](snippets/cpp-patterns/ovobject-factory-registration.md) | Gc.* 태그 등록 방법 |
| [ScCommon Thread 래퍼](snippets/cpp-patterns/sccommon-thread-wrapper.md) | GE 스레드 풀 래퍼 |
| [싱글톤 Manager::Instance()](snippets/cpp-patterns/singleton-manager-instance.md) | 스레드 안전 싱글톤 |
| [상태머신 enum + switch](snippets/cpp-patterns/state-machine-enum-switch.md) | AcqAssistState 패턴 |
| [OVGraphics 렌더 객체](snippets/cpp-patterns/ovgraphics-render-objects.md) | 렌더링 객체 계층 |
| [Frame 구조체](snippets/cpp-patterns/frame-struct.md) | 프레임 데이터 구조 |
| [Debug 파라미터 패턴](snippets/cpp-patterns/debug-params-pattern.md) | 디버그 전용 파라미터 |
| [세 가지 로깅 시스템](snippets/cpp-patterns/three-logging-systems.md) | ScLog vs AuditLog vs Trace |
| [COM 인터페이스 in gipc](snippets/cpp-patterns/com-interface-in-gipc.md) | gipc-app COM 사용 패턴 |
| [Variant 타입](snippets/cpp-patterns/variant-type.md) | GC 파라미터 Variant 상세 |
| [16ms 프레임 버짓 관리](snippets/cpp-patterns/frame-budget-16ms.md) | Render() ≤8ms, ScCommon::Timer 오버런 감지 |
| [오브젝트 풀: 프레임 버퍼](snippets/cpp-patterns/object-pool-frame-buffer.md) | FramePool + ScopedFrame RAII |
| [Lock-free 상태 플래그](snippets/cpp-patterns/lock-free-status-flag.md) | std::atomic<bool/int>, memory_order 쌍 |
| [GC 파라미터 키 문자열 캐싱](snippets/cpp-patterns/string-key-cache-pattern.md) | static const → 프레임당 alloc 제거 |
| [렌더 스레드 격리 패턴](snippets/cpp-patterns/render-thread-isolation.md) | async AI 추론 + 더블버퍼 atomic swap |

---

## 🏗 아키텍처 (20)

| 아티클 | 핵심 개념 |
|--------|----------|
| [Layer A vs Layer B](snippets/architecture/layer-a-vs-layer-b.md) | EchoScanner vs GcViewer 분리 |
| [GrandCentral 프레임워크](snippets/architecture/gc-framework-overview.md) | UDT 객체 그래프 + 파라미터 버스 |
| [ESMain 내부 구조](snippets/architecture/esmain-internals.md) | 11개 호출 사이트 + 게이팅 술어 |
| [ESMain 진입점](snippets/architecture/esmain-as-system-hub.md) | 시스템 허브로서의 ESMain |
| [GC 파라미터 버스](snippets/architecture/gc-parameter-bus.md) | 문자열 키/값 메시지 버스 |
| [GC 파라미터 키 네이밍](snippets/architecture/gc-param-key-naming.md) | "Gc." 접두사 규칙 |
| [Manager-Factory-Handler 패턴](snippets/architecture/manager-factory-handler-pattern.md) | 3-파트 AA 패턴 |
| [OVObject 상속](snippets/architecture/ovobject-inheritance.md) | 렌더링 기반 클래스 |
| [OVObject 렌더 루프](snippets/architecture/ovobject-render-loop.md) | Render() 호출 주기와 제약 |
| [패키지 의존 방향 규칙](snippets/architecture/package-dependency-rules.md) | 4개 레이어 허용/금지 방향 |
| [Echo* 패키지 분류](snippets/architecture/echo-package-taxonomy.md) | ~50개 패키지 역할 |
| [외부 컨트랙트 - 절대 건드리지 말 것](snippets/architecture/external-contracts-dont-touch.md) | 4개 load-bearing 지점 |
| [프론트 패널 → ESMain 라우팅](snippets/architecture/front-panel-to-esmain.md) | 사용자 입력 흐름 |
| [사용자 입력 이벤트 라우팅](snippets/architecture/user-input-event-routing.md) | 버튼/노브 이벤트 처리 |
| [프리셋 로딩 파이프라인](snippets/architecture/preset-loading-pipeline.md) | EchoConfig 프리셋 적용 흐름 |
| [RAII 셧다운 순서](snippets/architecture/raii-shutdown-order.md) | 역순 소멸자 = 안전한 셧다운 |
| [ScCommon 라이브러리 지도](snippets/architecture/sccommon-library-map.md) | 10개 핵심 헤더 맵 |
| [왜 GC 파라미터 버스인가](snippets/architecture/why-gc-param-bus.md) | 대안 3가지를 거부한 이유 + 실제 비용 솔직 분석 |
| [실패 모드 지도](snippets/architecture/failure-modes-map.md) | 6가지 실패 패턴 — 증상→원인→진단→수정 룩업 테이블 |
| [26년 코드베이스 지도](snippets/architecture/legacy-modern-code-boundary.md) | 지뢰밭(ESMain, MFC, COM) vs 안전 지대(OVObject, Handler) |
| [새 코드는 어디에 짜나](snippets/architecture/where-does-new-code-go.md) | Layer A/B 결정 트리 — 기능 받자마자 첫 번째로 여는 문서 |

---

## 🏥 도메인 지식 (15)

| 아티클 | 핵심 개념 |
|--------|----------|
| [초음파 이미지 파이프라인](snippets/domain/ultrasound-image-pipeline.md) | RF→빔포밍→포락선→스캔변환→표시 |
| [SWE - 전단파 탄성 초음파](snippets/domain/swe-shear-wave-elastography.md) | 간 섬유화 비침습 측정 |
| [UGAP - 초음파 감쇠 파라미터](snippets/domain/ugap-ultrasound-guided-attenuation-parameter.md) | 지방간 비침습 측정 |
| [IEC 62304 소프트웨어 안전 등급](snippets/domain/iec-62304-sw-class.md) | Class A/B/C 실무 영향 |
| [DICOM SR 측정값 구조](snippets/domain/dicom-sr-measurement.md) | SR 트리, UCUM 코드, 규정 준수 |
| [Watchdog & 안전 모니터](snippets/domain/watchdog-and-safety-monitor.md) | 렌더링 루프 Watchdog + fail-safe |
| [간 경고 3종](snippets/domain/liver-warning-triad.md) | LargeSCD + PoorProbe + ObliqueCapsule |
| [Acquisition Assistant 상태 머신](snippets/domain/acq-assist-state-machine.md) | AcqAssistState 5개 상태 |
| [ROI Processor 개념](snippets/domain/roi-processor-concept.md) | SWEROIProcessor vs ACResults |
| [프로브 접촉 품질 지시자](snippets/domain/probe-contact-quality-indicator.md) | ProbeContactQualityIndicator |
| [LiverAI Processor](snippets/domain/liver-ai-processor.md) | AI 모델 파이프라인 |
| [피부-캡슐 거리 (SCD)](snippets/domain/skin-to-capsule-distance.md) | SCD 측정 및 경고 조건 |
| [ROI 좌표계](snippets/domain/roi-coordinate-system.md) | 화면 좌표 vs 물리 좌표 |
| [워치독 & 치명 오류 핸들러](snippets/domain/watchdog-crash-handler.md) | FatalErrorHandler + WatchdogCB |
| [감사 로그 사용자 이벤트](snippets/domain/audit-log-user-event.md) | AuditLogUserEvent + FDA 규정 |

---

## 📦 패키지 가이드 (11)

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
| [UDT와 GcUdtHandle](snippets/udt/what-is-udt.md) | UDT 핵심 개념 |

---

## ⚙️ 워크플로 (12)

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
| [Windows Minidump 분석](snippets/workflow/crash-dump-analysis.md) | WinDbg 시퀀스, 3대 크래시 패턴 |
| [GC 파라미터 흐름 디버깅](snippets/workflow/gc-param-debug-trace.md) | 5단계 체크리스트, 키 대소문자 함정 |
| [C++11 수동 Mock 패턴](snippets/workflow/mock-interface-for-testing.md) | CallRecorder + Seam 패턴 |
| [빌드 에러 TOP 5](snippets/workflow/common-build-errors.md) | C4100/C2664/LNK2019/C4996/C1083 |
| [40k 파일 코드 탐색법](snippets/workflow/codebase-search-strategies.md) | 상황별 grep 명령 — 6가지 시나리오 |
| [ESMain.cpp 10,000줄 탐색법](snippets/workflow/esmain-navigation.md) | 구역 지도 + 작업별 grep + VS 단축키 |
| [GC 파라미터 기능 개발 전체 절차](snippets/workflow/gc-param-feature-development.md) | 키 설계→발행→수신→Render→디버깅 6단계 |
| [커밋 전 안전성 체크](snippets/workflow/pre-commit-safety-net.md) | CI 없는 코드베이스의 10분 안전망 |

---

## 📖 읽기 순서 추천

### 1단계: 전체 그림 (20분)
1. [방송국 비유로 이해하는 gipc-app](snippets/system/mental-model-broadcast-system.md) ← **여기서 시작 (비유로 즉시 파악)**
2. [gipc-app 전체 리소스 구조도](snippets/system/gipc-app-resource-map.md)
3. [버튼 → 화면 렌더링까지 완전한 흐름](snippets/system/user-action-to-render-flow.md)
4. [왜 GC 파라미터 버스인가](snippets/architecture/why-gc-param-bus.md) ← **설계 이유를 알면 사용법이 납득됨**

### 2단계: 코딩 규칙 (10분)
5. [명명 규칙](snippets/cpp/member-naming-conventions.md)
6. [에러 처리](snippets/cpp/bool-out-param-error-handling.md)
7. [세 가지 로깅 시스템](snippets/cpp-patterns/three-logging-systems.md)

### 3단계: 실전 개발 준비 (20분)
8. [새 코드는 어디에 짜나](snippets/architecture/where-does-new-code-go.md) ← **기능 받을 때마다**
9. [40k 파일 코드 탐색법](snippets/workflow/codebase-search-strategies.md)
10. [ESMain.cpp 10,000줄 탐색법](snippets/workflow/esmain-navigation.md)
11. [GC 파라미터 기능 개발 전체 절차](snippets/workflow/gc-param-feature-development.md)
12. [커밋 전 안전성 체크](snippets/workflow/pre-commit-safety-net.md) ← **커밋 전마다**

### 4단계: 깊은 이해 (30분)
13. [스레드 모델: 어느 코드가 어느 스레드에](snippets/system/gipc-app-threading-model.md) ← **레이스 컨디션 예방**
14. [왜 GC 파라미터 버스인가](snippets/architecture/why-gc-param-bus.md)
15. [실패 모드 지도](snippets/architecture/failure-modes-map.md) ← **디버깅 막힐 때**
16. [26년 코드베이스 지도](snippets/architecture/legacy-modern-code-boundary.md)

---

## 🔖 컨텍스트

- **제품**: GE Healthcare Vivid 시리즈 초음파 진단기기
- **코드베이스**: `gipc-app` (~4.9GB, ~40k C++ 파일)
- **기술 스택**: C++11/14, MSVC, MFC, COM, ATL, GoogleTest
- **규제 환경**: FDA, IEC 62304 (의료기기 소프트웨어)
- **레거시 기간**: 26년 (GE + Vingmed 인수)

---

*inspired by [30 seconds of code](https://github.com/30-seconds/30-seconds-of-code)*
