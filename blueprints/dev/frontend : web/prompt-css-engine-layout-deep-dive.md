# CSS Engine & Layout Deep Dive 레포지토리 제작 프롬프트

나는 "CSS Engine & Layout Deep Dive" 레포지토리를 만들려고 해.
캐스케이드·박스 모델·Flexbox/Grid 레이아웃 알고리즘·서식 컨텍스트가 브라우저 엔진에서 *실제로 어떻게 계산*되는지를 완전히 파헤치는 레포야.
"CSS 속성을 적용하는 것"과 "왜 이 요소가 이 크기·이 위치인지, 왜 가운데 정렬이 안 되는지 알고리즘 수준에서 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "CSS를 적용하는 것과, 레이아웃 엔진이 박스의 크기·위치를 어떤 알고리즘으로 계산하는지 아는 것은 다르다"

**핵심 차별화**:
1. 캐스케이드는 알고리즘이다 — 어떤 규칙이 이기는지(특이성·레이어·중요도)의 정확한 순서
2. 서식 컨텍스트(Formatting Context) — BFC/IFC/Flex/Grid가 레이아웃의 "세계"를 결정
3. Flexbox/Grid 내부 — 공간 분배·정렬이 어떤 단계로 계산되나(왜 늘어나고 줄어드나)
4. 포지셔닝·스택 — z-index가 안 먹는 이유, stacking context의 생성 조건

**타겟 독자**:
- CSS로 레이아웃하지만 "왜 이렇게 배치되는지" 종종 막히는 개발자
- margin collapse·BFC를 단어로만 아는 개발자
- Flex의 `flex-grow`·`flex-basis` 계산을 정확히 모르는 개발자
- z-index를 올려도 안 먹는 이유를 모르는 개발자
- `browser-rendering`의 Layout 단계를 알고리즘 수준으로 파려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `browser-rendering-deep-dive`(Layout이 CRP의 한 단계).
**🤝 시너지**: `web-performance-deep-dive`(레이아웃 비용·CLS), `css-engine`은 `react-internals`의 스타일링 토대.
**🧬 수렴**: `rendering-pipelines-compared`(브라우저 Flexbox/Grid ↔ Compose/SwiftUI 레이아웃 시스템).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: CSS 처리와 캐스케이드 (5개 문서)
- CSS 파싱→CSSOM — 규칙이 트리가 되는 과정(browser-rendering 연결)
- 캐스케이드 알고리즘 — origin·layer·specificity·order의 우선순위 결정 순서
- 특이성(Specificity) — 계산법, 흔한 오해, `!important`·인라인의 위치
- 상속과 계산값 — inherit/initial/unset, computed vs used value
- Cascade Layers — `@layer`로 우선순위를 명시적으로 관리

### Chapter 2: 박스 모델과 서식 컨텍스트 (5개 문서)
- 박스 모델 — content/padding/border/margin, box-sizing, 시각적 vs 레이아웃 박스
- 서식 컨텍스트 개념 — 박스들이 배치되는 "규칙 공간", 종류 개요
- 블록 서식 컨텍스트(BFC) — 생성 조건, float 포함·margin collapse 차단 용도
- Margin Collapsing — 인접/부모-자식/빈 박스 병합 규칙, 왜 헷갈리나
- 인라인 서식 컨텍스트(IFC) — line box, baseline, vertical-align의 실체

### Chapter 3: Flexbox 알고리즘 (5개 문서)
- Flex 컨테이너·아이템 — main/cross axis, flex 서식 컨텍스트
- 공간 분배 — flex-grow/shrink/basis의 정확한 계산 단계
- 정렬 — justify-content·align-items·align-self의 축별 동작
- 자동 마진·최소 크기 — `min-width:auto`가 줄어듦을 막는 함정
- 흔한 함정 — flex item 오버플로, 텍스트 줄바꿈, 중첩 flex

### Chapter 4: Grid 알고리즘 (6개 문서)
- Grid 컨테이너 — 명시적/암시적 트랙, grid 서식 컨텍스트
- 트랙 사이징 — fr·minmax·auto·content 기반 크기 계산
- 배치 — line·area·span, auto-placement 알고리즘
- 정렬 — grid에서의 justify/align, 셀 vs 콘텐츠
- 반응형 Grid — auto-fit/auto-fill·minmax로 미디어쿼리 없는 반응형
- Subgrid — 중첩 그리드 정렬 공유

