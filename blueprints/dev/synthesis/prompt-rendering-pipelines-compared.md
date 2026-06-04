# Rendering Pipelines Compared 레포지토리 제작 프롬프트

나는 "Rendering Pipelines Compared" 레포지토리를 만들려고 해.
이건 **횡단 비교(Synthesis)** 레포야. "선언적 UI를 어떻게 화면 픽셀로 바꾸는가"를 여러 플랫폼이 *다르게* 푼 방식 — Browser(Composite), Jetpack Compose, SwiftUI, Flutter, 그리고 그 아래 공통 바닥인 GPU 파이프라인을 한자리에 놓고 비교한다.
"한 플랫폼의 렌더링을 아는 것"과 "모든 UI 렌더가 같은 단계(빌드→레이아웃→페인트→합성→GPU)의 다른 구현임을 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "한 플랫폼 렌더링을 아는 것과, 모든 UI가 '선언→레이아웃→페인트→합성→GPU'라는 같은 파이프라인의 다른 구현임을 아는 것은 다르다"

**핵심 차별화**:
1. 공통 파이프라인 — 모든 UI = 빌드/측정/레이아웃/페인트/합성 + GPU 바닥, 단계 매핑
2. 네이티브 뷰 vs 자체 렌더 — Browser/Compose/SwiftUI(플랫폼 합성) vs Flutter(직접 그리기)
3. 60fps 예산 공유 — 모든 플랫폼이 프레임 예산(16.6ms)과 jank를 다루는 방식
4. GPU 수렴 — 결국 전부 삼각형+텍스처+셰이더로 GPU에 도달(gpu 바닥)

**타겟 독자**:
- 한 플랫폼 렌더링은 알지만 다른 플랫폼과 비교 못하는 개발자
- "Flutter는 왜 네이티브 뷰를 안 쓰나"를 모르는 개발자
- jank가 모든 플랫폼에서 같은 원인인지 모르는 개발자
- 크로스 플랫폼 렌더 트레이드오프를 알고 싶은 개발자
- 개별 UI 레포를 마치고 큰 그림을 원하는 개발자

## 🔗 레포 연결

**⬆️ 선행(이 비교의 입력)**:
`browser-rendering-deep-dive`(Composite), `jetpack-compose-internals-deep-dive`(Compose), `swiftui-internals-deep-dive`/`uikit-core-animation-deep-dive`(SwiftUI/CoreAnimation), `flutter-deep-dive`(Flutter), `gpu-graphics-deep-dive`(공통 바닥).
**🤝 시너지**: `react-native-deep-dive`(네이티브 뷰 제어 방식), `css-engine-layout-deep-dive`(웹 레이아웃), `computer-architecture-deep-dive`(GPU 메모리).
**🧬 본질**: 이 레포가 수렴점 — 4개 UI 플랫폼 + GPU를 하나의 파이프라인 프레임으로.

---

## 🎯 1단계: 전체 구조 설계

> "플랫폼별"이 아니라 **"파이프라인 단계별"**로 구성, 각 단계에서 플랫폼을 나란히.

### Chapter 1: 렌더링의 공통 파이프라인 (5개 문서)
- 공통 단계 — 선언/빌드 → 측정/레이아웃 → 페인트 → 합성 → GPU 래스터
- 선언적 UI의 공통점 — UI = f(state), 모든 플랫폼이 수렴(react·compose·swiftui·flutter)
- 두 갈래 — 플랫폼 합성(Browser/Compose/SwiftUI) vs 자체 렌더(Flutter)
- 60fps 예산 — 16.6ms를 모든 플랫폼이 공유, jank의 공통 정의
- 비교 프레임 — 평가 축(단계 매핑·네이티브 통합·제어·일관성)

