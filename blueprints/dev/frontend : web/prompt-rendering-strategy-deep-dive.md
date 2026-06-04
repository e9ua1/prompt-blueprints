# Rendering Strategy Deep Dive 레포지토리 제작 프롬프트

나는 "Rendering Strategy Deep Dive" 레포지토리를 만들려고 해.
HTML이 *어디서·언제* 만들어지는가 — CSR/SSR/SSG/ISR/RSC와 하이드레이션 전략이 내부에서 어떻게 동작하고 무엇을 트레이드오프하는지를 완전히 파헤치는 레포야.
"Next.js 기능을 쓰는 것"과 "각 렌더링 전략이 어느 시점에 어디서 무슨 비용으로 HTML/JS를 만드는지 알고 선택하는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "프레임워크 렌더링 옵션을 켜는 것과, HTML이 어디서·언제 생성되고 어떻게 인터랙티브해지는지 아는 것은 다르다"

**핵심 차별화**:
1. "어디서·언제" 축 — 빌드타임 vs 요청타임 vs 클라이언트, 서버 vs 브라우저의 4사분면
2. 하이드레이션의 실체 — 정적 HTML이 인터랙티브해지는 비용, Selective/Progressive/Islands
3. RSC 패러다임 — 서버 전용 컴포넌트가 번들에서 빠지고 직렬화되는 방식
4. 선택의 근거 — 콘텐츠 성격(정적/동적/개인화)과 성능 지표에 따른 전략

**타겟 독자**:
- Next.js를 쓰지만 SSR/SSG/ISR/RSC 차이를 "느낌"으로 아는 개발자
- 하이드레이션이 뭔지, 왜 비용인지 모르는 개발자
- "서버 컴포넌트"를 단어로만 아는 개발자
- TTFB·LCP가 렌더링 전략과 어떻게 연결되는지 모르는 개발자
- `react-internals`·`browser-rendering`을 마치고 렌더 전략을 정리하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `react-internals-deep-dive`(하이드레이션·RSC·Suspense), `browser-rendering-deep-dive`(HTML→픽셀), `network-deep-dive`(TTFB·스트리밍).
**🤝 시너지**: `web-performance-deep-dive`(전략이 CWV에 주는 영향), `frontend-state-management-deep-dive`(서버 상태).
**🧬 수렴**: `web-performance-deep-dive`와 함께 프론트 성능의 양대 축.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 렌더링 전략 개요 (5개 문서)
- "HTML은 어디서 오는가" — 빌드/서버/클라이언트 생성의 본질적 차이
- 4사분면 — (서버 vs 클라이언트) × (빌드 vs 요청), 각 전략의 위치
- 성능 지표와 연결 — TTFB·FCP·LCP·TTI·INP가 전략별로 달라지는 지점
- SPA vs MPA — 페이지 전환 모델, 클라이언트 라우팅의 득실
- 전략 진화사 — 정적 HTML→SPA→SSR→SSG→ISR→RSC, 각 단계가 푼 문제

### Chapter 2: CSR (Client-Side Rendering) (5개 문서)
- CSR 흐름 — 빈 HTML→JS 다운로드→실행→렌더, 빈 화면 구간의 원인
- 장단점 — 서버 부담↓·전환 빠름 vs 초기 로딩·SEO 약점
- 라우팅·코드 스플리팅 — 라우트 단위 청크, 지연 로딩
- 데이터 페칭 — 마운트 후 요청, 워터폴, 로딩 상태 관리
- CSR가 맞는 곳 — 대시보드·인증 뒤 앱, SEO 불필요한 경우

### Chapter 3: SSR (Server-Side Rendering) (6개 문서)
- SSR 흐름 — 요청 시 서버가 HTML 생성→전송→하이드레이션
- 하이드레이션 — 서버 HTML에 이벤트·상태 연결, 왜 비용인가(JS 재실행)
- 스트리밍 SSR — 청크 단위 HTML 전송, TTFB 개선, Suspense 경계
- 서버 데이터 페칭 — 요청 컨텍스트, 캐시, 개인화
- SSR의 함정 — 하이드레이션 불일치(mismatch), 서버/클라 환경 차이, 메모리
- TTFB와 서버 비용 — 렌더 시간이 TTFB에 직접 반영, 캐싱 필요성

### Chapter 4: SSG / ISR (5개 문서)
- SSG — 빌드 타임 사전 생성, CDN 배포, 가장 빠른 TTFB
- 동적 경로 — 빌드 시 경로 생성, 대규모 페이지의 빌드 시간 문제
- ISR — 정적 + 백그라운드 재생성, stale-while-revalidate, 빌드 없이 갱신
- 캐시 무효화 — on-demand revalidation, 태그 기반 무효화
- 적합성 — 블로그·문서·커머스 카탈로그, 개인화 한계

