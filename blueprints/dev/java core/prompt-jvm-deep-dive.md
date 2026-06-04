# JVM Deep Dive 레포지토리 제작 프롬프트

나는 "JVM Deep Dive" 레포지토리를 만들려고 해.
**JVM을 블랙박스가 아닌, 완전히 해부된 기계로 이해하기** 위해 바이트코드부터 GC 알고리즘, JIT 컴파일러, Java Memory Model까지 끝까지 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "Java 코드를 작성하는 것과, Java 코드가 어떻게 살아 움직이는지 아는 것은 다르다"

**핵심 차별화**:
1. "무엇인가"가 아닌 "**왜 이렇게 설계됐는가**" 라는 질문 중심
2. 모든 개념은 실행 가능한 코드 + `javap` / JFR / JMH 실측으로 증명
3. CPU 캐시·메모리 배리어 같은 하드웨어 레벨까지 내려가기
4. HotSpot OpenJDK 소스 / 핵심 클래스 추적
5. 블랙박스로 통하는 통념을 모두 분해 — "GC는 메모리를 수거합니다" 같은 한 줄을 알고리즘 단계까지

**타겟 독자**:
- Java를 능숙하게 쓰지만 JVM이 어떻게 작동하는지는 모호한 시니어/주니어
- GC 튜닝, OOM 분석, 락 경합 디버깅을 해야 하는 백엔드 개발자
- "volatile은 가시성을 보장한다"가 정확히 무슨 의미인지 답하고 싶은 면접 준비생
- JIT 컴파일러가 만든 코드를 직접 보고 싶은 성능 엔지니어
- ZGC / Shenandoah 같은 최신 GC가 어떻게 1ms 미만 pause를 달성하는지 궁금한 개발자
- `synchronized`와 `ReentrantLock` 내부 차이를 코드로 증명하고 싶은 개발자

**기존 레포와의 관계**:
- **Effective Java**: 모범 사례의 "왜 그래야 하는가"를 JVM 레벨에서 증명
- **Modern Java in Action**: Lambda, CompletableFuture가 JVM 위에서 어떻게 구현되는가
- **Java Concurrency**: AQS, ThreadLocal, Virtual Thread가 JVM 메커니즘과 연결되는 지점

---

## 🎯 1단계: 전체 구조 (이미 확정 — 변경 금지)

이 레포는 **9개 섹션, 총 69개 문서**로 구성된다. 챕터·문서 제목·핵심 내용은 아래에 고정되어 있으며 추가/삭제/순서 변경 없이 이대로 작성한다.

### 🔹 Section 1: Class Loading (7개 문서)
> **핵심 질문:** `new MyClass()` 를 호출하기 전, JVM은 무엇을 하는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. ClassLoader Hierarchy](./class-loading/01-classloader-hierarchy.md) | Bootstrap / Extension / Application 계층과 Parent Delegation Model |
| [02. Loading → Linking → Initializing](./class-loading/02-loading-linking-initializing.md) | 3단계 책임 분리, static 초기화 블록이 실행되는 정확한 시점 |
| [03. Bytecode Verification](./class-loading/03-bytecode-verification.md) | JVM이 .class 파일을 어떻게 신뢰하는가, Verifier 동작 원리 |
| [04. Symbolic Reference Resolution](./class-loading/04-symbolic-reference-resolution.md) | ConstantPool의 심볼릭 참조가 직접 참조로 변환되는 과정 |
| [05. Class Unloading](./class-loading/05-class-unloading.md) | 클래스가 언로딩되는 조건, ClassLoader 누수와 메모리 누수 |
| [06. Custom ClassLoader](./class-loading/06-custom-classloader.md) | `findClass()` vs `loadClass()`, 암호화된 클래스 런타임 복호화 |
| [07. ClassLoader Isolation](./class-loading/07-classloader-isolation.md) | 같은 클래스명이 두 ClassLoader에서 로드되면 `==` 결과는? |

