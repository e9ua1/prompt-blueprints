# TypeScript Type System Deep Dive 레포지토리 제작 프롬프트

나는 "TypeScript Type System Deep Dive" 레포지토리를 만들려고 해.
구조적 타이핑·제네릭·조건부/매핑 타입·타입 추론이 *타입 레벨 프로그래밍 언어*로서 어떻게 동작하고, 컴파일러(Checker)가 타입을 어떻게 검사하는지를 완전히 파헤치는 레포야.
"타입 어노테이션을 다는 것"과 "타입 시스템이 무엇을 증명하고 추론하며 어디서 한계를 갖는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "타입을 다는 것과, 타입 시스템이 어떻게 추론·검사하고 무엇을 보장(못)하는지 아는 것은 다르다"

**핵심 차별화**:
1. 구조적 타이핑 — "이름이 아니라 모양", Java(명목적)와의 근본 차이
2. 타입은 프로그래밍 언어다 — 조건부·매핑·재귀 타입으로 타입 레벨 연산
3. 추론의 원리 — 컴파일러가 어노테이션 없이 타입을 좁히고 넓히는 알고리즘
4. 건전성(soundness)의 의도된 구멍 — TS가 실용을 위해 안전성을 양보한 지점들

**타겟 독자**:
- TS를 쓰지만 복잡한 제네릭·조건부 타입 앞에서 막히는 개발자
- `any`로 도망치고 타입 에러의 원인을 못 읽는 개발자
- 구조적 타이핑이 뭔지, 왜 가끔 예상외로 통과/거부되는지 모르는 개발자
- 라이브러리 타입 정의를 읽지 못하는 개발자
- `javascript-deep-dive`를 마치고 타입 레이어를 단단히 하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `javascript-deep-dive`(타입은 JS 의미 위에), `compiler-deep-dive`(타입 검사·추론의 일반 이론).
**🤝 시너지**: `react-internals-deep-dive`(컴포넌트 타이핑), `rust-deep-dive`(타입 시스템 표현력 대조).
**🧬 수렴**: 프론트 코드 품질의 토대. `compiler-deep-dive`의 타입 추론(Ch4)의 실전 사례.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 구조적 타이핑과 기본 (5개 문서)
- 구조적 타이핑 — 모양이 맞으면 호환, 명목적(Java/Rust)과의 차이와 함의
- 타입 vs 값 공간 — 타입 세계와 값 세계의 분리, `typeof`·`keyof`의 다리
- 집합으로서의 타입 — 타입을 값의 집합으로 보기, 서브타입 = 부분집합
- 유니온과 인터섹션 — 합집합·교집합, 좁히기의 기반
- 리터럴 타입과 와이드닝 — 리터럴 타입, const 추론, 넓혀짐(widening)

### Chapter 2: 제네릭 (5개 문서)
- 제네릭 기본 — 타입 파라미터, 함수·클래스·타입의 일반화
- 제약(constraints) — `extends`로 파라미터 제한, keyof 제약
- 제네릭 추론 — 인자에서 타입 파라미터를 역추론하는 원리
- 기본 타입 파라미터·분산 — 기본값, 다중 파라미터 관계
- 제네릭 설계 — 과한 제네릭의 함정, 추론 친화적 시그니처

### Chapter 3: 조건부·매핑 타입 — 타입 레벨 프로그래밍 (6개 문서)
- 조건부 타입 — `T extends U ? X : Y`, 타입 레벨 분기
- infer — 조건부 안에서 타입 추출(ReturnType·Parameters의 원리)
- 분배 조건부 — 유니온에 조건부가 분배되는 동작, 분배 끄기
- 매핑 타입 — `{ [K in keyof T]: ... }`, Partial·Readonly·Pick의 구현
- 키 재매핑·템플릿 리터럴 타입 — as로 키 변형, 문자열 타입 연산
- 재귀 타입 — 재귀 조건부/매핑으로 깊은 변환, 깊이 제한

### Chapter 4: 타입 추론과 좁히기 (5개 문서)
- 추론 알고리즘 — 컨텍스트 타입·후보 수집·최적 공통 타입
- 제어 흐름 분석 — typeof/instanceof/in/truthy로 타입 좁히기
- 타입 가드 — 사용자 정의 가드(`is`), assertion 함수
- 판별 유니온 — discriminant로 안전한 분기, 망라성 검사(never)
- 좁히기 함정 — 좁힘이 풀리는 경우(클로저·재할당), as const

### Chapter 5: 변성과 고급 관계 (5개 문서)
- 변성(variance) — 공변/반공변/이변, 함수 매개변수의 반공변
- 할당 가능성 — 무엇이 무엇에 할당 가능한가의 규칙
- 건전성의 구멍 — bivariance·any·타입 단언·인덱스 접근의 의도된 비건전성
- 과잉 프로퍼티 검사 — 리터럴에만 적용되는 추가 검사의 이유
- 타입 좁히기 vs 단언 — 안전한 좁히기와 위험한 단언의 경계

