# Compiler & Language Implementation Deep Dive 레포지토리 제작 프롬프트

나는 "Compiler & Language Implementation Deep Dive" 레포지토리를 만들려고 해.
소스 코드가 토큰→AST→타입 검사→IR→기계어가 되는 전 과정을 직접 작은 컴파일러를 만들며 완전히 파헤치는 레포야.
"언어를 쓰는 것"과 "언어가 어떻게 구현되는지 아는 것"의 차이를 만드는 레포다. JVM·V8·ART·TypeScript가 *전부 컴파일러*라는 사실을 깨닫는 순간, 흩어진 레포들이 하나의 골격으로 묶인다.

## 📋 프로젝트 목표

**컨셉**: "언어를 사용하는 것과, 컴파일러가 그 언어를 어떻게 해석·검증·번역하는지 아는 것은 다르다"

**핵심 차별화**:
1. 직접 만들며 이해 — 토이 언어의 Lexer→Parser→TypeChecker→IR→Codegen을 단계별로 구현
2. 타입 시스템은 증명기다 — 타입 검사가 무엇을 보장하고 무엇을 못 하는지, 추론 알고리즘의 실체
3. 최적화의 원리 — SSA·상수 폴딩·죽은 코드 제거가 IR 위에서 동작하는 방식
4. JIT vs AOT — 같은 코드를 두 전략이 다르게 번역하는 이유, 워밍업과 역최적화

**타겟 독자**:
- 언어를 쓰지만 "컴파일 에러"가 컴파일러의 어느 단계에서 나오는지 모르는 개발자
- JVM/V8 Deep Dive에서 JIT를 봤지만 "최적화"가 구체적으로 뭘 하는지 모르는 개발자
- TypeScript 타입 추론이 어떻게 동작하는지 블랙박스로 두는 개발자
- 파서·인터프리터를 만들어보고 싶지만 시작점을 모르는 개발자
- 정규식·DSL·트랜스파일러를 다루는데 파싱 이론이 없는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `computer-architecture-deep-dive`(코드젠의 타깃이 결국 기계어·레지스터).
**🤝 시너지**: `jvm-deep-dive`/`v8-engine-deep-dive`(JIT는 런타임 컴파일러), `typescript-type-system-deep-dive`(타입 체커의 실전판), `android-runtime-deep-dive`(ART의 AOT).
**🧬 수렴**: 모든 런타임 레포의 *공통 골격*. JVM·V8·ART·Dart VM이 전부 "컴파일러 + 런타임"임을 보여주는 메타 레포.

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 설계해줘:

### Chapter 1: 컴파일러 파이프라인 전체 조망 (5개 문서)
- 컴파일러란 무엇인가 — 소스→토큰→AST→IR→타깃, 각 단계의 책임, 인터프리터와의 경계
- 프론트엔드 vs 백엔드 — 언어 의존부(파싱·타입)와 타깃 의존부(코드젠)의 분리, LLVM이 왜 이 구조인가
- 우리가 만들 토이 언어 정의 — 문법(BNF), 타입, 의미론 명세, 전체 로드맵
- 컴파일 타임 vs 런타임 — 무엇이 컴파일 때 결정되고 무엇이 실행 때 결정되나
- 에러의 위치 — Lexical/Syntax/Semantic/Runtime 에러가 각 단계에서 어떻게 발생하나

### Chapter 2: 어휘 분석(Lexer) (5개 문서)
- 토큰화 — 문자 스트림을 토큰으로, 토큰 종류 설계, 공백·주석 처리
- 정규 언어와 유한 오토마타 — 토큰이 정규 언어인 이유, NFA→DFA, 정규식 엔진의 원리
- Lexer 직접 구현 — 스캐너 루프, lookahead, 위치 추적(line/col)으로 에러 메시지 만들기
- 까다로운 케이스 — 숫자 리터럴·문자열 이스케이프·키워드 vs 식별자·최장 일치
- 정규식 엔진의 내부 — 백트래킹 vs Thompson NFA, ReDoS가 생기는 원리