### 🔹 Section 2: Runtime Data Areas (7개 문서)
> **핵심 질문:** 내 객체는 JVM 메모리 어디에, 어떤 모습으로 존재하는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. Heap Structure](./runtime-data-areas/01-heap-structure.md) | Eden / Survivor / Old Generation 물리적 구조, 객체 이동 조건 |
| [02. TLAB (Thread-Local Allocation Buffer)](./runtime-data-areas/02-tlab-thread-local-allocation.md) | TLAB가 없으면 생기는 경합, 스레드별 Eden 파티셔닝 원리 |
| [03. Stack And Frames](./runtime-data-areas/03-stack-and-frames.md) | Stack Frame 구조(LVA / Operand Stack / Frame Data), StackOverflowError 시점 |
| [04. Method Area & Metaspace](./runtime-data-areas/04-method-area-metaspace.md) | PermGen이 사라진 이유, Metaspace OOM 시나리오 |
| [05. Runtime Constant Pool](./runtime-data-areas/05-runtime-constant-pool.md) | 클래스 파일 상수풀 vs 런타임 상수풀, 문자열 리터럴의 위치 |
| [06. Object Layout In Memory](./runtime-data-areas/06-object-layout-in-memory.md) | Object Header + Instance Data + Padding, JOL로 실측 |
| [07. Off-Heap & Direct Memory](./runtime-data-areas/07-off-heap-direct-memory.md) | ByteBuffer, `sun.misc.Unsafe`, GC가 닿지 않는 메모리 |

### 🔹 Section 3: Bytecode (7개 문서)
> **핵심 질문:** 내가 짠 Java 코드가 JVM의 언어로 어떻게 번역되는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. Class File Format](./bytecode/01-class-file-format.md) | `.class` 파일 바이너리 구조, magic number부터 attributes까지 |
| [02. Bytecode Instruction Set](./bytecode/02-bytecode-instruction-set.md) | 200+ 명령어 카테고리 분류, 타입별 명령어 분리 이유 |
| [03. Operand Stack Mechanism](./bytecode/03-operand-stack-mechanism.md) | 스택 기반 VM vs 레지스터 기반 VM, 명령어가 스택에 하는 일 |
| [04. Method Invocation Instructions](./bytecode/04-method-invocation-instructions.md) | `invokevirtual` / `invokeinterface` / `invokespecial` / `invokestatic` 차이 |
| [05. Exception Handling Bytecode](./bytecode/05-exception-handling-bytecode.md) | try-catch-finally가 bytecode에서 Exception Table로 변환되는 방식 |
| [06. Lambda & InvokeDynamic](./bytecode/06-lambda-and-invokedynamic.md) | Lambda가 내부 클래스가 아닌 이유, `LambdaMetafactory` 동작 원리 |
| [07. Bytecode Manipulation (ASM)](./bytecode/07-bytecode-manipulation-asm.md) | ASM으로 런타임에 바이트코드 조작, AOP 구현 원리 |

### 🔹 Section 4: Execution Engine (7개 문서)
> **핵심 질문:** JVM은 bytecode를 어떻게 "빠르게" 실행하는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. Interpreter Mechanism](./execution-engine/01-interpreter-mechanism.md) | Template Interpreter 구조, bytecode → 기계어 디스패치 테이블 |
| [02. JIT Compilation Basics](./execution-engine/02-jit-compilation-basics.md) | Warm-up 임계값, 컴파일 대상 선정 기준 (`-XX:+PrintCompilation`) |
| [03. Tiered Compilation](./execution-engine/03-tiered-compilation.md) | Level 0~4 전환 조건, C1 / C2 컴파일러 역할 분리 |
| [04. JIT Optimizations](./execution-engine/04-jit-optimizations.md) | Inlining, Escape Analysis, Loop Unrolling, Dead Code Elimination |
| [05. On-Stack Replacement (OSR)](./execution-engine/05-on-stack-replacement.md) | 이미 실행 중인 메서드를 JIT 버전으로 교체하는 메커니즘 |
| [06. Deoptimization](./execution-engine/06-deoptimization.md) | Speculative Optimization 실패 시 Interpreter로 복귀하는 과정 |
| [07. JVM Intrinsics](./execution-engine/07-intrinsics.md) | JVM이 특정 메서드를 CPU 명령어로 직접 대체하는 방식 |

