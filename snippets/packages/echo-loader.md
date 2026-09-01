---
title: "EchoLoader — 애플리케이션 엔트리포인트"
category: packages
tags: [bootstrap, dependency-injection, orchestration]
difficulty: advanced
---

gipc-app 전체 패키지를 순서대로 로드하고, 의존성 주입 컨테이너를 구성하는 최상위 오케스트레이터. 이 레포 이름("echoloader")의 유래.

## 역할

- 패키지 로딩 순서(EchoRoot → EchoConfig → EchoScanner → … ) 제어
- 의존성 주입(DI) 컨테이너 초기화 및 서비스 바인딩
- 애플리케이션 라이프사이클(Init → Run → Shutdown) 진입점

## 위치

```
src/packages/EchoLoader/
├── EchoLoaderPck.h      ← 패키지 헤더, 로드 순서 상수
├── EchoLoaderPck.cpp    ← Init/Shutdown 구현
└── main.cpp             ← WinMain / 진입점
```

## 핵심 클래스

| 클래스 | 역할 |
|---|---|
| `EchoLoaderPck` | 패키지 등록·로드 오케스트레이터 |
| `EchoAppContext` | DI 컨테이너, 서비스 로케이터 루트 |

## 의존 관계

```
EchoLoader
  → EchoRoot        (부트스트랩/워치독)
  → EchoConfig      (프리셋·설정)
  → EchoScanner     (스캐닝 엔진)
  → GcViewer        (렌더링)
  → EchoMeasure     (측정)
  → EchoWorksheet   (리포트)
  ← (최상위, 아무도 EchoLoader를 import하지 않음)
```

## 주의사항

- 로드 순서 변경 시 순환 의존 발생 가능 — 반드시 의존 그래프 확인 후 수정
- `main.cpp` 직접 수정은 최소화; 새 서비스는 DI 컨테이너 등록 패턴 사용
- MFC 메시지 루프와 WinMain 진입이 여기서 시작됨 — COM 초기화(CoInitialize) 선행 필수
