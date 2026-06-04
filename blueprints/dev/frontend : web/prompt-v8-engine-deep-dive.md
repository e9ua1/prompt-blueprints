# V8 Engine Deep Dive 레포지토리 제작 프롬프트

나는 "V8 Engine Deep Dive" 레포지토리를 만들려고 해.
JavaScript가 어떻게 바이트코드로 해석되고, 핫 코드가 어떻게 단계적으로(Ignition→Sparkplug→Maglev→TurboFan) 기계어로 최적화되며, Hidden Class·Inline Cache가 동적 언어를 어떻게 빠르게 만드는지를 완전히 파헤치는 레포야.
"JS를 작성하는 것"과 "엔진이 그것을 어떻게 최적화하고 왜 역최적화하는지 알고 빠른 코드를 짜는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "JS를 짜는 것과, V8이 그것을 바이트코드→JIT로 어떻게 변환하고 객체를 어떻게 표현하는지 아는 것은 다르다"

**핵심 차별화**:
1. 티어드 JIT — 인터프리터로 시작해 핫 코드만 점점 공격적으로 컴파일하는 이유와 단계
2. Hidden Class & Inline Cache — 동적 객체를 정적처럼 빠르게 만드는 핵심 메커니즘
3. 역최적화(Deopt) — 투기 최적화가 틀렸을 때 되돌리는 과정, deopt를 유발하는 코드
4. Orinoco GC — 세대별·동시·병렬 GC가 멈춤 없이 메모리를 회수하는 방식

**타겟 독자**:
- JS를 쓰지만 "이 코드가 왜 느린지" 엔진 관점에서 설명 못하는 개발자
- "객체 형태를 일관되게 유지하라"는 조언을 들었지만 이유(Hidden Class)를 모르는 개발자
- JIT·역최적화를 단어로만 아는 개발자
- Node 성능 문제를 만나지만 V8 내부가 블랙박스인 개발자
- `compiler-deep-dive`에서 JIT를 봤고 실제 엔진에서 보고 싶은 개발자

## 🔗 레포 연결

**⬆️ 선행**: `compiler-deep-dive`(파싱·IR·JIT 티어의 일반 이론), `computer-architecture-deep-dive`(기계어·캐시).
**🤝 시너지**: `javascript-deep-dive`(언어 의미가 엔진 동작을 결정), `event-loop-async-deep-dive`(엔진 + 런타임 경계), `browser-rendering-deep-dive`(JS가 메인 스레드 점유).
**🧬 수렴**: `memory-management-compared`(Orinoco GC ↔ JVM·Go·ART GC), `concurrency-models-compared`.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: V8 아키텍처 (5개 문서)
- V8 전체 구조 — 파서·Ignition·Sparkplug·Maglev·TurboFan·Orinoco의 역할 분담
- 엔진과 런타임의 경계 — V8(엔진) vs Node/브라우저(이벤트 루프·API), 무엇이 어디 소속인가
- JS 실행 한눈에 — 소스→AST→바이트코드→(핫)→최적화 기계어
- d8로 들여다보기 — V8 디버그 셸, 내부 플래그로 엔진 동작 관찰하는 법
- 동적 언어를 빠르게 만드는 문제 — 타입이 런타임에만 있는 언어를 최적화하는 도전

### Chapter 2: 파싱과 바이트코드 — Ignition (6개 문서)
- 파싱 — lazy 파싱·preparse, 함수가 호출될 때까지 미루는 이유
- AST와 스코프 — 스코프 분석, 클로저 변수 캡처 표현
- Ignition 바이트코드 — 레지스터 기반 바이트코드, 왜 바이트코드로 시작하나(메모리 절약)
- 바이트코드 읽기 — `--print-bytecode`로 함수의 바이트코드 분석
- 피드백 벡터(Feedback Vector) — 실행 중 타입 정보 수집, 최적화의 연료
- 핫 함수 감지 — 호출 횟수·루프 카운터로 "뜨거운" 코드 판정, 티어업 트리거

### Chapter 3: JIT 티어 — Sparkplug부터 TurboFan까지 (6개 문서)
- 티어드 컴파일 철학 — 빠른 시작 vs 빠른 피크, 단계마다 컴파일 비용↑ 실행 속도↑
- Sparkplug — 비최적화 베이스라인 JIT, 바이트코드를 빠르게 기계어로
- Maglev — 중간 티어 최적화 컴파일러, SSA 기반, 적당한 최적화
- TurboFan — 최상위 최적화, Sea of Nodes IR, 공격적 투기 최적화
- 최적화 기법 — 인라이닝·타입 특수화·이스케이프 분석·범위 체크 제거
- 티어업/티어다운 — 언제 다음 티어로, 가정 깨지면 어떻게 내려오나

### Chapter 4: 객체 모델 — Hidden Class & Inline Cache (6개 문서)
- 객체 표현 — 프로퍼티 저장(in-object vs properties backing store), 동적 추가의 비용
- Hidden Class(Map) — 같은 형태 객체가 구조를 공유, 전이(transition) 트리
- 형태(Shape) 안정성 — 프로퍼티 추가 순서가 형태를 가르는 이유, 형태 분기 회피
- Inline Cache(IC) — 프로퍼티 접근 지점이 형태를 기억, monomorphic/polymorphic/megamorphic
- IC 상태 전이 — 한 형태(빠름)→여러 형태→포기, 성능 단계적 하락
- 배열 최적화 — packed vs holey, SMI/double/object elements kind, 구멍이 느린 이유