### 🔹 Section 5: Garbage Collection (11개 문서)
> **핵심 질문:** JVM은 어떻게 "죽은 객체"를 판단하고, 어떻게 제거하는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. GC Roots & Reachability](./garbage-collection/01-gc-roots-and-reachability.md) | GC Root 종류, 순환 참조가 왜 문제가 안 되는가 |
| [02. Reference Types](./garbage-collection/02-reference-types.md) | Strong / Soft / Weak / Phantom Reference별 GC 동작, `WeakHashMap` |
| [03. Mark-Sweep-Compact](./garbage-collection/03-mark-sweep-compact.md) | 3단계 알고리즘, Fragmentation 문제와 Compaction 비용 |
| [04. Generational Hypothesis](./garbage-collection/04-generational-hypothesis.md) | "대부분의 객체는 젊어서 죽는다"는 가설이 GC 설계에 미친 영향 |
| [05. Serial & Parallel GC](./garbage-collection/05-serial-parallel-gc.md) | 단순 GC 동작 원리, Stop-The-World 비용 |
| [06. CMS GC & Problems](./garbage-collection/06-cms-gc-and-problems.md) | Concurrent Mark의 혁신과 Concurrent Mode Failure 한계, G1 탄생 배경 |
| [07. G1 GC Deep Dive](./garbage-collection/07-g1-gc-deep-dive.md) | Region 기반 구조, Concurrent Marking → Evacuation, Pause Prediction Model |
| [08. ZGC Deep Dive](./garbage-collection/08-zgc-deep-dive.md) | Colored Pointer, Load Barrier, Concurrent Relocation — pause < 1ms 원리 |
| [09. Shenandoah GC](./garbage-collection/09-shenandoah-gc.md) | Brooks Pointer, ZGC와의 설계 철학 차이 |
| [10. GC Tuning Flags](./garbage-collection/10-gc-tuning-flags.md) | 실전에서 쓰는 JVM 플래그 완전 정리 |
| [11. GC Log Analysis](./garbage-collection/11-gc-log-analysis.md) | `-Xlog:gc*` 로그 해석, STW 시간 측정, 메모리 누수 탐지 |

### 🔹 Section 6: Java Memory Model (7개 문서)
> **핵심 질문:** 멀티코어 CPU에서 Java 코드는 왜 예상과 다르게 동작하는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. CPU Cache & Visibility Problem](./java-memory-model/01-cpu-cache-and-visibility-problem.md) | 캐시 계층 구조, 명령어 재정렬, JMM이 이 모든 것을 추상화하는 이유 |
| [02. Happens-Before](./java-memory-model/02-happens-before.md) | HB 규칙 8가지, "실행 순서"와 "가시성 보장 순서"가 다른 이유 |
| [03. Volatile Deep Dive](./java-memory-model/03-volatile-deep-dive.md) | volatile이 보장하는 것(가시성 + 재정렬 금지)과 보장 안 하는 것(원자성) |
| [04. Final Field Semantics](./java-memory-model/04-final-field-semantics.md) | 생성자 완료 후 final 필드가 보장되는 범위, 안전한 불변 객체 공개 |
| [05. Publication & Escape](./java-memory-model/05-publication-and-escape.md) | 객체가 "탈출"하는 경우, 안전한 공개(Safe Publication) 패턴 |
| [06. Synchronized Internals](./java-memory-model/06-synchronized-internals.md) | synchronized가 삽입하는 Memory Barrier, 모니터 락의 메모리 의미론 |
| [07. Memory Barriers](./java-memory-model/07-memory-barriers.md) | LoadLoad / StoreStore / LoadStore / StoreLoad 배리어와 CPU 명령어 |

### 🔹 Section 7: Concurrency Internals (9개 문서)
> **핵심 질문:** `synchronized`와 `ReentrantLock`은 내부에서 어떻게 다른가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. Object Monitor](./concurrency-internals/01-object-monitor.md) | Monitor 구조, Entry Set / Wait Set, `wait()` / `notify()` 내부 동작 |
| [02. Lock: Biased → Thin → Fat](./concurrency-internals/02-lock-biased-thin-fat.md) | Mark Word 변화로 보는 Lock 상태 전이, Biased Lock deprecated 이유 |
| [03. CAS & Atomic Operations](./concurrency-internals/03-cas-and-atomic-operations.md) | CPU의 `CMPXCHG` 명령어, ABA 문제, AtomicInteger 내부 구현 |
| [04. False Sharing & Cache Line](./concurrency-internals/04-false-sharing-and-cache-line.md) | 64바이트 캐시라인, `@Contended`, JMH로 False Sharing 실측 |
| [05. AQS Internals](./concurrency-internals/05-aqs-internals.md) | CLH Queue, `ReentrantLock` / `Semaphore` / `CountDownLatch` 공통 기반 |
| [06. Thread States & Scheduler](./concurrency-internals/06-thread-states-and-scheduler.md) | OS 스레드 상태 vs JVM 스레드 상태, Context Switching 비용 |
| [07. ThreadLocal Internals](./concurrency-internals/07-thread-local-internals.md) | `ThreadLocalMap` 내부 구조, 메모리 누수 발생 조건 |
| [08. Virtual Threads (Project Loom)](./concurrency-internals/08-virtual-threads-loom.md) | Carrier Thread, Structured Concurrency, pinning 주의사항 |
| [09. Safepoint Mechanism](./concurrency-internals/09-safepoint-mechanism.md) | Safepoint가 필요한 이유, Time-To-Safepoint 지연 원인과 분석 |

