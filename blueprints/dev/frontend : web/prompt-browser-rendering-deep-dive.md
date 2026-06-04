# Browser Rendering Deep Dive 레포지토리 제작 프롬프트

나는 "Browser Rendering Deep Dive" 레포지토리를 만들려고 해.
HTML/CSS가 화면 픽셀이 되기까지 — 파싱·스타일·레이아웃·페인트·합성의 Critical Rendering Path를 완전히 파헤치는 레포야.
"CSS를 적용하는 것"과 "왜 이 변경이 reflow를 일으키고 스크롤이 끊기는지 알고 설계하는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "CSS로 스타일을 주는 것과, 브라우저가 그것을 픽셀로 바꾸는 파이프라인을 아는 것은 다르다"

**핵심 차별화**:
1. Critical Rendering Path — DOM/CSSOM→Render Tree→Layout→Paint→Composite, 각 단계의 비용
2. Reflow vs Repaint vs Composite — 어떤 속성 변경이 어느 단계를 트리거하는가(transform이 싼 이유)
3. 합성 레이어 — GPU 합성, 레이어 승격, will-change의 양날
4. 메인 스레드의 병목 — JS·레이아웃·페인트가 한 스레드에서 경쟁, 60fps 예산

**타겟 독자**:
- CSS는 잘 쓰지만 왜 어떤 애니메이션은 끊기고 어떤 건 부드러운지 모르는 개발자
- "GPU 가속"을 들어봤지만 무엇이 GPU로 가는지 모르는 개발자
- DevTools Performance 탭을 열어봤지만 보라색(Layout)·초록색(Paint)이 뭔지 모르는 개발자
- React를 쓰지만 그 아래 브라우저 렌더링이 블랙박스인 개발자

## 🔗 레포 연결

**⬆️ 선행**: `gpu-graphics-deep-dive`(Composite·페인트가 GPU로 가는 바닥), `network-deep-dive`(리소스 로딩·CRP의 시작).
**🤝 시너지**: `v8-engine-deep-dive`(JS 실행이 메인 스레드 점유), `css-engine-layout-deep-dive`(레이아웃 알고리즘 상세), `react-internals-deep-dive`(VDOM이 결국 DOM 변경).
**🧬 수렴**: `rendering-pipelines-compared`(브라우저 합성 ↔ Compose/SwiftUI/Flutter), `web-performance-deep-dive`.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 브라우저 아키텍처 (5개 문서)
- 멀티프로세스 브라우저 — 브라우저/렌더러/GPU/네트워크 프로세스 분리, 사이트 격리
- 렌더러 프로세스 내부 — 메인 스레드·컴포지터 스레드·래스터 스레드의 분업
- Blink/WebKit 개요 — 렌더링 엔진의 역할, V8과의 경계
- 픽셀 파이프라인 한눈에 — Loading→Scripting→Rendering→Painting의 전체 흐름
- 프레임 생명주기 — vsync, 한 프레임에 일어나는 일, requestAnimationFrame 타이밍

### Chapter 2: 파싱 — DOM과 CSSOM (5개 문서)
- HTML 파싱 — 토크나이즈→트리 구성, 스트리밍 파싱, 프리로드 스캐너
- DOM 트리 구성 — 노드 표현, 파서 차단(script)이 DOM 구성을 막는 이유
- CSSOM 구성 — CSS 파싱, CSS가 렌더링 차단 리소스인 이유
- script와 파싱 — async/defer/module이 파싱·실행 타이밍에 주는 영향
- Render Tree — DOM + CSSOM 결합, display:none vs visibility:hidden의 트리 차이

### Chapter 3: 스타일과 레이아웃 (6개 문서)
- 스타일 계산 — 셀렉터 매칭, Cascade·상속, 계산된 스타일(computed style)
- 셀렉터 성능 — 우→좌 매칭, 비싼 셀렉터, 스타일 무효화(invalidation) 범위
- 레이아웃(Reflow) — 박스의 위치·크기 계산, 레이아웃이 트리 전체에 번지는 경우
- 강제 동기 레이아웃(Layout Thrashing) — 읽기/쓰기 교차가 레이아웃을 반복 강제, 측정·해결
- 레이아웃 격리 — `contain`·content-visibility로 레이아웃 범위 제한
- 레이아웃 트리거 속성 — 어떤 속성 읽기/쓰기가 즉시 레이아웃을 강제하나(offsetTop 등)

### Chapter 4: 페인트와 합성 (6개 문서)
- 페인트 — Render Tree → 페인트 명령(draw call) 목록, 페인트 순서(stacking)
- 레이어 — 합성 레이어란, 레이어 승격 조건, 레이어 트리
- 래스터화 — 페인트 명령을 비트맵으로, 타일링, GPU 래스터
- 합성(Composite) — 레이어를 GPU에서 합치기, 메인 스레드 없이 동작 가능한 작업
- transform/opacity가 싼 이유 — 합성만으로 처리(레이아웃·페인트 건너뜀)
- will-change와 레이어 폭발 — 레이어의 메모리 비용, 과용의 역효과

### Chapter 5: 렌더링 성능 — 60fps (6개 문서)
- 프레임 예산 — 16.6ms 안에 JS+스타일+레이아웃+페인트+합성, 초과 시 jank
- 애니메이션 비용 등급 — composite-only(최선) vs paint vs layout(최악), 어떤 속성을 쓸까
- 스크롤 성능 — 합성 스크롤, passive listener, 스크롤 중 레이아웃 회피
- requestAnimationFrame vs setTimeout — 프레임 정렬, rAF로 애니메이션 동기화
- 강제 레이아웃 일괄 처리 — 읽기 모아서·쓰기 모아서(FastDOM 패턴)
- 렌더 차단 최소화 — critical CSS, 리소스 우선순위, 폰트 로딩(FOIT/FOUT)