### Chapter 5: 역최적화와 성능 함정 (6개 문서)
- 역최적화(Deopt) — 투기 가정 위반 시 최적화 코드 폐기·인터프리터 복귀, bailout
- deopt 유발 패턴 — 형태 변경·`arguments` 누수·try/catch(과거)·타입 불안정
- `--trace-opt`/`--trace-deopt` — 최적화/역최적화 로그 읽기, deopt 이유 코드
- 인라이닝 한계 — 인라인되는/안 되는 함수, 메가모픽 호출의 비용
- 숨은 비용 — 클로저·이터레이터·구조 분해의 엔진 레벨 비용 측정
- "최적화 가능한 JS" 작성법 — 형태 일관성·단형 함수·예측 가능한 코드

### Chapter 6: 가비지 컬렉션 — Orinoco (6개 문서)
- 힙 구조 — new space(young) / old space, large object space, 세대 가설
- Scavenge(Minor GC) — 반공간 복사, young 객체의 빠른 회수
- Major GC(Mark-Compact) — 표시·쓸기·압축, old space 회수
- 동시·병렬·증분 GC — 메인 스레드 멈춤을 줄이는 Orinoco의 기법들
- 쓰기 배리어와 세대 간 참조 — old→young 참조 추적(remembered set)
- 메모리 누수 패턴 — 의도치 않은 참조 유지(클로저·전역·리스너), heap snapshot으로 추적

### Chapter 7: 측정과 실전 (5개 문서)
- 프로파일링 — `node --prof`·`--cpu-prof`, Chrome DevTools Performance/Memory
- 벤치마킹 함정 — JIT 워밍업·죽은 코드 제거를 고려한 신뢰성 있는 측정
- 메모리 분석 — heap snapshot, retained size, detached DOM 추적
- `%` 네이티브 문법 — `--allow-natives-syntax`로 최적화 상태 직접 질의
- 케이스 스터디 — 느린 함수를 trace-deopt로 진단→형태 안정화→before/after

→ **총 40개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 바이트코드·Hidden Class·IC, `💻 실전 실험`은 d8/node 플래그로 *실제 엔진 출력* 관찰. `📊`는 mono vs mega·deopt 전후의 정량 비교.

## 🎨 스타일 가이드

1. **엔진 출력으로 증명** — "최적화된다"가 아니라 `--trace-opt` 로그로
2. **형태(Shape) 중심 서술** — 대부분의 성능 차이를 Hidden Class/IC로 환원
3. **compiler 레포와 연결** — 일반 JIT 이론(compiler)이 V8에서 어떻게 구체화되나
4. **나쁜 패턴→deopt 로그→개선** 순서
5. 티어 전이·Hidden Class 전이는 다이어그램으로

## 🔬 검증 환경

> docker 불필요. **d8/node의 V8 플래그**가 검증 도구.

```bash
# node에 V8 플래그 전달 (또는 d8 빌드)
node --print-bytecode --print-bytecode-filter=myFn script.js   # 바이트코드
node --trace-opt --trace-deopt script.js                       # 최적화/역최적화 로그
node --allow-natives-syntax script.js                          # %문법 사용

# script.js 안에서 최적화 상태 직접 질의
function f(o){ return o.x; }
%PrepareFunctionForOptimization(f);
f({x:1}); f({x:1});
%OptimizeFunctionOnNextCall(f);
f({x:1});
console.log(%GetOptimizationStatus(f));   // 최적화됨? 확인

# 형태 분기 관찰: 같은 함수에 다른 형태 객체 → IC가 mega로
# GC 관찰
node --trace-gc script.js
# 프로파일
node --cpu-prof script.js    # → DevTools에서 로드
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🌐 Frontend 톤, "JS를 짠다 vs 엔진이 어떻게 최적화하나" 포지셔닝, `🔗 레포 연결`(compiler·memory-management-compared)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- V8 공식 블로그 — https://v8.dev/blog
- "Ignition·TurboFan·Maglev" V8 팀 글들 — https://v8.dev/docs
- *JavaScript engine fundamentals* (Mathias Bynens & Benedikt Meurer)
- Franziska Hinkelmann 발표들
- V8 소스 — https://github.com/v8/v8

## 💡 핵심 분석 대상

```
티어드 JIT (빠른 시작 → 빠른 피크):
  소스 → AST → Ignition 바이트코드 (인터프리터, 즉시 시작)
                  │ 핫?
                  ▼ Sparkplug (베이스라인 JIT, 빠른 컴파일)
                  │ 더 핫?
                  ▼ Maglev (중간 최적화)
                  │ 최핫?
                  ▼ TurboFan (공격적 최적화, 투기)
                       │ 가정 깨짐
                       ▼ Deopt → 바이트코드로 복귀

Hidden Class:
  function P(x,y){ this.x=x; this.y=y; }
  new P(1,2), new P(3,4) → 같은 Hidden Class 공유 (형태 동일)
  반면 객체마다 프로퍼티 추가 순서 다르면 → 형태 분기 → 느려짐

Inline Cache:
  function getX(o){ return o.x; }   // 이 접근 지점이 형태를 기억
  항상 같은 형태 → monomorphic (가장 빠름)
  2~4개 형태   → polymorphic
  많은 형태    → megamorphic (IC 포기, 느림)

Deopt 유발:
  function add(a,b){ return a+b; }
  add(1,2)         // 숫자로 최적화
  add("x","y")     // 문자열! 가정 깨짐 → deopt → 재최적화
  → 타입 안정성이 성능을 좌우
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 40개 확인 + V8 플래그 검증 환경 + compiler/javascript/memory-management-compared 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