### 🔹 Section 8: Performance Tuning (7개 문서)
> **핵심 질문:** JVM을 어떻게 측정하고, 어떻게 최적화하는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. JVM Flags Complete Guide](./performance-tuning/01-jvm-flags-complete-guide.md) | 실전에서 쓰는 플래그 전체 정리 (`-Xms`, `-Xmx`, `-XX:*`) |
| [02. Heap Sizing Strategy](./performance-tuning/02-heap-sizing-strategy.md) | Initial / Max Heap 비율, Young/Old 비율, 컨테이너 환경 주의사항 |
| [03. GC Ergonomics](./performance-tuning/03-gc-ergonomics.md) | JVM이 스스로 GC와 힙 크기를 조정하는 자동 튜닝 원리 |
| [04. Profiling with JFR](./performance-tuning/04-profiling-with-jfr.md) | Java Flight Recorder + JDK Mission Control, Flame Graph 읽기 |
| [05. Profiling with async-profiler](./performance-tuning/05-profiling-with-async-profiler.md) | CPU / 메모리 / 락 프로파일링, `alloc` 모드로 GC 압박 찾기 |
| [06. Memory Leak Analysis](./performance-tuning/06-memory-leak-analysis.md) | Heap Dump 분석, 누수 패턴 (static, ThreadLocal, ClassLoader) |
| [07. Benchmarking with JMH](./performance-tuning/07-benchmarking-with-jmh.md) | 왜 `System.nanoTime()`은 부정확한가, Warm-up / Blackhole / @State |

### 🔹 Section 9: Advanced Internals (7개 문서)
> **핵심 질문:** JVM이 숨기고 있는 더 깊은 층에는 무엇이 있는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. Object Header & Mark Word](./advanced-internals/01-object-header-and-mark-word.md) | 64비트 Mark Word 레이아웃, 해시코드 / Lock 상태 / GC 나이 필드 |
| [02. Compressed Oops](./advanced-internals/02-compressed-oops.md) | 64비트 JVM에서 포인터를 32비트로 압축하는 원리, 32GB 힙 제한 이유 |
| [03. String Pool & Interning](./advanced-internals/03-string-pool-interning.md) | 문자열 상수풀 위치 변화 (PermGen → Heap), `intern()` 비용 |
| [04. Unsafe API](./advanced-internals/04-unsafe-api.md) | `sun.misc.Unsafe`로 직접 메모리 조작, JDK 내부 코드가 쓰는 이유 |
| [05. Reflection & Performance](./advanced-internals/05-reflection-and-performance.md) | Reflection 호출 경로, 15회 임계값 후 바이트코드 생성, JIT와의 관계 |
| [06. Instrumentation & Java Agent](./advanced-internals/06-instrumentation-and-agent.md) | `-javaagent` 동작 원리, `ClassFileTransformer`로 클래스 변환 |
| [07. JNI Internals](./advanced-internals/07-jni-internals.md) | JVM ↔ Native 코드 경계, JNI 호출 비용, Global / Local Reference |

**총 문서 수: 7 + 7 + 7 + 7 + 11 + 7 + 9 + 7 + 7 = 69개 (확정)**

---

## 📐 문서 구조 (모든 문서 동일 적용)