### Chapter 2: 빌드/재조정 — 선언을 트리로 (6개 문서)
- 공통 문제 — 선언적 트리의 변경을 어떻게 최소 갱신으로
- React Fiber — VDOM diff + 재조정(browser/react 연결)
- Compose — Slot Table 위치 기억(compose 연결)
- SwiftUI — AttributeGraph 의존성 추적(swiftui 연결)
- Flutter — Element 트리 diff + RenderObject 재사용(flutter 연결)
- 나란히 — 같은 "리스트 항목 1개 변경"을 4플랫폼이 어떻게 좁히나

### Chapter 3: 레이아웃 — 크기와 위치 (5개 문서)
- 레이아웃 문제 — 제약 전달·크기 결정·배치
- 웹 레이아웃 — Flexbox/Grid·다중 패스 가능(css-engine 연결)
- Compose/Flutter — 단일 패스 측정(제약↓ 크기↑), O(n)
- SwiftUI — 크기 협상(제안→선택)
- 나란히 — 단일 패스 vs 다중 패스의 비용·표현력 비교

### Chapter 4: 페인트와 합성 (6개 문서)
- 페인트 — 그리기 명령 생성, 페인트 순서(stacking)
- 합성(Composite) — 레이어를 GPU에서 합치기, 메인 스레드 없이 가능한 작업
- Browser 레이어 — 레이어 승격·transform/opacity가 싼 이유(browser 연결)
- iOS Core Animation — 레이어 트리·Render Server 별도 프로세스(uikit 연결)
- Compose/Flutter 합성 — Skia/Impeller로 레이어·repaint boundary
- 나란히 — "transform 애니메이션"이 각 플랫폼에서 합성만으로 처리되는지

### Chapter 5: 네이티브 뷰 vs 자체 렌더 (5개 문서)
- 두 철학 — 플랫폼 위젯 합성 vs 자체 캔버스 직접 그리기
- 플랫폼 합성 — Browser·Compose·SwiftUI·RN(네이티브 뷰 제어)
- 자체 렌더 — Flutter(엔진이 모든 픽셀, 네이티브 뷰 안 씀)(flutter 연결)
- 트레이드오프 — 플랫폼 일관성·룩앤필 vs 완전한 제어·크로스 일관성
- RN vs Flutter — 같은 "크로스 플랫폼"의 정반대 접근(react-native 연결)

### Chapter 6: GPU 바닥 — 모두 여기로 수렴 (6개 문서)
- 공통 종착점 — 모든 UI가 결국 삼각형+텍스처+셰이더(gpu 연결)
- 2D도 GPU — 사각형·텍스트·이미지가 GPU 프리미티브가 되는 원리
- 렌더 엔진 — Skia(Browser/Flutter)·Core Animation(iOS)·각 엔진의 GPU 활용
- 텍스트 렌더링 — 글리프 아틀라스, 모든 플랫폼의 공통 과제(gpu 연결)
- 셰이더·합성 — GPU 합성이 모든 플랫폼의 마지막 단계
- 나란히 — 같은 화면이 각 플랫폼에서 GPU에 도달하는 경로

### Chapter 7: 성능과 종합 (5개 문서)
- jank의 공통 원인 — 프레임 예산 초과, 어느 단계(빌드/레이아웃/페인트)든
- 플랫폼별 jank 도구 — DevTools·Layout Inspector·Instruments·Flutter DevTools
- 같은 화면 비교 — 동일 UI를 4플랫폼으로 구현·프레임 비용 측정
- 트레이드오프 종합 — 단계 매핑·네이티브 통합·제어·일관성 표
- 종합 — 렌더링 지형도, 새 UI 프레임워크를 만나도 이 파이프라인으로 분석

→ **총 38개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). 비교가 핵심이라 `🔬`·`📊`에서 *항상 여러 플랫폼을 나란히*. `💻 실전 실험`은 같은 화면을 여러 플랫폼으로 측정.

## 🎨 스타일 가이드

