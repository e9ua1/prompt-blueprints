# Event Loop & Async Deep Dive 레포지토리 제작 프롬프트

나는 "Event Loop & Async Deep Dive" 레포지토리를 만들려고 해.
단일 스레드 JS가 어떻게 동시성을 흉내내는지 — 태스크/마이크로태스크 큐, Promise 내부, async/await의 상태머신 변환, 브라우저와 Node(libuv)의 차이를 완전히 파헤치는 레포야.
"async/await를 쓰는 것"과 "이벤트 루프가 콜백·Promise·렌더링을 어떤 순서로 처리하는지 알고 설계하는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "비동기 코드를 짜는 것과, 이벤트 루프가 그것을 어떤 순서·어떤 큐로 처리하는지 아는 것은 다르다"

**핵심 차별화**:
1. 단일 스레드 동시성 — 한 스레드로 수천 연결을 처리하는 이벤트 루프의 원리
2. 태스크 vs 마이크로태스크 — Promise가 setTimeout보다 먼저 실행되는 이유와 순서 규칙
3. async/await의 실체 — 문법 설탕이 어떻게 Promise + 상태머신으로 변환되나
4. 브라우저 vs Node — 렌더링 단계가 끼는 브라우저 루프 vs libuv 페이즈 기반 Node 루프

**타겟 독자**:
- async/await를 쓰지만 실행 순서를 정확히 예측 못하는 개발자
- "마이크로태스크"를 들어봤지만 태스크와의 차이를 모르는 개발자
- Node와 브라우저의 이벤트 루프가 다르다는 걸 모르는 개발자
- Promise 체인의 타이밍 버그를 디버깅 못하는 개발자
- UI가 멈추는 이유(메인 스레드 블로킹)를 이벤트 루프로 설명 못하는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `javascript-deep-dive`(콜 스택·실행 컨텍스트), `v8-engine-deep-dive`(엔진과 런타임 경계).
**🤝 시너지**: `browser-rendering-deep-dive`(렌더 단계가 루프에 끼는 위치), `linux-for-backend-deep-dive`(epoll = liboop의 기반), `spring-webflux-deep-dive`(Reactor 이벤트 루프 대조).
**🧬 수렴**: `concurrency-models-compared`(이벤트 루프 ↔ 고루틴·가상 스레드·Actor).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 이벤트 루프의 개념 (5개 문서)
- 단일 스레드의 동시성 — 왜 JS는 단일 스레드인가, 블로킹 vs 논블로킹
- 콜 스택과 실행 — 동기 실행, 스택 오버플로, 스택이 비어야 다음 작업
- 이벤트 루프 한 줄 정의 — "스택이 비면 큐에서 꺼내 실행"의 반복
- 런타임 구성 — 엔진(V8) + 큐 + 루프 + 외부 API(타이머·네트워크)의 협업
- 동기 블로킹의 죄 — `while(true)`·무거운 연산이 모든 걸 멈추는 이유

### Chapter 2: 태스크와 마이크로태스크 (5개 문서)
- 매크로태스크 — setTimeout·이벤트·I/O 콜백, 한 번에 하나씩 꺼냄
- 마이크로태스크 — Promise 콜백·queueMicrotask·MutationObserver
- 실행 규칙 — 태스크 하나 → 마이크로태스크 *전부 비우기* → (렌더) → 다음 태스크
- 순서 예측 — setTimeout vs Promise vs sync의 출력 순서를 규칙으로 도출
- 마이크로태스크 기아 — 마이크로태스크가 무한 생성되면 태스크·렌더가 굶는 문제

### Chapter 3: Promise 내부 (5개 문서)
- Promise 상태 머신 — pending/fulfilled/rejected, 한 번만 정착(settle)
- then/catch/finally — 콜백 등록, 마이크로태스크로 예약되는 시점
- Promise 체이닝 — 반환값이 다음 then으로, thenable 흡수, 평탄화
- 에러 전파 — reject 전파, unhandled rejection, catch 위치의 중요성
- 동시성 헬퍼 — all/allSettled/race/any의 의미와 단락(short-circuit) 동작

### Chapter 4: async/await (5개 문서)
- 문법 설탕의 정체 — async 함수가 Promise를 반환, await가 then의 설탕
- 상태머신 변환 — await마다 함수가 일시정지·재개되는 구조(generator 유사)
- await의 타이밍 — await가 마이크로태스크로 나머지를 예약하는 정확한 순서
- 병렬 vs 순차 — 불필요한 await 직렬화, Promise.all로 병렬화
- 함정 — 루프 안의 await, await 누락, async 함수의 에러 처리

### Chapter 5: 브라우저 vs Node (5개 문서)
- 브라우저 이벤트 루프 — HTML 명세의 처리 모델, 렌더링 단계가 끼는 위치
- requestAnimationFrame — 렌더 직전 콜백, 태스크·마이크로태스크와의 순서
- Node 이벤트 루프 — libuv 페이즈(timers·pending·poll·check·close)
- Node 특유 — process.nextTick vs Promise 마이크로태스크 우선순위, setImmediate
- libuv와 OS — epoll/kqueue/IOCP, 스레드 풀(파일 I/O), 네트워크 논블로킹

