# Modern Java in Action 레포지토리 제작 프롬프트

나는 "Modern Java in Action" 레포지토리를 만들려고 해.
JVM Deep Dive, Effective Java, Object 책 정리를 완성한 경험을 바탕으로, **자바 8 이후 도입된 모던 자바 기능들이 바이트코드 레벨에서 어떻게 동작하는지** 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "자바 8부터 21까지, 함수형 패러다임이 자바를 어떻게 바꿨는가"

**핵심 차별화**:
1. "어떤 기능이 추가됐나"가 아닌 "어떻게 동작하나" (원리 중심)
2. 바이트코드 레벨 분석 (`javap -c -v`로 invokedynamic 분해)
3. JDK 표준 라이브러리 소스코드 직접 추적 (`Stream`, `Optional`, `CompletableFuture` 내부)
4. Java 버전별 진화 추적 (8 → 11 → 17 → 21)
5. Before/After 비교 (전통적 코드 vs 함수형 코드의 바이트코드 차이)

**타겟 독자**:
- Lambda를 매일 쓰지만 invokedynamic이 뭔지 모르는 개발자
- Stream을 쓰는데 왜 lazy한지 설명 못하는 개발자
- CompletableFuture의 thenApply/thenCompose 차이를 헷갈리는 개발자
- "왜 effectively final이어야 하지?" 같은 질문에 정확히 답하고 싶은 개발자
- Virtual Thread, Record, Pattern Matching 등 최신 기능을 깊이 이해하고 싶은 개발자

**기존 레포와의 관계**:
- **JVM Deep Dive**: invokedynamic, ForkJoinPool, Continuation 내부 동작 연결
- **Effective Java**: 모범 사례의 "왜 그래야 하는가"를 바이트코드로 증명
- **Object**: 함수형 패러다임이 OOP 설계를 어떻게 보완하는가
- **Java Concurrency**: CompletableFuture, Virtual Thread와 동시성 모델 연결

**책과의 관계**:
- "Modern Java in Action" 책의 주제는 모두 커버하되, 책보다 훨씬 깊게 들어감
- 책에서 다루지 않는 Java 14+ 기능 (Record, Sealed Class, Pattern Matching, Virtual Thread)도 포함
- 책의 챕터 순서는 따르지 않음 — 주제별로 재구성

---

## 🎯 1단계: 전체 구조 설계

다음 10개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Lambda Expression Internals (6개 문서)
- Lambda는 어떻게 바이트코드로 변환되는가 (`invokedynamic` + `LambdaMetafactory`)
- @FunctionalInterface와 SAM(Single Abstract Method) 변환 메커니즘
- Method Reference 4가지 종류 (`Class::staticMethod`, `instance::method`, `Class::instanceMethod`, `Class::new`)와 각각의 바이트코드
- Closure와 Variable Capture: 왜 effectively final이어야 하는가 (스택/힙 분석)
- Lambda vs Anonymous Inner Class: 메모리, 성능, 바이트코드 비교
- Functional Interface 5가지 카테고리 (Function, Consumer, Supplier, Predicate, Operator)와 박싱 회피 변형

### Chapter 2: Stream API 내부 동작 (8개 문서)
- Stream Pipeline 3단계 구조 (Source → Intermediate → Terminal)
- ReferencePipeline 소스코드 추적 (Head, StatelessOp, StatefulOp)
- Lazy Evaluation은 어떻게 구현되는가
- Spliterator: 분할 가능 컬렉션 인터페이스
- Short-circuit Operations 동작 원리 (`findFirst`, `anyMatch`, `limit`)
- Stateless vs Stateful 연산의 본질적 차이 (`map` vs `sorted`/`distinct`)
- Collector 내부 (Mutable Reduction, characteristics, combiner)
- Custom Collector 작성과 `Collector.of()` 분석

### Chapter 3: 병렬 스트림과 Fork/Join (5개 문서)
- ForkJoinPool 동작 원리 (Work-Stealing 알고리즘)
- Common Pool 의존성과 위험성 (왜 `ForkJoinPool.commonPool()`을 피해야 하나)
- Spliterator 분할 전략 (`trySplit`, ARRAY/SUBSIZED characteristics)
- 병렬 스트림 성능 함정 (박싱, autoboxing, 작은 데이터셋)
- 언제 병렬 스트림을 써야 하는가 (NQ Model, 데이터 크기 임계값)