### Chapter 6: 컴파일러 내부 (6개 문서)
- 컴파일 파이프라인 — 스캐너·파서·바인더·체커·이미터(compiler 레포 연결)
- 바인더와 심볼 — 선언을 심볼로, 스코프 구성
- 체커 동작 — 타입 관계 계산, 구조적 비교의 실제
- 타입 표현 — 컴파일러 내부의 타입 객체, 캐싱
- 성능 — 타입 검사가 느려지는 패턴, `--extendedDiagnostics`로 진단
- 선언 파일(.d.ts) — 타입만의 계약, 라이브러리 타이핑, declaration merging

### Chapter 7: 실전 타입 설계 (5개 문서)
- API 타이핑 — 함수 오버로드 vs 제네릭, 추론을 살리는 설계
- 유틸리티 타입 재구현 — Pick/Omit/Exclude를 직접 만들며 이해
- 타입 안전 패턴 — 브랜딩(명목적 흉내)·exhaustiveness·빌더
- any와의 전쟁 — unknown·제네릭·가드로 any 제거
- 종합 — 타입 챌린지 난제를 조건부·infer·재귀로 풀어내기

→ **총 37개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 체커 동작·타입 관계 계산, `💻 실전 실험`은 타입 챌린지·`tsc` 진단으로 검증. `📊`는 타입 설계별 추론 품질·컴파일 시간 비교.

## 🎨 스타일 가이드

1. **타입=집합 직관** — 모든 관계를 부분집합/할당가능성으로 환원
2. **에러 메시지 읽기** — TS 에러를 단계별로 해독하는 법
3. **직접 만들며** — 유틸리티 타입을 재구현하며 매핑/조건부 체득
4. **건전성 구멍 명시** — "여기서 TS는 안전성을 양보한다"를 분명히
5. **compiler 레포와 연결** — 일반 타입 검사 이론이 TS에서 구체화

## 🔬 검증 환경

> docker 불필요. **tsc + 타입 도구**가 검증 도구. 런타임 없이 타입만.

```bash
npm i -g typescript
tsc --noEmit --strict file.ts          # 타입만 검사
tsc --extendedDiagnostics file.ts      # 체크 시간·타입 인스턴스화 수
tsc --explainFiles                     # 어떤 파일이 왜 포함됐나
tsc --generateTrace traceDir           # 컴파일러 트레이스(성능 분석)

# 타입 레벨 단언 (런타임 없이 타입 검증)
type Expect<T extends true> = T;
type Equal<X, Y> = (<T>() => T extends X ? 1 : 2) extends
                   (<T>() => T extends Y ? 1 : 2) ? true : false;
type _ = Expect<Equal<MyType, ExpectedType>>;   // 틀리면 컴파일 에러

# 도구
# - TypeScript Playground: 타입 호버로 추론 결과 확인
# - ts-ast-viewer.com: AST·타입 구조 시각화
# - type-challenges 레포로 연습
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🌐 Frontend 톤, "타입을 단다 vs 타입 시스템이 어떻게 동작하나" 포지셔닝, `🔗 레포 연결`(javascript·compiler·react)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- TypeScript Handbook — https://www.typescriptlang.org/docs/handbook/
- *Effective TypeScript* (Dan Vanderkam)
- type-challenges — https://github.com/type-challenges/type-challenges
- "Type System Internals" TS 위키 — https://github.com/microsoft/TypeScript/wiki
- Total TypeScript (Matt Pocock)

## 💡 핵심 분석 대상

```
구조적 vs 명목적:
  interface Point { x:number; y:number; }
  const p = { x:1, y:2, z:3 };
  const pt: Point = p;     // ✓ TS: 모양이 맞으면 OK(구조적)
  // Java였다면 Point를 implements 안 했으니 ✗(명목적)

타입 = 집합:
  type A = 'a' | 'b';           // {'a','b'}
  type B = string;              // 모든 문자열
  A extends B ? ... : ...       // A ⊆ B → true

조건부 + infer (ReturnType 원리):
  type ReturnType<T> = T extends (...a:any)=>infer R ? R : never;
  //                          함수면 반환타입 R을 추출 ───┘

매핑 타입 (Partial 원리):
  type Partial<T> = { [K in keyof T]?: T[K] };
  //                  T의 모든 키를 돌며 옵셔널로

분배 조건부:
  type ToArray<T> = T extends any ? T[] : never;
  ToArray<string | number>  // string[] | number[] (유니온에 분배됨)

건전성 구멍 (의도적):
  const arr: number[] = [1,2];
  const ro: readonly number[] = arr;  // OK
  (ro as number[]).push(3);           // 단언으로 우회 가능 → 비건전
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 37개 확인 + tsc/타입챌린지 검증 환경 + javascript/compiler/react 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