### Chapter 6: 렌더링·타이밍과의 상호작용 (5개 문서)
- 프레임과 루프 — 한 프레임에 태스크·마이크로태스크·렌더가 어떻게 배치되나
- rAF로 애니메이션 — setTimeout 애니메이션이 끊기는 이유, rAF 동기화
- long task와 INP — 입력 응답을 막는 긴 작업, 측정(Long Tasks API)
- 작업 양보 — scheduler.yield·setTimeout(0)·isInputPending으로 메인 스레드 양보
- requestIdleCallback — 여유 시간에 저우선 작업, deadline 기반

### Chapter 7: 디버깅과 실전 (5개 문서)
- 실행 순서 디버깅 — 복잡한 sync/micro/macro 혼합 코드의 순서 추적법
- 비동기 스택 트레이스 — DevTools의 async stack, 끊긴 컨텍스트 잇기
- 흔한 버그 — race condition·정렬 안 된 await·이벤트 핸들러 누적
- 성능 측정 — Performance API, 마이크로태스크 폭증 탐지
- 종합 사례 — 출력 순서 퀴즈 10선을 규칙으로 완전 분해

→ **총 35개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 큐·루프 단계, `💻 실전 실험`은 출력 순서 코드를 직접 실행해 규칙 검증(node vs 브라우저 비교). `📊`는 직렬 vs 병렬 await의 시간 비교.

## 🎨 스타일 가이드

1. **출력 순서로 가르친다** — 모든 개념을 "이 코드의 출력 순서는?"으로 검증
2. **규칙으로 환원** — "태스크 1개 → 마이크로 전부 → 렌더"를 모든 예제에 적용
3. **브라우저 vs Node 항상 대조** — 같은 코드의 출력이 환경마다 다른 지점
4. **블로킹과 렌더 연결** — 이벤트 루프 점유가 browser-rendering의 jank로 이어짐
5. 루프·큐는 항상 다이어그램으로

## 🔬 검증 환경

> docker 불필요. **node + 브라우저 콘솔**로 출력 순서를 직접 비교.

```bash
# 같은 스크립트를 두 환경에서 실행해 순서 비교
node order.js                # Node (libuv 페이즈)
# 브라우저: 같은 코드를 콘솔/HTML에서 → 차이 관찰(특히 nextTick·rAF)

# order.js 예시
console.log('1 sync');
setTimeout(() => console.log('2 timeout(macro)'), 0);
Promise.resolve().then(() => console.log('3 promise(micro)'));
queueMicrotask(() => console.log('4 microtask'));
console.log('5 sync');
// 예측: 1,5,3,4,2  ← 규칙으로 도출하고 실행으로 검증

# Node 특유 우선순위
process.nextTick(() => console.log('nextTick'));   // Promise보다 먼저
Promise.resolve().then(() => console.log('promise'));

# 직렬 vs 병렬 await 시간 측정
console.time(); await a(); await b(); console.timeEnd();        // 직렬
console.time(); await Promise.all([a(), b()]); console.timeEnd(); // 병렬
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🌐 Frontend 톤, "async를 쓴다 vs 어떤 순서로 처리되나" 포지셔닝, `🔗 레포 연결`
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- "What the heck is the event loop anyway?" (Philip Roberts, JSConf)
- "In The Loop" (Jake Archibald, JSConf) — 태스크/마이크로태스크/렌더
- HTML 명세 Event Loops — https://html.spec.whatwg.org/multipage/webappapis.html#event-loops
- Node.js 이벤트 루프 가이드 — https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick
- libuv 설계 — https://docs.libuv.org/

## 💡 핵심 분석 대상

```
이벤트 루프 한 사이클:

  ┌─► 매크로태스크 1개 실행 (setTimeout/이벤트/IO)
  │        │
  │        ▼
  │   마이크로태스크 큐 *전부* 비우기 (Promise/queueMicrotask)
  │        │
  │        ▼
  │   (브라우저) 렌더 필요? → rAF → style → layout → paint
  │        │
  └────────┘ 반복

출력 순서 규칙:
  console.log('A');                      // sync
  setTimeout(()=>log('B'),0);            // macro
  Promise.resolve().then(()=>log('C'));  // micro
  console.log('D');                      // sync
  → A, D (sync 먼저)
  → C    (마이크로 — 현재 태스크 끝나고 즉시)
  → B    (매크로 — 다음 태스크)

async/await = Promise + 상태머신:
  async function f(){ log(1); await g(); log(2); }
  ≈ function f(){ log(1); return g().then(()=>{ log(2); }); }
  → await 이후는 마이크로태스크로 예약

브라우저 vs Node:
  브라우저: 태스크 → 마이크로 → (렌더) 반복
  Node    : timers→pending→poll→check→close 페이즈,
            각 페이즈 사이 마이크로(+nextTick) 비움
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 35개 확인 + node/브라우저 출력순서 검증 환경 + javascript/browser-rendering/concurrency-models-compared 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
