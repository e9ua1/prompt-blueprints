# Web Performance Deep Dive 레포지토리 제작 프롬프트

나는 "Web Performance Deep Dive" 레포지토리를 만들려고 해.
Core Web Vitals(LCP·INP·CLS)가 *무엇을 측정하고*, 로딩·상호작용·시각 안정성 각각이 렌더링·네트워크·JS의 어느 메커니즘에서 결정되는지를 완전히 파헤치는 레포야.
"성능 점수를 올리는 것"과 "각 지표가 파이프라인의 무엇에서 나오는지 알고 근본 원인을 고치는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "Lighthouse 점수를 올리는 것과, 각 성능 지표가 어느 메커니즘에서 나오는지 알고 근본을 고치는 것은 다르다"

**핵심 차별화**:
1. 지표 → 메커니즘 — LCP·INP·CLS를 점수가 아닌 *원인*(CRP·long task·레이아웃)으로 분해
2. 로딩 워터폴 — 리소스 우선순위·CRP가 첫 화면 시점을 결정하는 방식
3. 상호작용 응답성 — INP가 메인 스레드 점유·이벤트 처리와 연결되는 원리
4. 측정의 진실 — Lab(Lighthouse) vs Field(RUM)의 차이, 무엇을 믿을까

**타겟 독자**:
- Lighthouse 점수만 보고 무엇을 고칠지 모르는 개발자
- LCP·INP·CLS를 단어로만 아는 개발자
- 이미지·폰트·JS 최적화를 "팁"으로만 적용하는 개발자
- Lab과 Field 점수가 다른 이유를 모르는 개발자
- `browser-rendering`·`event-loop`를 마치고 성능을 종합하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `browser-rendering-deep-dive`(렌더 파이프라인), `event-loop-async-deep-dive`(long task·INP), `network-deep-dive`(로딩).
**🤝 시너지**: `rendering-strategy-deep-dive`(전략별 지표), `frontend-build-tools-deep-dive`(번들 크기), `css-engine-layout-deep-dive`(CLS).
**🧬 수렴**: 프론트 전 레포의 성능 결과가 모이는 종합 지점.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 성능의 정의 (5개 문서)
- 왜 성능인가 — 비즈니스 영향, 사용자 인지 성능
- Core Web Vitals — LCP·INP·CLS의 정의와 임계값, 무엇을 대표하나
- 보조 지표 — TTFB·FCP·TTI·TBT, 각각이 파이프라인의 어느 단계
- 인지 성능 — 체감 속도, 스켈레톤·낙관적 UI
- 성능 모델 — RAIL, 사용자 중심 측정

### Chapter 2: 로딩 성능 — LCP (6개 문서)
- LCP의 정체 — 가장 큰 콘텐츠 요소가 그려지는 시점, 보통 무엇이 LCP인가
- 로딩 워터폴 — 리소스 발견·우선순위·다운로드 순서, CRP(browser-rendering 연결)
- 렌더 차단 — CSS·동기 JS가 첫 렌더를 막는 구조, critical CSS
- 리소스 힌트 — preload·preconnect·fetchpriority로 우선순위 제어
- 이미지 LCP — 우선 로딩, 반응형 이미지, 지연 로딩의 역효과
- TTFB와 서버 — 서버 응답·CDN·엣지가 LCP 하한을 결정

### Chapter 3: 상호작용 — INP (5개 문서)
- INP의 정체 — 입력→다음 페인트까지 지연, 전체 인터랙션 응답성
- long task — 메인 스레드를 막는 긴 작업(event-loop 연결), 50ms 규칙
- 입력 지연 분해 — input delay·processing·presentation delay
- 작업 분할 — yield·청킹·Web Worker로 메인 스레드 양보
- INP 개선 — 이벤트 핸들러 경량화, 비싼 리렌더 회피(react-internals 연결)

### Chapter 4: 시각 안정성 — CLS (5개 문서)
- CLS의 정체 — 예기치 않은 레이아웃 시프트, 점수 계산
- 시프트 원인 — 크기 미지정 이미지·광고·동적 삽입·웹폰트(css-engine 연결)
- 공간 예약 — width/height·aspect-ratio·skeleton으로 시프트 방지
- 폰트와 시프트 — FOIT/FOUT, font-display, 폰트 매칭
- 측정·디버깅 — Layout Shift regions, 시프트 출처 추적

### Chapter 5: 자산 최적화 (6개 문서)
- 이미지 — 포맷(AVIF/WebP)·반응형·압축·CDN 변환
- 폰트 — 서브셋·preload·font-display·가변 폰트
- JavaScript — 번들 크기·코드 스플리팅(build-tools 연결)·지연 로딩
- CSS — critical CSS·미사용 제거·로딩 순서
- 서드파티 — 태그·분석 스크립트의 비용, facade 패턴
- 우선순위 — 무엇을 먼저 최적화할지 ROI 판단

