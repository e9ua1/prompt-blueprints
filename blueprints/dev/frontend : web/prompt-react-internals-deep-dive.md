# React Internals Deep Dive 레포지토리 제작 프롬프트

나는 "React Internals Deep Dive" 레포지토리를 만들려고 해.
Fiber 아키텍처·재조정(Reconciliation)·Hooks·Concurrent 렌더링이 *실제로 어떻게 구현*되는지 — 선언적 UI가 어떻게 효율적인 DOM 변경으로 번역되는지를 완전히 파헤치는 레포야.
"React를 사용하는 것"과 "Fiber가 왜 일을 쪼개고 Hooks가 어떻게 상태를 기억하며 언제 리렌더가 일어나는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "React로 컴포넌트를 짜는 것과, Fiber·재조정·Hooks가 내부에서 어떻게 동작하는지 아는 것은 다르다"

**핵심 차별화**:
1. Fiber 아키텍처 — 렌더링을 중단·재개 가능한 작업 단위로 쪼갠 이유와 구조
2. 재조정과 Diffing — 두 트리를 비교해 최소 DOM 변경을 찾는 휴리스틱(key의 진짜 역할)
3. Hooks의 실체 — 함수형 컴포넌트가 클로저로 상태를 기억하는 메커니즘, 규칙의 이유
4. Concurrent 렌더링 — Lane 우선순위·Transition·Suspense가 응답성을 지키는 방식

**타겟 독자**:
- React를 쓰지만 "왜 리렌더되는지" 정확히 설명 못하는 개발자
- key·useMemo·useCallback을 외워 쓰지만 원리를 모르는 개발자
- Hooks 규칙(조건부 호출 금지)의 이유를 모르는 개발자
- Concurrent·Suspense·RSC를 단어로만 아는 개발자
- 가상 DOM이 "빠르다"고만 알고 메커니즘을 모르는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `javascript-deep-dive`(클로저가 Hooks의 핵심), `browser-rendering-deep-dive`(결국 DOM 변경→렌더), `event-loop-async-deep-dive`(스케줄링).
**🤝 시너지**: `frontend-state-management-deep-dive`(React 바깥 상태), `rendering-strategy-deep-dive`(RSC·SSR), `react-native-deep-dive`(같은 reconciler, 다른 renderer).
**🧬 수렴**: `reactivity-state-compared`(React 리렌더 ↔ Signal·Compose Snapshot·SwiftUI), `rendering-pipelines-compared`.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: React의 철학과 Element (6개 문서)
- 선언적 UI — "어떻게"가 아닌 "무엇을", UI = f(state)의 의미
- JSX와 Element — JSX가 createElement 호출로 변환, Element는 가벼운 객체(인스턴스 아님)
- Element vs Component vs Instance — 셋의 구분, 컴포넌트는 함수/클래스
- 가상 DOM의 동기 — 직접 DOM 조작의 비용(browser-rendering 연결), 추상화의 이득과 비용
- 렌더와 커밋 2단계 — Render(계산, 중단 가능) vs Commit(DOM 반영, 동기)
- React의 구성 — reconciler(코어) + renderer(DOM/Native), 분리 설계

### Chapter 2: Fiber 아키텍처 (7개 문서)
- Fiber란 — 작업 단위이자 노드, 왜 스택 기반에서 Fiber로 바뀌었나(중단 가능성)
- Fiber 노드 구조 — type·key·child·sibling·return, 연결 리스트로 트리 표현
- 작업 루프(Work Loop) — beginWork/completeWork, 단위 작업의 분할
- 더블 버퍼링 — current 트리 vs workInProgress 트리, alternate 포인터
- 시간 분할 — 작업을 쪼개 메인 스레드 양보, 긴 렌더가 입력을 막지 않게
- 스케줄러 — 우선순위 기반 작업 예약, 브라우저 양보(scheduler 패키지)
- effect 수집 — 변경된 Fiber 표시, 커밋 단계로 넘길 부수효과 리스트

### Chapter 3: 재조정(Reconciliation)과 Diffing (6개 문서)
- 재조정 개요 — 새 Element 트리 vs 기존 Fiber 트리 비교, 무엇을 재사용/생성/삭제
- Diffing 휴리스틱 — O(n³)을 O(n)으로 만든 두 가정(타입·key)
- 타입 비교 — 같은 타입이면 업데이트, 다르면 언마운트→마운트(서브트리 전부)
- key의 역할 — 리스트 재정렬 시 동일성 추적, key 없을 때/index key의 버그
- bailout — props·state 동일 시 서브트리 작업 건너뛰기, 리렌더 최적화의 토대
- 리렌더의 진실 — 부모 리렌더가 자식에 미치는 영향, memo의 작동 지점

### Chapter 4: Hooks 내부 (7개 문서)
- Hooks가 푸는 문제 — 클래스의 문제, 로직 재사용, 함수형 + 상태
- Hooks 저장 구조 — Fiber에 연결된 hook 연결 리스트, 호출 순서로 매칭
- 호출 순서 규칙의 이유 — 왜 조건부/루프 안에서 호출 금지인가(인덱스 매칭 붕괴)
- useState/useReducer — 상태 저장·업데이트 큐·배칭, 함수형 업데이트
- useEffect/useLayoutEffect — 의존성 비교·정리(cleanup)·실행 타이밍 차이
- useMemo/useCallback/useRef — 메모이제이션 저장, ref의 가변 컨테이너
- 클로저 함정 — stale closure, 의존성 배열 실수, 원인과 해결