### Chapter 4: Optional 패턴 (5개 문서)
- Optional 내부 구조와 설계 의도 (왜 final class인가)
- `Optional.of` vs `ofNullable` vs `empty`: 정확한 사용 시점
- `map` vs `flatMap`: Functor와 Monad 패턴
- Optional 안티패턴 (필드, 매개변수, Collection 감싸기 금지 이유)
- 직렬화 문제와 ORM 엔티티에서의 Optional 처리

### Chapter 5: CompletableFuture 비동기 프로그래밍 (7개 문서)
- CompletableFuture 구조와 Future와의 차이 (Callback Hell 해결)
- `thenApply` vs `thenCompose` vs `thenCombine`: Function vs Future 차이
- `thenRun` vs `thenAccept` vs `thenApply`: 결과 사용 패턴
- Async 변형 (`thenApplyAsync`)과 Executor 지정 전략
- 예외 처리 3가지 (`exceptionally`, `handle`, `whenComplete`) 차이
- `allOf`, `anyOf` 구현 분석과 실전 패턴
- ForkJoinPool.commonPool() 의존성 문제와 해결

### Chapter 6: 인터페이스 진화 (5개 문서)
- Default Method 동작 원리 (`invokespecial` vs `invokevirtual`)
- 다이아몬드 상속 해결 규칙 (Class > Interface, Subinterface > Superinterface)
- Private Interface Method (Java 9): 코드 중복 제거
- Static Interface Method와 Helper Method 패턴
- Sealed Interface (Java 17): 상속 계층 봉인과 Pattern Matching 연계

### Chapter 7: 새로운 Date/Time API (4개 문서)
- `LocalDate`, `LocalDateTime`, `ZonedDateTime`, `Instant` 차이와 사용 시점
- 불변성 설계: 왜 `withYear()` 같은 메서드만 있는가
- TemporalAdjuster, TemporalQuery 활용 패턴
- Old `Date`/`Calendar` → New API 마이그레이션 전략과 함정

### Chapter 8: Pattern Matching & Records (6개 문서)
- Record 구조 (Java 16): 자동 생성 메서드와 바이트코드
- Record vs Lombok @Data: 어떤 차이가 있는가
- Pattern Matching for `instanceof` (Java 16): 타입 캐스팅 제거
- Switch Expression (Java 14): yield와 화살표 문법
- Sealed Classes (Java 17): exhaustive pattern matching 연계
- Record Pattern (Java 21): 구조 분해 패턴

### Chapter 9: Virtual Threads & Project Loom (5개 문서)
- Platform Thread vs Virtual Thread: 1:1 vs M:N 모델
- Continuation 메커니즘: stack을 어떻게 yield/resume하는가
- Pinning 문제 (synchronized, native call) 진단과 해결
- Structured Concurrency (Java 21 Preview)
- 기존 Thread Pool 기반 코드 → Virtual Thread 전환 전략

### Chapter 10: 함수형 프로그래밍 패턴 (5개 문서)
- 고차 함수 (Higher-Order Function): 자바에서의 적용
- 커링 (Currying)과 부분 적용 (Partial Application)
- 메모이제이션 패턴: ConcurrentHashMap.computeIfAbsent 활용
- 영속 자료구조 (Persistent Data Structure)와 함수형 컬렉션
- Either/Result 패턴 (Vavr 라이브러리 vs 직접 구현)

---

각 챕터는 **4~8개 문서**로 구성해줘. 총 **56개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 기능이 도입되었는가 (Java 진화 맥락)
## 😱 흔한 오해 또는 잘못된 사용 (Before)
## ✨ 올바른 이해와 사용 (After)
## 🔬 내부 동작 원리 (JDK 소스코드 + 바이트코드 분석)
## 💻 실험으로 확인하기 (실행 가능한 코드 + javap 분석)
## 📊 성능 비교 (JMH 벤치마크 또는 마이크로벤치마크)
## 🤔 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. JDK 표준 라이브러리 소스코드 직접 추적 (`Stream`, `Optional`, `CompletableFuture`, `LambdaMetafactory`)
2. 바이트코드 레벨 분석 (`javap -c -v` 활용, `invokedynamic` 분해)
3. Java 버전별 진화 추적 (해당 기능이 어느 버전에 도입됐고 이후 어떻게 발전했는가)
4. "전통적 방식 vs 모던 방식" 코드/바이트코드 비교
5. JMH 벤치마크 또는 간단한 측정 코드로 성능 검증
6. 실무 안티패턴 분석 (Optional 필드, parallelStream 남용, thenApply 오남용 등)