### Chapter 5: React Server Components (6개 문서)
- RSC 개념 — 서버에서만 렌더, 클라이언트 번들에서 제외, "제로 번들" 컴포넌트
- 직렬화 — RSC payload(JSON 유사) 포맷, 클라이언트 컴포넌트 경계
- 서버/클라이언트 경계 — `'use client'`, props 직렬화 제약
- 데이터 페칭 — 서버에서 직접 DB/API, 워터폴 회피, async 컴포넌트
- 스트리밍과 결합 — RSC + Suspense + 스트리밍의 조합
- RSC 트레이드오프 — 멘탈 모델 복잡도, 캐싱, 기존 라이브러리 호환

### Chapter 6: 하이드레이션 전략 (6개 문서)
- 하이드레이션 비용 — 전체 하이드레이션의 문제(거대 앱), TTI 지연
- Selective Hydration — 필요한 부분 먼저, 우선순위 기반(React 18)
- Progressive Hydration — 점진적 활성화
- Islands 아키텍처 — 정적 페이지에 인터랙티브 섬만(Astro), partial hydration
- Resumability — Qwik의 접근, 하이드레이션 없이 재개
- 전략 비교 — 각 방식이 TTI·번들·복잡도에 주는 영향

### Chapter 7: 선택과 측정 (5개 문서)
- 전략 결정 트리 — 콘텐츠 성격(정적/동적/개인화)·SEO·인터랙션 기준
- 혼합 전략 — 한 앱에서 페이지별 다른 전략, 경계 설계
- 측정 — 같은 페이지를 전략별로 만들어 TTFB·LCP·TTI 비교
- 엣지 렌더링 — CDN 엣지에서 SSR, 지연 단축
- 종합 — 동일 앱을 CSR/SSR/SSG/RSC로 구현·측정 비교

→ **총 38개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 HTML/payload 생성 흐름·하이드레이션, `💻 실전 실험`은 Next.js로 전략별 구현 + view-source·Network 탭. `📊`는 전략별 TTFB·LCP·번들·TTI 비교.

## 🎨 스타일 가이드

1. **"어디서·언제"로 환원** — 모든 전략을 생성 위치·시점으로 분류
2. **view-source로 증명** — CSR(빈 HTML) vs SSR(채워진 HTML) 차이를 *직접 보여준다*
3. **지표와 연결** — 각 전략이 어떤 CWV를 개선/악화하는지
4. **react-internals와 연결** — 하이드레이션·RSC·Suspense의 메커니즘은 그 레포로
5. 4사분면·하이드레이션 흐름은 다이어그램으로

## 🔬 검증 환경

> docker 불필요. **Next.js + 브라우저 DevTools**가 검증 도구.

```bash
npx create-next-app@latest render-lab   # App Router
npm run build && npm start

# 검증 방법
# 1) view-source: CSR(빈 div) vs SSR/SSG(완성 HTML) 직접 비교
# 2) DevTools Network: 문서 TTFB, RSC payload(.txt) 확인
# 3) DevTools Performance: 하이드레이션 구간, TTI
# 4) 같은 페이지를 'use client'(CSR)·서버 컴포넌트(RSC)·generateStaticParams(SSG)로
#    → Network·Performance·Lighthouse로 차이 측정
# 5) ISR: revalidate 설정 후 stale→재생성 동작 관찰
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `apps/`(전략별 동일 앱)
2. **README.md**: 🌐 Frontend 톤, "옵션을 켠다 vs 어디서·언제 생성되나" 포지셔닝, `🔗 레포 연결`(react-internals·web-performance)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- Next.js 문서(Rendering) — https://nextjs.org/docs/app/building-your-application/rendering
- "Rendering on the Web" (web.dev) — https://web.dev/articles/rendering-on-the-web
- React 서버 컴포넌트 공식 — https://react.dev/reference/rsc/server-components
- Astro Islands — https://docs.astro.build/en/concepts/islands/
- Qwik resumability 문서

## 💡 핵심 분석 대상

```
4사분면 (어디서 × 언제):
                빌드 타임        요청 타임        클라이언트
  서버         │ SSG/ISR      │ SSR/RSC       │  ─
  클라이언트   │  ─           │  ─            │ CSR

view-source 차이:
  CSR : <div id="root"></div>   ← 비어있음, JS가 채움
  SSR : <div id="root"><h1>실제 콘텐츠...</h1></div>  ← 이미 채워짐
  → SEO·LCP 차이의 근원

하이드레이션:
  서버 HTML(보임, 클릭 안 됨) → JS 다운로드 → 하이드레이션(이벤트 연결)
  → 이 구간 = "보이지만 안 눌림", TTI 지연
  Selective: 사용자가 만진 곳부터 우선 하이드레이션

RSC payload:
  서버 컴포넌트 → JS 번들에 0바이트, 대신 직렬화된 트리 전송
  'use client' 컴포넌트만 번들에 포함
  → 번들 크기 ↓, 서버에서 직접 데이터 접근
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 38개 확인 + Next.js/DevTools 검증 환경 + react-internals/web-performance 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