### Chapter 5: Concurrent 렌더링 (7개 문서)
- Concurrent의 의미 — 렌더를 중단·재개·버림, "동시"가 아닌 "가변 우선순위"
- Lane 모델 — 우선순위를 비트마스크로, 여러 업데이트의 우선순위 병합
- 배칭 — 자동 배칭(React 18), 여러 setState를 한 렌더로
- startTransition — 긴급 업데이트와 전환 업데이트 분리, 응답성 유지
- useDeferredValue — 값의 지연 처리, 입력 응답 우선
- 중단과 tearing — 동시 렌더 중 일관성 문제, useSyncExternalStore의 이유
- 시간 분할 실전 — 무거운 렌더를 transition으로 부드럽게(측정)

### Chapter 6: Suspense와 서버 컴포넌트 (6개 문서)
- Suspense 메커니즘 — promise를 throw해 폴백 표시, 경계의 동작
- 데이터 페칭과 Suspense — 선언적 로딩, 워터폴 회피
- 서버 컴포넌트(RSC) — 서버에서만 렌더되는 컴포넌트, 번들 제외, 직렬화
- 스트리밍 SSR — 점진적 HTML 전송, Selective Hydration(rendering-strategy 연결)
- 하이드레이션 — 서버 HTML에 이벤트 연결, 불일치(mismatch) 문제
- 경계 설계 — Suspense·Error Boundary 배치 전략

### Chapter 7: 성능과 측정 (6개 문서)
- 리렌더 측정 — React DevTools Profiler, 왜 리렌더됐는지 추적
- 최적화 도구 — memo·useMemo·useCallback을 *언제* 쓰나(과용의 비용)
- 상태 구조와 리렌더 — 상태 위치(colocation)·분할로 리렌더 범위 축소
- Context와 리렌더 — Context 값 변경이 모든 소비자를 리렌더하는 문제
- 큰 리스트 — 가상화(windowing)의 원리
- 종합 — 느린 React 앱을 Profiler로 진단→구조 개선→before/after

→ **총 45개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 Fiber·Hooks 연결 리스트·재조정 알고리즘(소스 수준), `💻 실전 실험`은 DevTools Profiler + 미니 reconciler 구현. `📊`는 최적화 전후 리렌더 횟수·시간.

## 🎨 스타일 가이드

1. **직접 만들며** — 미니 가상 DOM·미니 Hooks(useState)를 구현하며 원리 체득
2. **"왜 리렌더?"에 답한다** — 모든 성능 논의를 재조정·bailout으로 환원
3. **클로저로 Hooks 설명** — javascript 레포의 클로저가 여기서 결정적
4. **규칙의 이유** — "하지 마라"를 외우지 말고 *왜 깨지는지* 보여준다
5. Fiber 트리·더블 버퍼링은 다이어그램으로

## 🔬 검증 환경

> docker 불필요. **React DevTools Profiler + 미니 구현**이 검증 도구.

```bash
npx create-vite react-internals --template react-ts
# React DevTools(브라우저 확장) 설치

# 검증 방법
# 1) Profiler: 커밋별 렌더 시간·리렌더 컴포넌트·"why did this render"
# 2) <Profiler onRender={...}> API로 렌더 비용 로깅
# 3) StrictMode로 이중 호출(부수효과 검출) 관찰
# 4) why-did-you-render 라이브러리로 불필요 리렌더 탐지

# 미니 구현 트랙(레포 핵심 산출물)
# - createElement + render로 가상 DOM → 실제 DOM
# - 간단한 reconciler(타입·key diff)
# - useState를 클로저+인덱스로 직접 구현 → "호출 순서 규칙"의 이유 체득
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `mini-react/`(누적 구현)
2. **README.md**: 🌐 Frontend 톤, "React를 쓴다 vs 내부가 어떻게 도나" 포지셔닝, `🔗 레포 연결`(javascript·reactivity-state-compared·react-native)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- React 공식 (Learn·Reference) — https://react.dev/
- "React as a UI Runtime" (Dan Abramov) — https://overreacted.io/react-as-a-ui-runtime/
- "Build your own React" (Rodrigo Pombo) — https://pomb.us/build-your-own-react/
- React 소스 — https://github.com/facebook/react
- Jser·React 내부 분석 글들

## 💡 핵심 분석 대상

```
렌더 + 커밋 2단계:
  setState
    │
    ▼ Render Phase (중단 가능, 순수 계산)
    workInProgress 트리 구성, 재조정, effect 수집
    │
    ▼ Commit Phase (동기, 중단 불가)
    DOM 반영 → useLayoutEffect → 페인트 → useEffect

Fiber 더블 버퍼링:
  current 트리 ◄──alternate──► workInProgress 트리
  (화면에 보임)                  (계산 중)
  완료되면 포인터 스왑 → workInProgress가 current로

Diffing (key의 역할):
  [A, B, C] → [B, C, A]
  key 없음/index: 위치로 비교 → 전부 업데이트(낭비/버그)
  key=id:        동일 요소 추적 → 이동만 → 효율적

Hooks 호출 순서 매칭:
  function C(){
    const [a] = useState(1);  // hook[0]
    const [b] = useState(2);  // hook[1]
  }
  → Fiber에 hook 리스트 [상태1, 상태2]
  → 매 렌더 같은 순서로 호출돼야 인덱스가 맞음
  → 조건부 호출 시 인덱스 어긋남 → 규칙 위반

Lane 우선순위:
  입력(긴급) > transition(낮음)
  타이핑 중 무거운 리스트 갱신을 startTransition으로 → 입력 안 막힘
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 45개 확인 + DevTools Profiler/미니구현 검증 환경 + javascript/reactivity-state-compared/react-native 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
