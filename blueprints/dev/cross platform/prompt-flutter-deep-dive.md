# Flutter Deep Dive 레포지토리 제작 프롬프트

나는 "Flutter Deep Dive" 레포지토리를 만들려고 해.
Flutter가 네이티브 뷰를 *안 쓰고* 모든 픽셀을 직접 그리는 방식 — Dart VM, 3-Tree(Widget/Element/RenderObject), Skia→Impeller 렌더, build/layout/paint/composite 파이프라인을 완전히 파헤치는 레포야.
"위젯을 조합하는 것"과 "왜 모든 게 위젯이고 3개 트리가 어떻게 협력하며 Flutter가 어떻게 60fps로 직접 그리는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "위젯을 조합하는 것과, 3-Tree와 자체 렌더 엔진이 픽셀을 직접 그리는 방식을 아는 것은 다르다"

**핵심 차별화**:
1. 네이티브 뷰를 안 쓴다 — 자체 캔버스에 직접 그려 플랫폼 일관성·제어를 얻는 설계
2. 3-Tree 아키텍처 — Widget(설정)·Element(생명주기)·RenderObject(레이아웃/그리기)의 분리
3. Dart VM — JIT(개발 hot reload) + AOT(릴리스), Isolate 동시성
4. Skia → Impeller — 렌더 백엔드, 셰이더 컴파일 jank 문제와 Impeller의 해결

**타겟 독자**:
- Flutter를 쓰지만 3-Tree를 모르는 개발자
- "모든 게 위젯"의 의미를 모르는 개발자
- Flutter가 네이티브 뷰를 쓰는 줄 아는 개발자
- hot reload가 어떻게 되는지(JIT) 모르는 개발자
- `react-internals`·`gpu-graphics`·RN과 비교하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `gpu-graphics-deep-dive`(자체 렌더는 GPU 직접 활용), `compiler-deep-dive`(Dart JIT/AOT).
**🤝 시너지**: `react-native-deep-dive`(네이티브 뷰 제어 vs 자체 렌더 대조), `react-internals-deep-dive`(선언적 UI·재조정 대조), `jetpack-compose-internals`/`swiftui-internals`(선언적 UI 대조).
**🧬 수렴**: `rendering-pipelines-compared`(Flutter 자체 렌더 ↔ Browser ↔ Compose ↔ SwiftUI), `reactivity-state-compared`.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: Flutter 철학과 구조 (5개 문서)
- Flutter의 선택 — 네이티브 뷰 대신 자체 렌더, 그 이유(일관성·제어·성능)
- 레이어 구조 — Framework(Dart)·Engine(C++)·Embedder(플랫폼)
- "모든 게 위젯" — UI·레이아웃·스타일 전부 위젯, 합성으로 구성
- 선언적 UI — build(context)로 위젯 트리, UI = f(state)(react/compose 대조)
- 렌더 흐름 한눈에 — Widget→Element→RenderObject→픽셀

### Chapter 2: 3-Tree 아키텍처 (7개 문서)
- 왜 3개 트리인가 — 관심사 분리(설정·생명주기·렌더)
- Widget Tree — 불변 설정(configuration), 가볍게 자주 재생성
- Element Tree — 위젯의 생명주기·상태 보유, 트리 위치 추적
- RenderObject Tree — 실제 레이아웃·그리기·히트테스트
- 트리 간 매핑 — Widget→Element→RenderObject 대응 관계
- Element 재사용 — 위젯 재생성 시 Element diff·재사용(react Fiber 대조)
- Key의 역할 — Element 매칭, 리스트 재정렬(react key 대조)

### Chapter 3: 빌드와 재조정 (6개 문서)
- build 메서드 — 위젯 트리 생성, 언제 호출
- setState — 상태 변경→해당 Element 더티 표시→재빌드
- 재빌드 범위 — 어디서 시작해 어디까지, const 위젯의 스킵
- Element 업데이트 — 새 위젯과 기존 Element 비교(canUpdate)
- BuildContext — Element 자신, 트리 탐색(InheritedWidget 조회)
- 불필요 재빌드 — 흔한 함정, const·분리로 범위 축소

### Chapter 4: 레이아웃과 렌더 (6개 문서)
- 레이아웃 프로토콜 — "제약은 아래로, 크기는 위로, 부모가 위치" 단일 패스
- 제약(Constraints) — BoxConstraints, 부모→자식 제약 전달
- RenderObject 레이아웃 — performLayout, 측정 비용
- paint — Canvas에 그리기 명령, 페인트 순서
- Layer Tree와 합성 — 합성 레이어, repaint boundary
- 단일 패스의 이점 — O(n) 레이아웃, 웹/네이티브 다중 패스 대조

### Chapter 5: 렌더 엔진 — Skia / Impeller (5개 문서)
- 렌더 백엔드 — 그리기 명령→GPU, Skia의 역할(gpu 연결)
- 셰이더 컴파일 jank — Skia의 런타임 셰이더 컴파일이 첫 애니메이션을 끊기게
- Impeller — 셰이더 사전 컴파일로 jank 해결, 새 렌더 엔진
- 래스터·합성 스레드 — UI 스레드와 Raster 스레드 분업
- 텍스트·이미지 — 텍스트 렌더링, 이미지 디코딩(gpu 연결)

