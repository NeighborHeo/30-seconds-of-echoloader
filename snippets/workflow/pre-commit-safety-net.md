---
title: "커밋 전 안전성 체크: CI 없는 코드베이스에서 혼자 지키는 안전망"
category: workflow
tags: [pre-commit, build, safety, iec-62304, gating, dependency-check]
difficulty: beginner
---

gipc-app에는 CI 파이프라인이 없다. 아래 체크리스트를 직접 실행해야 팀 전체 빌드를 지킬 수 있다.

## 언제 이 아티클을 보나

- 코드 수정 후 커밋하기 직전
- "내 변경이 다른 사람 빌드를 깼나?" 확인할 때
- 새 기능 또는 아키텍처 변경 후 영향 범위를 점검할 때

## 필수 체크 (커밋 전 반드시)

```
[ ] 1. Debug 빌드 통과
    명령: Visual Studio → Build → Build Solution (Debug)
    왜: Debug에서만 활성화되는 런타임 검사(_DEBUG, DEBUG_NEW, EnableFastChecks)가
        Release에서는 꺼져 있어 버그를 놓침.

[ ] 2. Release 빌드 통과
    명령: Visual Studio → Build → Build Solution (Release)
    왜: Release 최적화 설정이 다르기 때문에 Debug에서 없던 경고/에러가 새로 생길 수 있음.

[ ] 3. /W4 경고 없음 확인
    방법: 출력 창 → "0 warning(s)" 확인
    왜: /W4 경고는 대부분 실제 버그 또는 미정의 동작을 가리킴.
    흔한 위반:
      C4100 — 미사용 파라미터 (함수 시그니처와 구현 불일치)
      C4018 — signed/unsigned 비교
      C4244 — 정밀도 손실 (double → float 등 암묵적 변환)

[ ] 4. 역방향 의존 검사
    명령:
      grep -r "include.*EchoScanner" src/GcViewer/ --include="*.h"
      grep -r "include.*EchoScanner" src/GcViewer/ --include="*.cpp"
      grep -r "include.*GcViewer"    src/EchoScanner/ --include="*.h"
      grep -r "include.*GcViewer"    src/EchoScanner/ --include="*.cpp"
    왜: 역방향 #include가 생기면 빌드 순환이 발생해 팀 전체 빌드가 블로킹됨.
        결과가 한 줄이라도 나오면 즉시 수정한다.

[ ] 5. 수정한 함수의 모든 호출처 확인
    방법: VS에서 함수명 우클릭 → "Find All References"
    왜: 함수 시그니처를 변경했을 때 호출처를 하나라도 놓치면
        링커 에러 또는 의도하지 않은 동작 변경이 생김.
    특히 주의: IAcqAssistHandler 가상 함수를 변경했다면
               SWE, UGAP 양쪽 구현체를 모두 확인한다.

[ ] 6. 게이팅 확인 (AA 관련 변경 시)
    확인 방법:
      grep -n "Manager::Instance" src/EchoScanner/ESMain.cpp
      → 새로 추가한 호출 사이트에 InElasto() 또는 IsInUGAPMode() 게이트가 있는지 확인
    왜: 게이팅이 없으면 잘못된 모드에서 코드가 실행되어 임상 데이터가 오염됨.
        이 항목은 타협 불가.

[ ] 7. GC 파라미터 키 상수화 확인
    확인 방법:
      grep -rn "SetParameterValue(\"" src/ --include="*.cpp"
      grep -rn "SetParameter.*\"" src/ --include="*.cpp"
    왜: 문자열 리터럴 오타는 컴파일 오류를 내지 않음.
        SetParameter()가 키를 찾지 못하면 아무 로그 없이 값이 사일런트 드롭됨.
        결과가 나오면 constexpr 상수로 교체한다.
```

## 추가 체크 (영향 범위가 넓을 때)

```
[ ] 8. IEC 62304 추적성 — 변경 이유가 티켓에 기록됐나?
    확인: RBUG/FBUG/CR- 번호가 코드 주석 또는 커밋 메시지에 포함됐는지 확인.
    왜: 의료기기 소프트웨어는 모든 변경에 요구사항 추적이 필요함.
        누락 시 규정 감사에서 지적 대상이 됨.

[ ] 9. 보안 이벤트를 추가했다면 AuditLog 확인
    확인:
      grep -rn "AuditLogUserEvent" src/
      → 기존 패턴을 보고 새 이벤트가 같은 형식으로 기록되는지 확인.
    왜: FDA 감사 시 보안 이벤트 누락은 심각한 규정 위반으로 처리됨.

[ ] 10. 새 파일을 추가했다면 GE CONFIDENTIAL 헤더 확인
    확인: 파일 상단에 GE CONFIDENTIAL 블록이 있는지 확인.
    왜: IP 보호 및 코드 출처 추적 요건. (ge-confidential-header.md 참고)
```

## 빠른 셀프 리뷰 (diff를 보면서)

```
내가 추가한 #include 목록:
  → 포함하는 파일의 레이어 ≥ 포함되는 파일의 레이어인가?
    (GcViewer가 EchoScanner를 포함하거나, EchoScanner가 GcViewer를 포함하면 역방향)
  → 새 외부 의존이 추가됐나? 꼭 필요한가?

내가 변경한 함수:
  → ESMain, IAcqAssistHandler, OVObject 가상 함수 중 하나인가?
  → 그렇다면 "외부 컨트랙트" — 변경 전 아키텍처 리뷰 필수.

내가 추가한 상태 변수:
  → Render()와 SetParameter() 양쪽에서 접근하나?
  → 그렇다면 std::atomic 또는 ScCommon::Mutex로 보호했나?
```

## 체크리스트 소요 시간 기준

```
빠른 버그 수정:  항목 1~6 필수 (약 10분)
새 기능 추가:   항목 1~8 필수 (약 20분)
아키텍처 변경:  항목 1~10 + 아키텍처 리뷰 (약 1시간)
```

## Key Points

- CI가 없으므로 빌드 통과는 최소 안전망 — Debug + Release 둘 다 반드시 확인
- 역방향 의존 검사는 `grep` 네 줄로 끝남 — 결과가 한 줄이라도 나오면 즉시 수정
- 게이팅 확인을 건너뛰면 임상 데이터 오염 위험 — 타협 불가
- 문자열 리터럴 직접 사용은 오타 버그 → `constexpr` 상수화가 유일한 방어선
- IEC 62304 요구사항 추적은 선택이 아님 — 규정 위반 시 제품 리콜 가능
