# Build Tools Deep Dive 레포지토리 제작 프롬프트

나는 "Build Tools Deep Dive" 레포지토리를 만들려고 해.
번들러가 모듈을 어떻게 해석하고, 변환(트랜스파일)하고, 트리 셰이킹으로 죽은 코드를 제거하고, HMR로 빠른 개발 서버를 만드는지를 완전히 파헤치는 레포야.
"Vite 설정을 복붙하는 것"과 "번들러가 의존성 그래프를 어떻게 만들고 왜 어떤 코드가 번들에서 빠지는지 알고 설정하는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "번들러를 설정하는 것과, 모듈 해석·변환·번들링·트리셰이킹이 내부에서 어떻게 동작하는지 아는 것은 다르다"

**핵심 차별화**:
1. 의존성 그래프 — 엔트리에서 모듈을 추적해 그래프를 만드는 원리
2. 변환 파이프라인 — Babel/SWC/esbuild가 코드를 AST로 변환하는 방식과 속도 차이
3. 트리 셰이킹 — ESM 정적 구조가 죽은 코드 제거를 가능케 하는 이유, side effect 함정
4. 개발 vs 프로덕션 — Vite의 dev(no-bundle ESM)와 build(Rollup)가 다른 이유

**타겟 독자**:
- webpack/vite 설정을 복붙하지만 동작을 모르는 개발자
- 번들이 큰 이유·트리셰이킹이 안 되는 이유를 모르는 개발자
- "Vite가 빠르다"고 들었지만 왜인지 모르는 개발자
- HMR이 어떻게 즉시 갱신되는지 모르는 개발자
- `javascript-deep-dive`의 모듈을 마치고 번들링을 파려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `javascript-deep-dive`(ESM/CJS 모듈 의미론), `compiler-deep-dive`(AST·변환).
**🤝 시너지**: `web-performance-deep-dive`(번들 크기·코드 스플리팅), `v8-engine-deep-dive`(번들이 실행될 코드).
**🧬 수렴**: 프론트 빌드 파이프라인의 토대.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 왜 번들러가 필요한가 (5개 문서)
- 모듈의 역사 — 전역 스크립트→IIFE→CommonJS→AMD→ESM, 각자가 푼 문제
- 번들러가 푸는 문제 — 많은 모듈·브라우저 요청 비용·구문 호환, 한 파일로
- 브라우저 ESM의 등장 — `<script type=module>`, 번들 없는 개발이 가능해진 배경
- 번들러 vs 트랜스파일러 vs 미니파이어 — 역할 구분(webpack vs babel vs terser)
- 도구 지형 — Webpack/Rollup/esbuild/Vite/Turbopack/Parcel의 위치

### Chapter 2: 모듈 해석과 의존성 그래프 (5개 문서)
- 엔트리와 그래프 — 엔트리에서 import를 따라 모듈 그래프 구성
- 모듈 해석(resolution) — node_modules 탐색, alias, extensions, exports 필드
- ESM vs CJS 해석 — 정적 import vs 동적 require, 상호운용 처리
- 그래프 표현 — 모듈 ID·의존성·청크 매핑
- 순환 의존 — 번들러의 처리, 런타임 문제

### Chapter 3: 변환(Transform) (5개 문서)
- 왜 변환하나 — 최신 문법·TS·JSX를 구형 호환으로, 폴리필
- AST 기반 변환 — parse→transform(visitor)→generate(compiler 레포 연결)
- Babel — 플러그인·프리셋, 느린 이유(JS 구현)
- SWC/esbuild — Rust/Go 기반 네이티브 속도, Babel 대비 수십 배
- 변환 비용 — 변환이 빌드 시간을 지배하는 지점, 캐싱

### Chapter 4: 번들링과 코드 스플리팅 (6개 문서)
- 번들 생성 — 모듈을 함수로 감싸 런타임에 등록·실행하는 구조
- 모듈 런타임 — `__webpack_require__` 류의 모듈 시스템 구현
- 코드 스플리팅 — 동적 import로 청크 분할, 라우트·컴포넌트 단위
- 청크 전략 — vendor·공통·런타임 청크, splitChunks
- 청크 로딩 — 지연 로딩, preload/prefetch, 청크 간 의존
- 출력 최적화 — 해시 파일명·long-term caching, 매니페스트

### Chapter 5: 트리 셰이킹과 최적화 (5개 문서)
- 트리 셰이킹 원리 — ESM 정적 구조로 미사용 export 제거
- side effect — `sideEffects` 필드, 부수효과 있는 모듈이 제거 안 되는 이유
- 안 되는 경우 — CJS·재export·동적 접근이 트리셰이킹을 막는 패턴
- 미니피케이션 — terser/esbuild minify, 죽은 코드·이름 단축, 스코프 호이스팅
- 번들 분석 — bundle analyzer로 무엇이 번들에 들었는지 추적·줄이기