### Chapter 5: 포지셔닝과 스택 (6개 문서)
- 포지셔닝 — static/relative/absolute/fixed/sticky, 포함 블록(containing block)
- absolute의 기준 — 가장 가까운 positioned 조상, 기준을 못 찾을 때
- sticky 동작 — 스크롤 컨테이너·임계값, sticky가 안 먹는 이유
- Stacking Context — 생성 조건(opacity·transform·z-index 등), 페인트 순서
- z-index 함정 — 스택 컨텍스트 안에 갇히는 이유, 형제 간만 비교
- 좌표·변환 — transform이 레이아웃이 아닌 페인트/합성 단계인 이유(browser-rendering 연결)

### Chapter 6: 모던 CSS와 반응형 (5개 문서)
- 단위 — px/em/rem/%/vw/vh, 논리 속성, 사용 시점
- Container Queries — 부모 크기 기반 반응형, 미디어쿼리와의 차이·구현
- 모던 기능 — :has()·clamp()·aspect-ratio·gap의 레이아웃 영향
- 반응형 전략 — 본질적 반응형(intrinsic), 콘텐츠 기반 레이아웃
- 격리 — contain·content-visibility로 레이아웃 범위 제한

### Chapter 7: 성능과 측정 (5개 문서)
- 레이아웃 비용 — 어떤 변경이 reflow를 트리거(browser-rendering 연결)
- 셀렉터 성능 — 매칭 비용, 스타일 무효화 범위
- CLS — 레이아웃 시프트의 원인, 예약 공간(aspect-ratio·skeleton)
- DevTools 활용 — Layout 패널·computed·박스 모델 시각화
- 종합 — 깨지는 레이아웃을 알고리즘으로 진단→수정

→ **총 37개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 레이아웃 알고리즘 단계, `💻 실전 실험`은 DevTools로 박스·서식 컨텍스트 검사. `📊`는 레이아웃 방식별 비용·CLS 비교.

## 🎨 스타일 가이드

1. **알고리즘 단계로 설명** — "이렇게 배치된다"를 계산 단계로 분해(특히 flex/grid 사이징)
2. **DevTools로 증명** — computed·박스 모델·grid 오버레이로 *보여준다*
3. **함정 중심** — z-index·margin collapse·min-width:auto 등 *알고리즘이 만든 함정*
4. **browser-rendering과 연결** — Layout 단계 비용은 그 레포로
5. flex/grid 사이징·stacking context는 다이어그램으로

## 🔬 검증 환경

> docker 불필요. **Chrome DevTools(Elements/Layout) + 정적 HTML**.

```bash
# 환경: Chrome. 빌드 없이 .html.

# 검증 방법
# 1) Elements > Computed: 실제 계산값(used value) 확인
# 2) Elements > Layout: Grid/Flex 오버레이로 트랙·라인 시각화
# 3) 박스 모델 다이어그램: content/padding/border/margin 실측
# 4) Rendering 패널: layout shift regions(CLS 시각화)
# 5) flex-grow 분배 실험: 컨테이너 폭 바꿔가며 아이템 크기 계산 검증
# 6) stacking context 실험: opacity/transform 추가 시 z-index 동작 변화 관찰
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `demos/`(레이아웃 실험 HTML)
2. **README.md**: 🌐 Frontend 톤, "CSS를 적용한다 vs 레이아웃을 어떻게 계산하나" 포지셔닝, `🔗 레포 연결`(browser-rendering·web-performance)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- CSS 명세 (CSSWG) — https://www.w3.org/Style/CSS/specs.en.html
- MDN CSS 레이아웃 — https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_layout
- "What Is The CSS ‘ch’ Unit" 등 Ahmad Shadeed 글
- web.dev Learn CSS — https://web.dev/learn/css
- Josh Comeau "CSS for JS Developers"

## 💡 핵심 분석 대상

```
캐스케이드 우선순위 (위가 이김):
  1. origin & importance (!important user > !important author > author > UA)
  2. cascade layer 순서
  3. specificity (인라인 > id > class/attr/pseudo-class > type)
  4. 소스 순서 (나중이 이김)

flex-grow 분배:
  컨테이너 600px, 아이템 basis 100·100·100 = 300, 여유 300
  grow 1·2·3 (합 6) → 300을 1:2:3 → +50·+100·+150
  최종 150·200·250
  단, min-width:auto가 콘텐츠보다 못 줄게 막음(함정)

stacking context (z-index가 갇힘):
  부모 A(z-index:1, 새 stacking context)
    └ 자식 (z-index:9999)
  부모 B(z-index:2)
  → 자식 9999여도 A 안에 갇혀 B(2)보다 아래!
  → z-index는 같은 stacking context 형제끼리만 비교

BFC (margin collapse 차단):
  부모-자식 margin이 병합되는 문제
  → 부모에 overflow:hidden 등으로 BFC 생성 → 병합 차단
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 37개 확인 + DevTools 검증 환경 + browser-rendering/web-performance/rendering-pipelines-compared 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