### Chapter 3: 구문 분석(Parser)과 AST (6개 문서)
- 문맥 자유 문법(CFG) — 왜 토큰화로는 부족한가, 문법 규칙과 파스 트리
- 재귀 하강 파서 — 직접 구현, 토큰 소비, 문법 규칙을 함수로 매핑
- 연산자 우선순위 — Pratt 파싱(우선순위 등반)으로 `1 + 2 * 3` 올바르게 파싱
- AST 설계 — 노드 타입, Visitor 패턴, 트리 순회
- 파싱 전략 비교 — LL vs LR, 하향식 vs 상향식, 파서 제너레이터(ANTLR/yacc)의 동작
- 에러 복구 — 한 에러에서 멈추지 않고 여러 에러 보고, panic mode

### Chapter 4: 의미 분석과 타입 시스템 (7개 문서)
- 심볼 테이블과 스코프 — 이름 해석(name resolution), 스코프 체인, 섀도잉
- 타입 검사 기초 — 타입 규칙, 타입 환경, `e : T` 판단을 코드로 구현
- 타입 추론 — Hindley-Milner의 직관, 단일화(unification), 왜 어노테이션 없이 타입이 결정되나
- 구조적 vs 명목적 타이핑 — TypeScript(구조적) vs Java(명목적), 각각의 검사 방식
- 제네릭·다형성 — 파라메트릭 다형성, 타입 변수, 변성(공변/반공변)
- 타입 시스템이 보장하는 것과 못 하는 것 — soundness vs completeness, null 안전성, 우회로
- 의미 분석 실전 — 타입 불일치·미정의 변수·도달 불가 코드 검출 구현

### Chapter 5: 중간 표현(IR)과 최적화 (6개 문서)
- IR이 필요한 이유 — AST에서 직접 코드젠하지 않는 이유, 다단계 IR(고/중/저)
- 제어 흐름 그래프(CFG) — 베이직 블록, 분기, 루프를 그래프로
- SSA(Static Single Assignment) — 변수 단일 대입, φ 함수, 최적화가 쉬워지는 이유
- 고전 최적화 — 상수 폴딩/전파, 죽은 코드 제거, 공통 부분식 제거(CSE)
- 루프 최적화 — 불변식 외부 이동(LICM), 강도 감소, 루프 언롤링
- 최적화의 위험 — UB(미정의 동작) 기반 최적화가 만드는 함정, `-O0` vs `-O2` 차이 관찰

### Chapter 6: 코드 생성과 실행 (5개 문서)
- 코드 생성 — IR→타깃(바이트코드 또는 어셈블리), 명령어 선택
- 레지스터 할당 — 그래프 컬러링, 스필(spill), 왜 레지스터가 부족한가
- 스택 vs 레지스터 머신 — JVM 바이트코드(스택) vs Dalvik(레지스터)의 설계 차이
- 인터프리터 vs 컴파일 실행 — 트리 워킹 인터프리터, 바이트코드 VM, 디스패치 기법
- 우리 언어 실행하기 — 만든 컴파일러로 실제 프로그램 실행, end-to-end

### Chapter 7: JIT와 현대 런타임 (4개 문서)
- JIT 컴파일 — 인터프리터로 시작해 핫 코드를 컴파일, 프로파일 기반 최적화
- 티어드 컴파일 — V8(Ignition→TurboFan)·JVM(C1→C2)의 단계적 최적화, 워밍업
- 역최적화(Deoptimization) — 투기적 최적화가 틀렸을 때 되돌리기, 가정 무효화
- AOT의 귀환 — GraalVM Native Image·ART AOT, JIT 대비 트레이드오프(시작/피크/메모리)

→ 각 챕터 4~7개 문서. **총 40개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 *직접 구현한 코드*로, `💻 실전 실험`은 godbolt·toy 컴파일러 실행으로.

## 🎨 스타일 가이드