### Chapter 6: 개발 서버와 HMR (5개 문서)
- dev 서버 차이 — Webpack(번들 후 서빙) vs Vite(no-bundle, 브라우저 ESM)
- Vite가 빠른 이유 — 사전 번들(esbuild)된 의존성 + 소스는 온디맨드 ESM
- HMR 원리 — 변경 모듈만 교체, 모듈 경계·수용(accept), 상태 보존
- HMR 경계 — 어디까지 핫 교체되고 어디서 전체 새로고침되나
- 소스맵 — 변환·번들 후 원본 위치 추적, 디버깅

### Chapter 7: 도구 비교와 측정 (5개 문서)
- 아키텍처 비교 — Webpack/Rollup/esbuild/Vite/Turbopack의 설계 철학
- 속도 차이의 근원 — JS vs 네이티브, 번들 vs no-bundle
- 빌드 측정 — 빌드 시간·번들 크기·청크 구성 정량 비교
- 마이그레이션 — Webpack→Vite 전환 시 고려, 호환성
- 종합 — 동일 앱을 여러 번들러로 빌드·측정 비교

→ **총 36개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 그래프 구성·변환·번들 런타임, `💻 실전 실험`은 각 번들러 CLI + bundle analyzer + 빌드 산출물 까보기. `📊`는 도구별 빌드시간·번들크기 비교.

## 🎨 스타일 가이드

1. **산출물을 까본다** — 번들된 코드를 직접 열어 모듈 런타임·트리셰이킹 결과 확인
2. **그래프로 사고** — 모든 동작을 의존성 그래프 위에서 설명
3. **측정으로 비교** — "Vite가 빠르다"를 빌드 시간 숫자로
4. **compiler/javascript와 연결** — 변환은 compiler, 모듈은 javascript 레포로
5. 의존성 그래프·청크 분할은 다이어그램으로

## 🔬 검증 환경

> docker 불필요. **node + 각 번들러 CLI + analyzer**.

```bash
# 도구 설치
npm i -D webpack webpack-cli vite rollup esbuild
npm i -D webpack-bundle-analyzer rollup-plugin-visualizer

# 검증 방법
# 1) 같은 소스를 4도구로 빌드 → dist 산출물 비교(코드·크기)
npx esbuild app.js --bundle --outfile=out.js   # 결과 열어 모듈 래핑 확인
# 2) 번들 분석
npx vite build && npx vite-bundle-visualizer
# 3) 트리셰이킹 검증: 미사용 export가 dist에서 빠졌는지 grep
# 4) 빌드 시간 측정: time npx webpack vs time npx vite build vs time esbuild
# 5) HMR: Vite dev에서 파일 수정 → Network에서 변경 모듈만 재요청 확인
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `comparisons/`(번들러별 동일 앱)
2. **README.md**: 🌐 Frontend 톤, "설정을 복붙한다 vs 어떻게 번들되나" 포지셔닝, `🔗 레포 연결`(javascript·compiler·web-performance)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- "Tooling.Report" — https://bundlers.tooling.report/
- Vite 문서(Why Vite) — https://vitejs.dev/guide/why.html
- Webpack 개념 문서 — https://webpack.js.org/concepts/
- Rollup·esbuild 공식 문서
- "Hot Module Replacement" 명세·Evan You 발표

## 💡 핵심 분석 대상

```
의존성 그래프:
  entry.js
    ├─ import a from './a'   → a.js
    │     └─ import './c'    → c.js
    └─ import b from './b'   → b.js
  → 그래프 전체를 순회해 번들 구성

트리 셰이킹 (ESM 정적 구조 덕분):
  // utils.js : export { used, unused }
  import { used } from './utils'   // unused는 어디서도 안 씀
  → 정적 분석으로 unused 제거 가능
  단, CJS(require)·동적 접근·side effect면 제거 불가

Vite가 빠른 이유:
  Webpack dev: 전체 번들 → 서빙 (앱 크면 느림)
  Vite dev   : 의존성만 esbuild 사전번들, 소스는 브라우저가 ESM으로 직접 요청
               → 변경분만 처리, 콜드 스타트 빠름
  Vite build : Rollup으로 최적 번들(프로덕션은 번들이 유리)

HMR:
  파일 수정 → 변경 모듈만 새 버전 전송 → accept 경계까지 교체
  → 전체 새로고침 없이 상태 유지하며 갱신
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 36개 확인 + 번들러 CLI/analyzer 검증 환경 + javascript/compiler/web-performance 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