### Chapter 6: JavaScript와 렌더링 (5개 문서)
- 메인 스레드 경쟁 — JS 실행이 렌더링을 막는 구조, long task
- 이벤트 루프와 렌더링 — 마이크로태스크·태스크·렌더 단계의 순서(event-loop 레포와 연결)
- 작업 분할 — long task를 쪼개기, scheduler.yield, requestIdleCallback
- Web Worker로 오프로딩 — 메인 스레드에서 무거운 연산 분리
- DOM 변경 비용 — 대량 DOM 조작, DocumentFragment, 가상 DOM이 푸는 문제

### Chapter 7: 측정과 최적화 (5개 문서)
- DevTools Performance — 프레임별 분석, 메인 스레드 플레임 차트 읽기(노랑·보라·초록)
- Rendering 패널 — paint flashing, layer borders, FPS 미터로 시각 진단
- Core Web Vitals 연결 — LCP·CLS·INP가 렌더링 파이프라인의 무엇과 연결되나
- about:tracing / Perfetto — 더 깊은 프레임 단계 추적
- 케이스 스터디 — 끊기는 애니메이션 하나를 진단→레이어/속성 교체→before/after 측정

→ **총 38개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 파이프라인 단계·Blink 동작, `💻 실전 실험`은 DevTools Performance/Rendering 패널로 *눈으로* 재현. `📊`는 reflow vs composite의 프레임타임 비교.

## 🎨 스타일 가이드

1. **파이프라인 단계로 귀결** — 모든 현상을 "어느 단계(Layout/Paint/Composite)의 일"로 분류
2. **DevTools로 증명** — paint flashing·layer border를 켜서 *보여준다*
3. **나쁜 예→측정→개선** — 끊기는 코드를 먼저 보이고 프레임타임으로 개선 증명
4. **GPU·React와 연결** — 합성은 gpu 레포로, DOM 변경은 react 레포로 이어준다
5. CRP·레이어 트리는 항상 다이어그램으로

## 🔬 검증 환경

> docker 불필요. **Chrome DevTools(Performance·Rendering·Layers) + about:tracing**가 검증 도구. 정적 HTML로 재현.

```bash
# 환경: Chrome. 빌드 없이 .html 직접 열기.

# 1) Performance 탭 — 프레임 녹화
#    노랑=Scripting, 보라=Rendering(Layout), 초록=Painting
#    long task·강제 동기 레이아웃 경고 확인

# 2) Rendering 패널 (DevTools > More tools > Rendering)
#    - Paint flashing: 다시 칠해지는 영역 초록 점멸
#    - Layer borders: 합성 레이어 경계
#    - Frame rendering stats: 실시간 FPS/GPU 메모리

# 3) Layers 패널: 합성 레이어 트리·승격 이유 확인

# 4) 강제 동기 레이아웃 재현
#    for(el of list){ el.style.width = el.offsetWidth+1+'px' }  // 읽기-쓰기 교차
#    → Performance에 보라색 'Layout' 반복 출현

# 5) transform vs top 애니메이션 프레임타임 비교
#    같은 이동을 transform(합성)과 top(레이아웃)으로 → 끊김 차이 측정
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `demos/`(재현 HTML)
2. **README.md**: 🌐 Frontend 톤, "CSS를 준다 vs 픽셀이 되는 파이프라인을 안다" 포지셔닝, `🔗 레포 연결`(gpu·v8·react)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- web.dev Rendering Performance — https://web.dev/articles/rendering-performance
- "Inside look at modern web browser" (Chrome 팀) — https://developer.chrome.com/blog/inside-browser-part1
- *High Performance Browser Networking* (Ilya Grigorik)
- Chromium 렌더링 문서 — https://www.chromium.org/developers/design-documents/
- CSS Triggers — https://csstriggers.com/

## 💡 핵심 분석 대상

```
Critical Rendering Path:

  HTML ─► [파싱] ─► DOM ─┐
  CSS  ─► [파싱] ─► CSSOM ┴─► Render Tree
                              │
                              ▼ Layout (위치·크기)  ← reflow
                              │
                              ▼ Paint (그리기 명령)  ← repaint
                              │
                              ▼ Composite (레이어 합성, GPU)  ← 가장 쌈
                              │
                              ▼ 화면

속성 변경 비용:
  width/top/margin 변경  → Layout부터 다시 (가장 비쌈)
  color/background 변경  → Paint부터 다시
  transform/opacity 변경 → Composite만 (Layout·Paint 건너뜀, GPU)
  → 애니메이션은 transform/opacity로!

Layout Thrashing:
  div.style.left = ...;     // 쓰기 (레이아웃 무효화)
  div.offsetTop;            // 읽기 → 강제 동기 레이아웃!
  div.style.top = ...;      // 쓰기
  div.offsetTop;            // 또 강제 레이아웃...
  → 읽기 모으고 → 쓰기 모으기로 해결

프레임 예산 (16.6ms @60fps):
  [JS 실행] [Style] [Layout] [Paint] [Composite]
  이 합이 16.6ms 초과 → 프레임 드롭(jank)
  long task(50ms+) 하나가 여러 프레임을 먹음
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 38개 확인 + DevTools 검증 환경 + gpu/v8/react/rendering-pipelines-compared 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