### Chapter 6: Dart VM과 동시성 (6개 문서)
- Dart VM — JIT(개발)·AOT(릴리스) 하이브리드(compiler 연결)
- hot reload — JIT 덕분에 상태 유지하며 코드 교체, 원리
- AOT 컴파일 — 릴리스 시 네이티브 코드, 시작·성능
- Isolate — 메모리 비공유 동시성(고루틴·Web Worker 대조), 메시지 패싱
- 이벤트 루프 — Dart의 단일 스레드 이벤트 루프, async/await(event-loop 대조)
- 메모리·GC — 세대별 GC, UI 스레드 보호

### Chapter 7: 성능과 실전 (5개 문서)
- 성능 측정 — DevTools 성능·타임라인, UI/Raster 스레드
- jank 진단 — build·layout·paint·셰이더 컴파일 어디가 병목
- 재빌드 최적화 — const·RepaintBoundary·선택적 재빌드
- 플랫폼 통합 — Platform Channel·네이티브 뷰 임베딩(PlatformView)
- 종합 — 끊기는 화면을 DevTools로 진단→수정, RN과 접근 비교

→ **총 38개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 3-Tree·레이아웃 프로토콜·렌더 엔진, `💻 실전 실험`은 Flutter DevTools·타임라인·repaint rainbow. `📊`는 재빌드·레이아웃·Skia vs Impeller 비교.

## 🎨 스타일 가이드

1. **3-Tree로 환원** — 모든 동작을 Widget/Element/RenderObject 협력으로
2. **RN과 항상 대조** — "자체 렌더"(Flutter) vs "네이티브 뷰 제어"(RN)
3. **선언적 UI 4자 대조** — Flutter Element ↔ React Fiber ↔ Compose Slot ↔ SwiftUI AttributeGraph
4. **gpu/compiler 레포로 착지** — 렌더는 gpu, JIT/AOT는 compiler
5. 3-Tree·레이아웃 프로토콜은 다이어그램으로

## 🔬 검증 환경

> Flutter SDK + DevTools + 시뮬레이터/기기.

```bash
flutter create flutter_lab
flutter run --profile      # 프로파일 모드(릴리스 성능 측정)

# 검증 방법
# 1) Flutter DevTools: Performance 탭, UI/Raster 스레드 타임라인
# 2) Repaint Rainbow: 다시 칠해지는 영역 시각화 → 과한 paint 확인
# 3) build 추적: debugProfileBuildsEnabled, 재빌드 위젯 확인
# 4) 3-Tree 관찰: Widget Inspector로 Widget/Element/RenderObject 트리
# 5) hot reload(JIT) vs hot restart 차이 체감
# 6) Skia vs Impeller: 첫 애니메이션 셰이더 jank 비교(가능 버전)
# 7) AOT: flutter build apk --release → 네이티브 컴파일 결과
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🔀 Cross-Platform 톤, "위젯을 조합한다 vs 3-Tree·자체 렌더가 어떻게 그리나" 포지셔닝, `🔗 레포 연결`(gpu·react-native·rendering-pipelines-compared)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- Flutter 아키텍처 개요 — https://docs.flutter.dev/resources/architectural-overview
- "Flutter's Rendering Pipeline" (구글 발표)
- "How Flutter renders Widgets" 발표·문서
- Impeller 문서 — https://docs.flutter.dev/perf/impeller
- Flutter Engine 소스 — https://github.com/flutter/engine

## 💡 핵심 분석 대상

```
3-Tree (관심사 분리):
  Widget Tree      : 불변 설정, 자주 재생성(가벼움)
  Element Tree     : 생명주기·상태, Widget↔RenderObject 다리
  RenderObject Tree: 레이아웃·그리기·히트테스트(무거움, 재사용)

  setState → Element 더티 → build() → 새 Widget
  → Element가 새 Widget과 비교(canUpdate) → RenderObject 재사용/갱신
  → "Widget은 버려도 RenderObject는 유지"(성능)

선언적 UI 4자 대조:
  React   : VDOM + Fiber 재조정
  Compose : Slot Table 위치 기억
  SwiftUI : AttributeGraph 의존성
  Flutter : Element 트리 diff + RenderObject 재사용
  → 모두 "선언 → 최소 갱신" (rendering-pipelines-compared)

레이아웃 프로토콜 (단일 패스):
  제약(constraints)은 아래로 → 크기(size)는 위로 → 부모가 위치 결정
  → O(n) 한 번에 (웹의 reflow·다중 패스와 대조)

Flutter vs RN (근본 차이):
  RN     : JS가 네이티브 뷰(UIView/android.View)를 제어
  Flutter: 네이티브 뷰 안 씀, 자체 엔진이 캔버스에 직접 그림
           → 플랫폼 일관성·완전한 제어, 단 네이티브 룩&필은 직접

셰이더 jank (Skia → Impeller):
  Skia : 첫 사용 셰이더를 런타임 컴파일 → 첫 애니메이션 끊김
  Impeller: 셰이더 사전 컴파일 → 첫 프레임부터 부드러움
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 38개 확인 + Flutter DevTools/Inspector 검증 환경 + gpu/react-native/rendering-pipelines-compared 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