1. **파이프라인 단계로 환원** — 모든 플랫폼을 같은 단계(빌드→레이아웃→페인트→합성→GPU)에 매핑
2. **항상 나란히** — 한 플랫폼만 말하지 말고 최소 2~3개를 같은 단계에서 비교
3. **선행 레포로 깊이 위임** — 각 플랫폼 *내부*는 해당 레포로, 여기선 *비교*
4. **GPU 수렴 강조** — 전부 결국 GPU 삼각형으로(gpu 레포가 공통 언어)
5. 공통 파이프라인·플랫폼 매핑은 다이어그램으로

## 🔬 검증 환경

> 멀티 플랫폼. 같은 화면을 각 플랫폼으로 구현해 프레임 비교.

```bash
# 같은 "스크롤 리스트 + 애니메이션" 화면을 4플랫폼으로 구현·측정

# Web (Browser)
#   Chrome DevTools Performance/Rendering: Layout/Paint/Composite 단계
#   Paint flashing·Layer borders로 합성 관찰

# Android (Compose)
#   Layout Inspector 리컴포지션, Macrobenchmark FrameTiming

# iOS (SwiftUI)
#   Instruments Core Animation FPS, _printChanges

# Flutter
#   Flutter DevTools Performance, Repaint Rainbow, UI/Raster 스레드

# 비교 측정:
#   - transform 애니메이션이 각 플랫폼에서 합성만으로 처리되나(레이아웃 건너뜀?)
#   - 같은 리스트 1항목 변경 시 각 플랫폼의 갱신 범위
#   - 프레임 시간(16.6ms 예산 대비)
#   - 텍스트 많은 화면의 렌더 비용
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `comparisons/`(플랫폼별 동일 화면)
2. **README.md**: 🧬 Synthesis 톤, "한 플랫폼 렌더 vs 공통 파이프라인의 다른 구현" 포지셔닝, 선행 5개 레포를 `🔗 레포 연결` 입력으로
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어, 항상 다중 플랫폼 비교

## 📚 참고 자료

- 각 선행 레포의 참고자료(이 레포는 그 종합)
- "Render pipeline" 관련 web.dev·Flutter·Android·Apple 문서
- *Real-Time Rendering* (GPU 공통 바닥)
- "How browsers work" / Flutter·Compose·SwiftUI 렌더 발표들

## 💡 핵심 분석 대상

```
공통 파이프라인 (모든 UI):
  선언/빌드 → 측정/레이아웃 → 페인트 → 합성 → GPU 래스터
     │            │            │        │         │
  React Fiber   단일/다중    그리기명령  레이어    삼각형+
  Compose Slot   패스        목록       합성       텍스처+
  SwiftUI AG                                       셰이더
  Flutter Element

두 갈래:
  플랫폼 합성: Browser·Compose·SwiftUI·RN
    → 플랫폼 위젯/레이어를 GPU 합성 (네이티브 룩앤필)
  자체 렌더: Flutter
    → 엔진이 모든 픽셀 직접(네이티브 뷰 0, 크로스 일관)

선언→트리 (4가지 좁히기):
  React  : VDOM diff + Fiber
  Compose: Slot Table 위치 기억
  SwiftUI: AttributeGraph 의존성
  Flutter: Element diff + RenderObject 재사용
  → 모두 "변경된 부분만" (reactivity-state-compared와 연결)

transform이 싼 이유 (공통):
  레이아웃·페인트 건너뛰고 합성만 → GPU가 처리
  → Browser·Compose·SwiftUI·Flutter 모두 동일 원리

GPU 수렴:
  사각형·텍스트·이미지 → 전부 삼각형 + 텍스처 + 셰이더
  → 모든 플랫폼의 마지막은 같은 GPU 파이프라인(gpu 레포)
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 38개 확인 + 멀티플랫폼 검증 환경 + 선행 5개 레포(browser-rendering·compose·swiftui/uikit·flutter·gpu)를 입력으로 하는 연결 명시. **항상 여러 플랫폼을 같은 파이프라인 단계에 매핑**하는 게 이 레포의 정체성.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
