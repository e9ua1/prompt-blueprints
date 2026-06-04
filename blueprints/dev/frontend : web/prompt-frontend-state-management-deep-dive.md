# State Management Deep Dive 레포지토리 제작 프롬프트

나는 "State Management Deep Dive" 레포지토리를 만들려고 해.
Flux/Redux 단방향, Signal 세밀 반응성, Proxy 기반, 원자(atom) 기반, 서버 상태 — 프론트엔드 상태 관리의 *서로 다른 패러다임*이 내부에서 어떻게 동작하고 무엇을 트레이드오프하는지를 완전히 파헤치는 레포야.
"상태 라이브러리를 쓰는 것"과 "각 패러다임이 리렌더를 어떻게 추적하고 왜 이걸 골라야 하는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "상태 라이브러리를 고르는 것과, 각 패러다임이 변경을 어떻게 추적·전파하는지 알고 선택하는 것은 다르다"

**핵심 차별화**:
1. 추적 메커니즘의 분류 — 명시적 디스패치 vs Proxy 자동 추적 vs Signal 구독, 셋의 근본 차이
2. 세밀 반응성 — VDOM 리렌더 없이 변경된 부분만 갱신하는 Signal의 원리
3. 서버 상태 ≠ 클라이언트 상태 — 캐싱·무효화·동기화는 다른 문제다
4. 선택의 근거 — 앱 규모·팀·렌더링 모델에 따른 트레이드오프

**타겟 독자**:
- Redux/Zustand/Recoil 중 뭘 쓸지 "느낌"으로 고르는 개발자
- "Signal이 빠르다"고 들었지만 왜인지 모르는 개발자
- 전역 상태에 다 넣고 리렌더 지옥을 겪는 개발자
- 서버 데이터를 전역 상태에 수동 캐싱하는 개발자
- `react-internals`를 마치고 상태 추적을 깊이 보려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `react-internals-deep-dive`(리렌더가 일어나는 메커니즘), `javascript-deep-dive`(클로저·Proxy).
**🤝 시너지**: `web-apis-wasm-deep-dive`(Proxy 트랩), `rendering-strategy-deep-dive`(서버 상태와 SSR).
**🧬 수렴**: `reactivity-state-compared`(Signal ↔ Compose Snapshot ↔ SwiftUI AttributeGraph ↔ MVCC — *이 레포가 그 비교의 웹 측 입력*).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 상태 관리의 문제 정의 (5개 문서)
- 무엇이 "상태"인가 — UI 상태·클라이언트 상태·서버 상태·URL 상태의 구분
- 핵심 난제 — 변경 추적·리렌더 범위·일관성, 모든 라이브러리가 푸는 같은 문제
- 추적 패러다임 3분류 — 명시적(Redux)·자동 추적(Proxy)·구독(Signal)
- 리렌더와 상태 — 상태 변경이 어떻게 리렌더로 이어지나(react-internals 연결)
- prop drilling과 Context — 왜 전역이 필요해지나, Context의 리렌더 한계

### Chapter 2: Flux / Redux — 단방향 (5개 문서)
- Flux 아키텍처 — 단방향 데이터 흐름, action→dispatcher→store→view
- Redux 코어 — 단일 store·순수 reducer·불변 업데이트, 예측 가능성
- 미들웨어 — dispatch 파이프라인, thunk·saga의 비동기 처리
- 셀렉터와 메모이제이션 — reselect, 리렌더 최소화, 구독 모델
- 트레이드오프 — 보일러플레이트 vs 예측가능성, RTK가 줄인 것, 언제 과한가

### Chapter 3: Signal — 세밀 반응성 (5개 문서)
- Signal 모델 — 값 + 구독자, 읽기 시 의존성 자동 등록
- 자동 의존성 추적 — computed가 읽은 signal을 구독, 변경 시 정확히 그 부분만
- 세밀 갱신 — VDOM diff 없이 DOM 노드 직접 갱신, Solid의 접근
- 글리치 없는 전파 — 일관된 업데이트 순서, 다이아몬드 의존성
- Signal vs VDOM — 왜 Signal이 리렌더를 줄이나, React가 Signal을 안 쓴 이유

### Chapter 4: Proxy 기반 — 자동 추적 (5개 문서)
- Proxy 트랩 — get/set 가로채기로 접근·변경 감지(javascript 레포 연결)
- MobX — 관찰 가능 상태·반응(reaction)·자동 추적, 파생 값
- Valtio — 가변 스타일 + Proxy, 스냅샷으로 React 연동
- 가변 vs 불변 — 가변 편의 + 내부 불변 스냅샷, Immer의 구조 공유
- 트레이드오프 — 마법 같은 편의 vs 디버깅·예측가능성

### Chapter 5: 원자(Atom) 기반 (5개 문서)
- 원자 모델 — 작은 상태 단위, 상향식 구성(Recoil/Jotai)
- 파생 원자 — 원자 조합으로 계산 상태, 의존 그래프
- 구독 정밀도 — 원자 단위 구독으로 리렌더 범위 축소
- 비동기 원자 — Suspense 연동, 로딩 상태 선언적 처리
- 트레이드오프 — 유연성 vs 상태 분산, 디버깅

