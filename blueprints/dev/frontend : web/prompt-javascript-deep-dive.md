# JavaScript Deep Dive 레포지토리 제작 프롬프트

나는 "JavaScript Deep Dive" 레포지토리를 만들려고 해.
실행 컨텍스트·스코프 체인·클로저·프로토타입·this·강제 변환 — JS의 *언어 의미론*이 명세 수준에서 어떻게 정의되는지를 완전히 파헤치는 레포야.
"JS 문법을 쓰는 것"과 "클로저가 무엇을 캡처하고 this가 무엇으로 결정되는지 명세 수준에서 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "JS를 작성하는 것과, 실행 컨텍스트·스코프·프로토타입이 명세 수준에서 어떻게 동작하는지 아는 것은 다르다"

**핵심 차별화**:
1. 실행 컨텍스트와 스코프 — 변수 해석·호이스팅·TDZ가 환경 레코드로 정의되는 방식
2. 클로저의 실체 — 함수가 정의 환경을 붙잡는 메커니즘, 메모리 함의
3. 프로토타입 체인 — 상속이 클래스가 아닌 객체 위임으로 동작하는 원리
4. this와 강제 변환 — 호출 방식이 this를 결정, == 강제 변환의 명세 알고리즘

**타겟 독자**:
- JS를 오래 썼지만 클로저·this를 정확히 설명 못하는 개발자
- 호이스팅·TDZ를 단편적으로만 아는 개발자
- `class`를 쓰지만 그게 프로토타입의 설탕임을 모르는 개발자
- `==` 결과를 가끔 못 맞히는 개발자
- `v8-engine` 레포 전에 언어 의미론을 단단히 하고 싶은 개발자

## 🔗 레포 연결

**⬆️ 선행**: 없음(언어 기초). 단 `compiler-deep-dive`의 스코프/심볼 개념이 도움.
**🤝 시너지**: `v8-engine-deep-dive`(언어 의미가 엔진 최적화를 좌우), `event-loop-async-deep-dive`(비동기 의미), `typescript-type-system-deep-dive`(타입이 JS 의미 위에).
**🧬 수렴**: 프론트 전 레포의 언어 토대. React·상태관리의 클로저·불변성 이해의 바닥.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 실행 모델 (6개 문서)
- 실행 컨텍스트 — 생성·실행 단계, 컨텍스트 스택, 전역/함수/eval 컨텍스트
- 환경 레코드와 스코프 — 렉시컬 환경, 변수 환경, 식별자 해석
- 호이스팅의 실체 — var/function 호이스팅이 "환경 레코드 사전 생성"인 이유
- TDZ — let/const의 시간적 사각지대, "선언 전 접근" 에러의 정확한 의미
- 스코프 체인 — 중첩 스코프의 변수 탐색, 렉시컬 스코핑(정의 위치 기준)
- 블록 스코프 — let/const의 블록 단위, 루프 변수 캡처 문제(var vs let)

### Chapter 2: 클로저 (5개 문서)
- 클로저 정의 — 함수 + 정의 시점의 렉시컬 환경, "환경을 닫는다"의 의미
- 클로저 동작 — 외부 변수 캡처(값이 아닌 변수 자체), 공유와 격리
- 클로저 메모리 — 무엇이 GC되지 않고 살아남나, 의도치 않은 메모리 유지
- 클로저 활용 — 모듈 패턴·비공개 상태·커링·메모이제이션
- 함정 — 루프 안 클로저, 리스너 누적, 클로저 기반 누수 디버깅