```markdown
## 🎯 핵심 질문
## 🔍 왜 이 메커니즘이 필요한가 (없으면 어떤 문제가 생기는가)
## 😱 흔한 오해 또는 잘못된 사용 (Before)
## ✨ 올바른 이해와 사용 (After)
## 🔬 내부 동작 원리 (HotSpot 소스 / 공식 스펙 추적)
## 💻 실험으로 확인하기 (실행 가능한 코드 + 도구 출력 분석)
## 📊 측정 결과 (JMH / JFR / GC Log 실측)
## 🤔 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

---

## 🎨 스타일 가이드

1. **모든 주장은 도구 출력으로 증명** — 글로 설명하지 말고 `javap`, `JFR`, `JMH`, `-XX:+PrintCompilation`, `jhsdb` 등으로 보여주기
2. **HotSpot OpenJDK 21을 기준** 으로 통일 (필요시 다른 버전 비교 명시)
3. **CPU 명령어 레벨까지 내려가는 절** 은 x86-64 기준 (ARM64는 차이 있는 부분만 별도 명시)
4. **"이렇게 동작한다" → "왜 이렇게 설계됐는가"** 흐름 — 메커니즘 설명 후 설계 의도 분석
5. **GC 로그·Flame Graph는 직접 캡처해서 첨부** (스크린샷 또는 텍스트 출력)
6. **각 문서는 독립적으로 읽을 수 있어야 함** — 다른 문서의 사전 지식 필요 시 명시적 링크

---

## 🛠️ 실험 도구

- `javap -c -v -p` (바이트코드 분석)
- `-XX:+PrintCompilation`, `-XX:+PrintInlining` (JIT 분석)
- JOL (Java Object Layout) — 객체 메모리 레이아웃 측정
- JFR (Java Flight Recorder) + JDK Mission Control
- async-profiler (CPU / Alloc / Lock 프로파일링)
- JMH (Java Microbenchmark Harness)
- jhsdb / hsdis (HotSpot 디스어셈블러)
- Heap Dump 분석 도구 (Eclipse MAT)
- `-Xlog:gc*` GC 로그 분석

---

## 🎯 2단계: 작업 순서

전체 구조는 위에 이미 확정되어 있으므로:

1. **디렉토리 생성**: bash 명령어로 9개 섹션 폴더 생성
   ```
   class-loading/
   runtime-data-areas/
   bytecode/
   execution-engine/
   garbage-collection/
   java-memory-model/
   concurrency-internals/
   performance-tuning/
   advanced-internals/
   ```
2. **README.md 작성**:
   - 빠른 시작 배지 9개 (각 섹션의 첫 문서)
   - 9개 섹션 details 토글 + 표 형식 문서 목록
   - "일반 자료 vs 이 레포" 비교 표
   - 학습 경로 (입문 / JVM 튜닝 실무자 / 면접 준비 / GC 마스터 / 성능 엔지니어)
3. **섹션별 문서 작성**:
   - Section 1부터 순서대로
   - 한 섹션 완성 후 다음으로
   - 각 문서는 3000~4500 단어 분량 (실측·코드 분량 포함)
   - **모든 문서는 위 10섹션 구조를 준수**

---

## 📚 참고 자료

- **Java Virtual Machine Specification (JVMS)** — Java SE 21 Edition (Oracle 공식)
- **Java Language Specification (JLS)** — Java SE 21 Edition
- **JEP** (JDK Enhancement Proposals) — 각 GC, JIT 변경 내역
- **OpenJDK HotSpot 소스코드** — github.com/openjdk/jdk
- **Aleksey Shipilev의 JVM 블로그** — shipilev.net
- **The Garbage Collection Handbook** (Richard Jones)
- **Java Performance: The Definitive Guide** (Scott Oaks)

---

## 💡 핵심 분석 대상

### Lambda → invokedynamic (Section 3)

```java
Function<Integer, Integer> f = x -> x * 2;
```

```
$ javap -c -v Main.class
0: invokedynamic #2,  0
BootstrapMethods:
  0: REF_invokeStatic LambdaMetafactory.metafactory(...)
    Method arguments:
      (Ljava/lang/Object;)Ljava/lang/Object;
      REF_invokeStatic Main.lambda$0:(Ljava/lang/Integer;)Ljava/lang/Integer;
      ...
```

→ Inner class 방식이 아닌 invokedynamic을 쓰는 이유, LambdaMetafactory가 런타임에 `Function` 구현체를 생성하는 과정

### G1 GC Region (Section 5)

```
$ jcmd <pid> GC.heap_info
 garbage-first heap   total 4194304K, used ...
  region size 2048K, 256 young (524288K), 16 survivors (32768K)
```

→ 고정 세대 분할이 아닌 Region 단위로 관리되는 이유, Pause Prediction Model이 실제로 어떻게 다음 GC를 결정하는지

### volatile vs synchronized 어셈블리 (Section 6)

```java
volatile int x = 0;
x = 42;  // 어떤 어셈블리가 생성되는가?
```

→ x86에서 `lock addl $0x0,(%rsp)` 같은 mfence 효과 명령이 어떻게 삽입되는지 hsdis로 직접 확인

---

## 🎯 진행 방식

**구조는 이미 확정되어 있으므로 1단계는 생략하고 바로 2단계(디렉토리 생성)부터 시작.**

진행 순서:
1. 디렉토리 9개 생성
2. README.md 작성 (빠른 시작 배지 + 학습 경로)
3. Section 1부터 순서대로 7 → 7 → 7 → 7 → 11 → 7 → 9 → 7 → 7 = 69개 문서 작성
4. 각 문서마다 10섹션 구조 강제 준수
5. 모든 주장은 도구 출력 증거 첨부

**시작해줘! 디렉토리 생성 후 README, 그다음 Section 1의 1번 문서부터.**