### Chapter 6: 서버 상태 (5개 문서)
- 서버 상태는 다르다 — 원격 소유·비동기·진부화(stale), 클라이언트 상태와의 차이
- 캐싱과 무효화 — 쿼리 키·캐시·stale-while-revalidate(React Query/SWR)
- 동기화 — 백그라운드 갱신·중복 제거·재시도, 낙관적 업데이트
- 캐시 일관성 — 무효화 전략, mutation 후 갱신
- "전역 상태에 서버 데이터" 안티패턴 — 왜 분리해야 하나

### Chapter 7: 비교와 선택 (5개 문서)
- 패러다임 종합 비교 — 추적 방식·리렌더 정밀도·보일러플레이트·러닝커브 표
- 같은 앱 다섯 구현 — 동일 기능을 Redux·Zustand·Jotai·Valtio·Signal로
- 리렌더 정밀도 측정 — 각 구현의 리렌더 횟수를 Profiler로 정량 비교
- 선택 가이드 — 규모·팀·렌더링 모델별 권장
- 종합 — 미니 반응성 시스템(Proxy/Signal) 직접 구현으로 원리 통합

→ **총 35개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 각 라이브러리의 추적·구독 메커니즘, `💻 실전 실험`은 동일 기능을 여러 라이브러리로 구현·DevTools 리렌더 측정. `📊`는 리렌더 횟수·번들 크기 비교.

## 🎨 스타일 가이드

1. **추적 메커니즘으로 분류** — 모든 라이브러리를 "어떻게 변경을 아는가"로 환원
2. **같은 기능 여러 구현** — 동일 예제로 패러다임 차이를 *체감*
3. **리렌더로 증명** — "정밀하다"를 Profiler 리렌더 횟수로
4. **직접 만들며** — 미니 Signal/Proxy 반응성으로 핵심 체득
5. **reactivity-state-compared로 연결** — 웹 패러다임이 모바일/DB와 만나는 지점 예고

## 🔬 검증 환경

> docker 불필요. **React DevTools Profiler + 멀티 라이브러리 벤치 하니스**가 검증 도구.

```bash
npx create-vite state-lab --template react-ts
npm i @reduxjs/toolkit react-redux zustand jotai valtio @tanstack/react-query
# Solid(Signal) 비교용 별도 미니 예제

# 검증 방법
# 1) 동일한 "Todo + 카운터 + 필터" 앱을 라이브러리별로 구현
# 2) React DevTools Profiler로 "한 항목 변경 시 리렌더되는 컴포넌트 수" 비교
# 3) 의도적으로 큰 리스트 → 각 패러다임의 리렌더 범위 측정
# 4) 미니 구현:
#    - signal(): {get,set,subscribe} + computed의 자동 의존성 추적
#    - proxy 기반 observable: get으로 의존성 수집, set으로 통지
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `comparisons/`(라이브러리별 동일 앱)
2. **README.md**: 🌐 Frontend 톤, "라이브러리를 고른다 vs 추적 메커니즘을 안다" 포지셔닝, `🔗 레포 연결`(react-internals·reactivity-state-compared)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- Redux·RTK 공식 — https://redux.js.org/
- "SolidJS Reactivity" 문서 — https://www.solidjs.com/guides/reactivity
- MobX·Jotai·Valtio·Zustand 공식 문서
- TanStack Query 문서 — https://tanstack.com/query
- "A Hands-on Introduction to Fine-Grained Reactivity" (Ryan Carniato)

## 💡 핵심 분석 대상

```
추적 메커니즘 3분류:
  Redux   : dispatch(action) → 명시적 통지 → 셀렉터로 구독
  Proxy   : state.x = 1 → set 트랩이 자동 감지 → 통지 (MobX/Valtio)
  Signal  : count() 읽으면 자동 구독 → set 시 그 구독자만 (Solid)

Signal 자동 의존성:
  const [count, setCount] = signal(0);
  const double = computed(() => count() * 2);
  // double 계산 중 count()를 읽음 → double이 count를 구독
  setCount(1) → count의 구독자(double)만 재계산 → 그 DOM만 갱신
  → VDOM diff 없음

리렌더 정밀도 (같은 변경, 다른 범위):
  Context: 값 변경 → 모든 소비자 리렌더 (넓음)
  Redux+셀렉터: 구독한 슬라이스만 (중간)
  Signal: 그 값을 읽은 DOM 노드만 (가장 좁음)

서버 상태 vs 클라이언트 상태:
  클라이언트: 내가 소유, 동기적, 진실의 원천
  서버:       원격 소유, 비동기, 캐시는 사본 → 진부화·재검증 필요
  → 같은 "상태"로 다루면 안 됨 (React Query가 분리하는 이유)
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 35개 확인 + Profiler/멀티라이브러리 검증 환경 + react-internals/reactivity-state-compared 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