1. **만들면서 이해** — 이론 설명마다 토이 언어 컴파일러의 해당 코드 조각을 병기
2. **godbolt 병기** — "이 최적화"를 실제 컴파일러(gcc/clang -O2) 출력으로 확인
3. **실제 언어와 연결** — 각 개념이 JVM/V8/TS/Rust에서 어떻게 나타나는지 명시
4. **에러 관점** — 각 단계가 어떤 에러를 잡는지("이 에러는 파서가 아니라 타입 체커가 낸다")
5. AST/CFG/SSA는 항상 그림(트리·그래프)으로

## 🔬 검증 환경

> docker-compose 불필요. **호스트 언어 + LLVM 툴체인 + godbolt**가 검증 도구.

```dockerfile
# Dockerfile — 컴파일러 실습 환경
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y \
    clang llvm llvm-dev \
    python3 python3-pip \
    default-jdk          # JVM 바이트코드 관찰(javap)용
# 토이 컴파일러 호스트 언어는 TypeScript/Python/Rust 중 택1 (가독성↑면 TS 권장)
```

```bash
# 핵심 관찰 명령어

# 실제 컴파일러의 각 단계 들여다보기
clang -Xclang -dump-tokens demo.c          # 토큰
clang -Xclang -ast-dump -fsyntax-only demo.c   # AST
clang -O2 -S -emit-llvm demo.c -o demo.ll  # LLVM IR (SSA 형태 관찰!)
clang -O0 -S -emit-llvm demo.c -o demo0.ll # 최적화 전후 IR diff

# JVM 바이트코드 (스택 머신)
javac Demo.java && javap -c Demo

# 최적화 효과 godbolt 또는
gcc -O0 -S demo.c -o o0.s ; gcc -O2 -S demo.c -o o2.s ; diff o0.s o2.s
```

핵심 산출물: 7챕터를 따라가면 **동작하는 토이 언어 컴파일러/인터프리터**가 완성된다. 각 챕터가 그 일부를 구현.

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `toy-compiler/`(누적 구현 코드)
2. **README.md**: 🪨 Foundations 톤, "언어를 쓴다 vs 언어가 어떻게 구현되나" 포지셔닝, 토이 언어 명세, `🔗 레포 연결`
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어, 구현 코드 누적

## 📚 참고 자료

- *Crafting Interpreters* (Robert Nystrom) — https://craftinginterpreters.com/
- *Engineering a Compiler* (Cooper & Torczon)
- *Compilers: Principles, Techniques, and Tools* (Dragon Book)
- *Types and Programming Languages* (Benjamin Pierce)
- LLVM Tutorial (Kaleidoscope) — https://llvm.org/docs/tutorial/
- Compiler Explorer — https://godbolt.org/

## 💡 핵심 분석 대상

```
컴파일러 파이프라인:

  source: "let x = 1 + 2 * 3"
    │
    ▼ Lexer
  tokens: [LET, IDENT(x), EQ, NUM(1), PLUS, NUM(2), STAR, NUM(3)]
    │
    ▼ Parser (Pratt: * 가 + 보다 우선)
  AST:        Let(x, Add(1, Mul(2, 3)))
    │
    ▼ TypeChecker (1,2,3 : Int → x : Int)
  typed AST
    │
    ▼ IR (SSA)
    %1 = mul 2, 3
    %2 = add 1, %1
    x  = %2
    │
    ▼ Optimizer (상수 폴딩: 전부 컴파일 타임 계산)
    x = 7          ← 1+2*3 이 7로 접힘
    │
    ▼ Codegen
    mov x, 7

스택 머신 vs 레지스터 머신 (1 + 2):
  JVM(스택):   iconst_1; iconst_2; iadd        ← 피연산자 스택
  Dalvik(레지스터): add-int v0, v1, v2          ← 레지스터 직접

JIT 티어:
  V8:  Ignition(바이트코드 인터프리터) → 핫? → Sparkplug → Maglev → TurboFan
  JVM: 인터프리터 → C1(빠른 컴파일) → C2(공격적 최적화)
  투기 최적화 틀림 → Deopt → 인터프리터로 복귀
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목(4~7개) + 핵심 내용(3~4줄) + 총 40개 확인 + 토이 언어 명세 + LLVM/godbolt 검증 환경 + JVM/V8/TS 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