### Chapter 6: 캐싱과 네트워크 (5개 문서)
- HTTP 캐싱 — Cache-Control·ETag·immutable, 재방문 성능
- CDN — 엣지 캐싱, 지역 지연 단축
- 프로토콜 — HTTP/2·HTTP/3가 로딩에 주는 영향(network 연결)
- Service Worker 캐싱 — 오프라인·즉시 로딩, 캐시 전략
- 프리페칭 — 다음 페이지 예측 로딩, 과용의 비용

### Chapter 7: 측정과 모니터링 (6개 문서)
- Lab vs Field — Lighthouse(통제 환경) vs RUM(실사용자), 왜 다른가
- Lighthouse 해석 — 점수 구성·진단·기회, 점수에 속지 않기
- Performance API — 마크·측정·web-vitals 라이브러리로 직접 수집
- RUM 구축 — 실사용자 지표 수집·집계, p75
- WebPageTest — 필름스트립·워터폴 심층 분석
- 종합 — 느린 페이지를 지표→메커니즘→근본 수정으로 before/after

→ **총 38개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 지표가 나오는 파이프라인 메커니즘, `💻 실전 실험`은 Lighthouse·WebPageTest·DevTools·web-vitals. `📊`는 최적화 전후 CWV 정량 비교.

## 🎨 스타일 가이드

1. **지표를 메커니즘으로** — 모든 지표를 "어느 단계의 결과"로 환원(점수 사냥 금지)
2. **근본 원인 우선** — 증상(점수)이 아니라 원인(CRP·long task·시프트)을 고친다
3. **Lab/Field 구분** — 측정 환경에 따라 다름을 늘 명시
4. **다른 프론트 레포로 착지** — LCP→browser-rendering, INP→event-loop, CLS→css-engine
5. 워터폴·지표 분해는 다이어그램으로

## 🔬 검증 환경

> docker 불필요. **Lighthouse + WebPageTest + DevTools + web-vitals**.

```bash
npm i web-vitals
npm i -g lighthouse

# 검증 방법
# 1) Lighthouse: lighthouse https://site --view (Lab 측정)
# 2) DevTools Performance: LCP 마커·long task·layout shift 확인
# 3) web-vitals 라이브러리로 실제 지표 수집
#    onLCP(console.log); onINP(console.log); onCLS(console.log);
# 4) WebPageTest: 워터폴·필름스트립으로 로딩 순서 분석
# 5) before/after: 최적화 전후 같은 페이지 CWV 측정
# 6) 네트워크 스로틀링(Fast 3G)·CPU 4x로 저사양 재현
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `demos/`(나쁜→좋은 예제)
2. **README.md**: 🌐 Frontend 톤, "점수를 올린다 vs 근본을 고친다" 포지셔닝, `🔗 레포 연결`(browser-rendering·event-loop·css-engine)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- web.dev Core Web Vitals — https://web.dev/articles/vitals
- web-vitals 라이브러리 — https://github.com/GoogleChrome/web-vitals
- "Optimize INP" (web.dev)
- WebPageTest 문서 — https://docs.webpagetest.org/
- *High Performance Browser Networking* (Ilya Grigorik)

## 💡 핵심 분석 대상

```
지표 → 메커니즘:
  LCP ← 로딩 워터폴 + CRP + 렌더 차단 + 이미지/서버
  INP ← 메인 스레드 점유(long task) + 이벤트 처리 + 리렌더
  CLS ← 크기 미지정 + 동적 삽입 + 폰트 스왑
  → 점수가 아니라 "어느 메커니즘"을 고쳐야 하는지로 분해

LCP 워터폴:
  HTML 도착(TTFB) → CSS/JS 차단 해제 → LCP 이미지 발견·다운로드 → 그리기
  preload로 발견 앞당김, fetchpriority로 우선순위, critical CSS로 차단 단축

INP 분해:
  [input delay] 메인 스레드 바쁨 → [processing] 핸들러 실행 → [presentation] 다음 페인트
  long task가 input delay를 키움 → 작업 쪼개기로 양보

CLS:
  <img> 크기 없음 → 로드되며 아래 콘텐츠 밀림 → 시프트
  → width/height·aspect-ratio로 공간 예약 → 시프트 0

Lab vs Field:
  Lab(Lighthouse): 통제된 1회, INP 측정 어려움(상호작용 없음)
  Field(RUM):      실사용자 분포(p75), 진짜 경험
  → 둘이 다르면 Field를 믿되 Lab으로 원인 진단
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 38개 확인 + Lighthouse/web-vitals/DevTools 검증 환경 + browser-rendering/event-loop/css-engine 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