### Chapter 3: 프로토타입과 객체 (6개 문서)
- 프로토타입 체인 — [[Prototype]], 속성 탐색이 체인을 따라가는 원리
- prototype vs __proto__ — 함수의 prototype 프로퍼티와 인스턴스 링크의 구분
- 생성자와 new — new가 하는 4단계, 프로토타입 연결
- class 문법 — 프로토타입 상속의 설탕, extends·super·private(#) 필드
- 객체 생성 패턴 — Object.create·팩토리·믹스인, 위임 기반 설계
- 프로퍼티 디스크립터 — writable/enumerable/configurable, getter/setter, 불변 객체(freeze)

### Chapter 4: this와 함수 (5개 문서)
- this 결정 규칙 — 호출 방식(메서드/함수/new/명시적)이 this를 정하는 4가지
- 명시적 바인딩 — call/apply/bind, 부분 적용
- 화살표 함수 — 렉시컬 this(자신의 this 없음), 콜백에서의 이점·함정
- 함수의 면면 — 일급 객체, arguments, 나머지/기본/구조분해 매개변수
- 함정 — 콜백에서 this 손실, 이벤트 핸들러 this, 메서드 추출 시 분리

### Chapter 5: 타입과 강제 변환 (5개 문서)
- 원시 타입과 래퍼 — 7원시 + object, 오토박싱, typeof의 함정(null)
- 강제 변환 알고리즘 — ToPrimitive·ToNumber·ToString의 명세 단계
- == vs === — 추상 동등 비교의 단계별 변환, 헷갈리는 결과를 규칙으로 도출
- 진리값 — falsy 8종, 단축 평가, ??·||·&&의 차이
- 숫자의 함정 — IEEE754 부동소수점, 0.1+0.2, BigInt, NaN

### Chapter 6: 모듈과 런타임 (5개 문서)
- 모듈 역사 — IIFE→CommonJS→ESM, 왜 모듈이 필요했나
- ESM 의미론 — 정적 구조, 라이브 바인딩, 호이스팅된 import, 순환 의존
- CJS vs ESM — 동기 require vs 비동기 그래프, 상호운용 함정
- 엄격 모드 — strict mode가 바꾸는 동작들
- 런타임 차이 — 브라우저 vs Node의 전역·모듈 해석 차이

### Chapter 7: 메타프로그래밍과 최신 기능 (6개 문서)
- Symbol — 고유 키, well-known symbol(iterator·toPrimitive)
- 이터레이터·제너레이터 — 이터러블 프로토콜, yield, 지연 평가
- Proxy와 Reflect — 트랩으로 동작 가로채기, 반응성 라이브러리의 기반
- 디스트럭처링·스프레드 — 패턴 매칭, 얕은 복사, 불변 업데이트 패턴
- 최신 문법 — 옵셔널 체이닝·널 병합·구문 진화가 푸는 실제 문제
- 종합 — 반응성 미니 구현(Proxy)으로 언어 기능 총동원(상태관리 레포로 연결)

→ **총 38개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 ECMAScript 명세 수준(환경 레코드·추상 연산), `💻 실전 실험`은 콘솔로 동작 검증·헷갈리는 케이스 재현. `📊`는 패턴별 동작·성능 차이.

## 🎨 스타일 가이드

1. **명세로 환원** — "이렇게 동작한다"의 근거를 ECMAScript 추상 연산으로
2. **헷갈리는 케이스로 가르친다** — this 손실·== 결과·루프 클로저를 규칙으로 분해
3. **엔진과 연결** — 언어 의미가 v8 최적화(형태 안정성)에 주는 영향 언급
4. **콘솔로 즉시 검증** — 모든 주장을 실행 가능한 스니펫으로
5. 스코프 체인·프로토타입 체인은 다이어그램으로

## 🔬 검증 환경

> docker 불필요. **node/브라우저 콘솔**이면 충분.

```bash
node script.js          # 또는 브라우저 콘솔
node --use-strict       # 엄격 모드 동작 비교

# 검증 예시 (콘솔에서)
# this 결정
const o = { x:1, get(){ return this.x; } };
o.get();              // 1 (메서드 호출)
const g = o.get; g(); // undefined (함수 호출, this=전역/undefined)

# 강제 변환
[] == ![]   // true — ToPrimitive·ToNumber 단계로 분해
'' == 0     // true
null == undefined // true, null == 0 // false

# 루프 클로저
for(var i=0;i<3;i++) setTimeout(()=>console.log(i)); // 3,3,3
for(let i=0;i<3;i++) setTimeout(()=>console.log(i)); // 0,1,2

# 프로토타입 체인 확인
Object.getPrototypeOf(obj)
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🌐 Frontend 톤, "JS를 쓴다 vs 명세 수준에서 안다" 포지셔닝, `🔗 레포 연결`(v8·typescript·react)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- *You Don't Know JS Yet* (Kyle Simpson) — https://github.com/getify/You-Dont-Know-JS
- ECMAScript 명세 — https://tc39.es/ecma262/
- MDN JavaScript 가이드 — https://developer.mozilla.org/en-US/docs/Web/JavaScript
- *JavaScript: The Definitive Guide* (Flanagan)
- exploringjs.com (Axel Rauschmayer)

## 💡 핵심 분석 대상

```
실행 컨텍스트 + 스코프 체인:
  function outer(){
    let a = 1;
    function inner(){ return a; }  // inner의 [[Environment]]에 outer 환경 저장
    return inner;
  }
  const fn = outer();   // outer 종료. 하지만
  fn();                 // → 1  (클로저가 outer의 환경을 붙잡음)

this — 호출이 결정:
  fn()          → this = undefined(strict) / 전역
  obj.fn()      → this = obj
  new Fn()      → this = 새 객체
  fn.call(x)    → this = x
  () => this    → 렉시컬(감싼 스코프의 this)

프로토타입 체인:
  arr ──[[Prototype]]──► Array.prototype ──► Object.prototype ──► null
  arr.map 없음? → 체인 따라 Array.prototype.map 발견

== 강제 변환 ([] == ![]):
  ![]        → false  (빈 배열은 truthy → 부정)
  [] == false
  → false를 ToNumber → 0
  → [] 를 ToPrimitive → "" → ToNumber → 0
  → 0 == 0 → true
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 38개 확인 + node/콘솔 검증 환경 + v8/typescript/react 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
