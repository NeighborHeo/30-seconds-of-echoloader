# 30 Seconds of EchoLoader

> **gipc-app** 코드베이스를 30초 안에 이해하는 레퍼런스 카드 모음

GE Healthcare 초음파 진단기기(Vivid 시리즈) 소프트웨어 `gipc-app`에 대한 개발 지식, 도메인 지식, 아키텍처 지식을 **30초 안에 읽을 수 있는 아티클** 형식으로 정리했습니다. [30 seconds of code](https://30secondsofcode.org) 형식에서 영감을 받았습니다.

---

## 📦 카테고리

| 카테고리 | 수 | 설명 |
|---------|-----|------|
| [C++ 패턴](snippets/cpp/) | 10 | gipc-app에서 실제로 쓰이는 C++ 관용구·컨벤션 |
| [도메인 지식](snippets/domain/) | 10 | 초음파 의료 영상, SWE, UGAP, 간 측정 개념 |
| [아키텍처](snippets/architecture/) | 10 | 패키지 구조, 설계 패턴, 레이어 분리 전략 |

---

## 🗂 C++ 패턴

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

---

## 🏥 도메인 지식

| 아티클 | 핵심 개념 |
|--------|----------|
| [SWE - 전단파 탄성 초음파](snippets/domain/swe-shear-wave-elastography.md) | 간 섬유화 비침습 측정 |
| [UGAP - 초음파 감쇠 파라미터](snippets/domain/ugap-ultrasound-guided-attenuation-parameter.md) | 지방간 비침습 측정 |
| [간 경고 3종](snippets/domain/liver-warning-triad.md) | LargeSCD + PoorProbe + ObliqueCapsule |
| [Acquisition Assistant 상태 머신](snippets/domain/acq-assist-state-machine.md) | AcqAssistState 5개 상태 |
| [Acquisition Assistant 단계(Phase) 생명주기](snippets/domain/acq-assist-phase-lifecycle.md) | Phase enum vs ScanState |
| [AutoPos 키 해석](snippets/domain/autopos-key-resolution.md) | SWE vs UGAP 키 차이 |
| [ROI Processor 개념](snippets/domain/roi-processor-concept.md) | SWEROIProcessor vs ACResults |
| [프로브 접촉 품질 지시자](snippets/domain/probe-contact-quality-indicator.md) | ProbeContactQualityIndicator |
| [LiverAI Processor](snippets/domain/liver-ai-processor.md) | AI 모델 파이프라인 |
| [피부-캡슐 거리 (SCD)](snippets/domain/skin-to-capsule-distance.md) | SCD 측정 및 경고 조건 |

---

## 🏗 아키텍처

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

---

## 📖 사용 방법

각 아티클은 독립적으로 읽을 수 있습니다. 처음이라면 이 순서를 추천합니다:

1. **아키텍처부터** → Layer A/B, Manager-Factory-Handler, Echo 패키지 분류
2. **도메인 이해** → SWE/UGAP 개념, 간 경고 3종, 상태 머신
3. **코딩 규칙** → 명명 규칙, 에러 처리, 로깅, 추적성

---

## 🔖 컨텍스트

- **제품**: GE Healthcare Vivid 시리즈 초음파 진단기기
- **코드베이스**: `gipc-app` (~4.9GB, ~40k C++ 파일)
- **기술 스택**: C++11/14, MSVC, MFC, COM, ATL, GoogleTest
- **규제 환경**: FDA, IEC 62304 (의료기기 소프트웨어)
- **레거시 기간**: 26년 (GE + Vingmed 인수)
- **아키텍처 문서**: `analysis/SWEUGAPAcqAssist/` 디렉토리

---

*inspired by [30 seconds of code](https://github.com/30-seconds/30-seconds-of-code)*