**실험 도구**:
- `javap -c -v -p` (바이트코드 + LambdaMetafactory 인자 확인)
- JMH (Java Microbenchmark Harness)
- JFR (Java Flight Recorder) for Virtual Thread 분석
- Async-profiler for ForkJoinPool 분석
- IntelliJ Debugger (Stream Trace 활용)

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성 (`chapter01-lambda`, `chapter02-stream`, ...)
2. **README.md 작성**: 
   - 기존 Modern Java in Action README의 디자인 톤 유지 (배지, 이모지, 학습 경로)
   - 책 챕터 매핑이 아닌 주제별 재구성으로 변경
   - Dev Book Lab 시리즈와의 연결 명시
   - 학습 경로 (입문 4주 / 중급 8주 / 고급 3개월)
3. **챕터별 문서 작성**: 
   - Chapter 1부터 순서대로
   - 한 챕터씩 완성 후 다음으로
   - 각 문서는 2500~3500 단어 분량

---

## 📚 참고 자료

<JVM Deep Dive README를 여기에 붙여넣기>

<Effective Java README를 여기에 붙여넣기>

<기존 Modern Java in Action README를 여기에 붙여넣기>

위 README들을 참고해서 비슷한 구조와 깊이로 새 버전을 만들어줘.

**기존 Modern Java in Action 레포와의 차이**:
- 책 챕터 그대로 따라가기 ❌ → 주제별 재구성 ✅
- 학습 노트 ❌ → JDK 소스코드 + 바이트코드 추적 ✅
- 책 요약 ❌ → "왜 이렇게 설계됐는가" 깊이 분석 ✅
- Java 8 위주 ❌ → Java 8 ~ 21 전체 진화 ✅

---

## 💡 핵심 분석 대상

### Lambda → invokedynamic 변환

```java
// 소스 코드
Function<Integer, Integer> doubler = x -> x * 2;

// javap -c -v 결과 (대략)
0: invokedynamic #2,  0   // InvokeDynamic #0:apply:()Ljava/util/function/Function;

BootstrapMethods:
  0: #28 REF_invokeStatic java/lang/invoke/LambdaMetafactory.metafactory(...)
    Method arguments:
      #29 (Ljava/lang/Object;)Ljava/lang/Object;
      #30 REF_invokeStatic Main.lambda$0:(Ljava/lang/Integer;)Ljava/lang/Integer;
      #31 (Ljava/lang/Integer;)Ljava/lang/Integer;
```

→ `LambdaMetafactory`가 런타임에 어떻게 `Function` 구현체를 생성하는가 분해

### Stream Pipeline 구조

```java
// ReferencePipeline 계층
abstract class AbstractPipeline<...> {
    AbstractPipeline previousStage;
    AbstractPipeline sourceStage;
    int sourceOrOpFlags;
    // ...
}

class Head extends ReferencePipeline { ... }
class StatelessOp extends ReferencePipeline { ... }
class StatefulOp extends ReferencePipeline { ... }
```

→ 중간 연산이 즉시 실행되지 않고 어떻게 연결만 되는지, terminal 연산에서 어떻게 평가되는지 추적

### CompletableFuture 내부 구조

```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
    volatile Object result;       // null=incomplete, T=success, AltResult=exception
    volatile Completion stack;    // Treiber stack of dependent actions
    // ...
}
```

→ Lock-free 자료구조로 어떻게 비동기 콜백 체인을 구현하는가

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~8개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (56개 목표)
- 주요 분석 대상 (`LambdaMetafactory`, `ReferencePipeline`, `CompletableFuture` 소스 등)
- 각 챕터에서 다룰 Java 버전 명시 (Java 8/9/11/14/16/17/21)

설계가 완료되면 내가 검토하고, 승인하면 2단계(디렉토리 생성)로 넘어가자.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
