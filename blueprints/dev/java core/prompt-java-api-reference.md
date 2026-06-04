# Java API Reference 레포지토리 제작 프롬프트

나는 "Java API Reference" 레포지토리를 만들려고 해.
**암기가 아닌 이해, 요약이 아닌 통찰** 을 목표로, Java 표준 라이브러리를 실무 개발부터 코딩 테스트까지 활용할 수 있도록 정리하는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "암기가 아닌 이해, 요약이 아닌 통찰"

**핵심 차별화**:
1. 단순 메서드 나열 ❌ → **실전에서 바로 써먹을 수 있는** 패턴 중심
2. 모든 코드는 **복사-붙여넣기 즉시 실행 가능** (200+ 예제)
3. 같은 기능의 **여러 방법 비교** + 성능 벤치마크 (예: `String.format` vs `+` vs `StringBuilder`)
4. **함정 회피 가이드** — 초보자가 자주 하는 실수를 미리 차단
5. **25+ 실전 연습 문제** — 학습 내용 즉시 확인

**타겟 독자**:
- Java 표준 라이브러리를 능숙하게 쓰고 싶은 주니어
- 코딩 테스트에서 ArrayList vs LinkedList 선택 기준이 헷갈리는 개발자
- HashMap 내부 구조를 이해하고 면접에 답하고 싶은 사람
- Stream API를 매일 쓰지만 lazy evaluation이 뭔지 모르는 개발자
- 동시성 컬렉션 / Atomic / CompletableFuture를 정리하고 싶은 개발자
- Reflection · Annotation 활용도를 높이고 싶은 개발자

**기존 레포와의 관계**:
- **JVM Deep Dive**: HashMap의 해시 충돌·resize 메커니즘 같은 깊이를 표면에서 활용
- **Modern Java in Action**: Lambda, Stream의 깊이 있는 분석은 그쪽에 양보, 이 레포는 "쓰는 법" 중심
- **Effective Java**: 모범 사례를 표준 라이브러리 활용 측면에서 재해석
- **Java Design Patterns**: Iterator, Strategy 등 패턴이 표준 API에 어떻게 녹아있는지

---

## 🎯 1단계: 전체 구조 (이미 확정 — 변경 금지)

이 레포는 **6개 카테고리(섹션), 총 69개 문서**로 구성된다. 카테고리·하위 분류·문서 제목·핵심 키워드는 아래에 고정되어 있으며 추가/삭제/순서 변경 없이 이대로 작성한다.

### 🔹 1. 기초 다지기 (Basics) — 16개

#### 🔤 String & 문자열 (7개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[01. 기본 개념](./string/String-01-기본개념.md)** | Immutable & String Pool | 불변성, intern(), 메모리 구조 |
| **[02. 생성과 비교](./string/String-02-생성과비교.md)** | 생성 방법과 비교 메서드 | equals, compareTo, valueOf |
| **[03. 검색과 인덱싱](./string/String-03-검색과인덱싱.md)** | 문자 접근과 위치 찾기 | charAt, indexOf, substring |
| **[04. 변환과 치환](./string/String-04-변환과치환.md)** | 문자열 변환하기 | toUpperCase, replace, trim |
| **[05. 분리와 결합](./string/String-05-분리와결합.md)** | 나누고 합치기 | split, join, StringJoiner |
| **[06. StringBuilder & StringBuffer](./string/String-06-StringBuilder-StringBuffer.md)** | 가변 문자열 처리 | 성능 최적화, 동기화 |
| **[07. 실전 패턴](./string/String-07-실전패턴.md)** | 알고리즘과 실무 패턴 | 팰린드롬, 검증, 파싱, 최적화 |

#### 🔢 Math & Number (3개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[01. Math 클래스](./math/Math-01-Math클래스.md)** | 기본 수학 연산과 난수 | abs, pow, sqrt, random, round |
| **[02. Wrapper 클래스](./math/Math-02-Wrapper.md)** | 기본 타입의 객체화 | Integer, valueOf, parseInt, Boxing |
| **[03. BigInteger & BigDecimal](./math/Math-03-Big.md)** | 대용량 및 정밀 연산 | BigInteger, BigDecimal, 정밀도 |

#### 📦 Arrays (배열) (6개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[01. 배열 기본](./arrays/Arrays-01-배열기본.md)** | 배열과 Arrays 클래스 | 선언, 초기화, 기본 연산 |
| **[02. 정렬](./arrays/Arrays-02-정렬.md)** | sort, parallelSort | Comparator, 성능 비교 |
| **[03. 검색](./arrays/Arrays-03-검색.md)** | binarySearch | 이진 탐색 활용 |
| **[04. 비교와 복사](./arrays/Arrays-04-비교와복사.md)** | equals, copyOf | 깊은 복사, 얕은 복사 |
| **[05. 변환](./arrays/Arrays-05-변환.md)** | stream, asList | 배열 ↔ List 변환 |
| **[06. 다차원 배열](./arrays/Arrays-06-다차원배열.md)** | 2D, 3D 배열 | deepEquals, deepToString |

### 🔹 2. 타입 안전성과 예외 처리 (Type Safety & Error Handling) — 8개

#### 🛡️ Generics (제네릭) (2개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[01. Generics Basic](./generics/Generics-01-Basic.md)** | 제네릭 클래스와 메서드 기초 | `<T>`, Bounded Type, Type Erasure |
| **[02. Wildcard & PECS](./generics/Generics-02-Wildcard.md)** | 와일드카드와 유연성 설계 | `<?>`, `extends`, `super`, PECS 공식 |

#### 🚦 Enum (열거형) (3개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[01. Enum Basic](./enum/Enum-01-Basic.md)** | 열거형 기초와 특징 | enum, values(), valueOf() |
| **[02. Enum Advanced](./enum/Enum-02-Advanced.md)** | 상수별 동작 구현과 싱글톤 | abstract method, Singleton Pattern |
| **[03. Enum Patterns](./enum/Enum-03-Patterns.md)** | 실전 활용 패턴과 최적화 | Strategy Pattern, EnumMap, EnumSet |

#### ⚠️ Exception (예외 처리) (3개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[01. Exception Basic](./exception/Exception-01-Basic.md)** | 예외 계층 구조와 처리 | try-catch, Checked vs Unchecked |
| **[02. Custom Exception](./exception/Exception-02-Custom.md)** | 사용자 정의 예외 설계 | Domain Exception, Exception Chain |
| **[03. Best Practices](./exception/Exception-03-Best.md)** | 실무 예외 처리 패턴 | Logging, Fail-Fast, Anti-Patterns |

### 🔹 3. 자료구조 (Collections Framework) — 16개

#### 📋 List Interface (4개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[01. Collections 개요](./collections/Collections-01-Overview.md)** | 프레임워크 전체 구조 | Collection, Iterable, 계층 구조 |
| **[02. ArrayList](./collections/Collections-02-ArrayList.md)** | 동적 배열 구현 | add, get, resize, 성능 |
| **[03. LinkedList](./collections/Collections-03-LinkedList.md)** | 이중 연결 리스트 | addFirst, removeLast, Node |
| **[04. List 비교와 선택](./collections/Collections-04-ListComparison.md)** | 상황별 List 선택 가이드 | 조회 vs 삽입/삭제, 성능 비교 |

#### 🎯 Set Interface (4개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[05. HashSet](./collections/Collections-05-HashSet.md)** | 중복 없는 데이터 집합 | hashCode, equals, 유일성 |
| **[06. LinkedHashSet](./collections/Collections-06-LinkedHashSet.md)** | 순서가 있는 Set | 입력 순서 보장, LRU 캐시 |
| **[07. TreeSet](./collections/Collections-07-TreeSet.md)** | 정렬된 Set | 이진 탐색 트리, 범위 검색 |
| **[08. Set 비교와 선택](./collections/Collections-08-SetComparison.md)** | 상황별 Set 선택 가이드 | 정렬 필요성, 입력 순서 |

#### 🗺️ Map Interface (4개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[09. HashMap](./collections/Collections-09-HashMap.md)** | 키-값 쌍 데이터 저장 | put, get, 해시 충돌, 버킷 |
| **[10. LinkedHashMap](./collections/Collections-10-LinkedHashMap.md)** | 순서가 있는 Map | 삽입 순서, 접근 순서 |
| **[11. TreeMap](./collections/Collections-11-TreeMap.md)** | 정렬된 Map | 키 기준 정렬, NavigableMap |
| **[12. Map 비교와 선택](./collections/Collections-12-MapComparison.md)** | 상황별 Map 선택 가이드 | 키 정렬, 순서 보장 여부 |

#### 📤 Queue & Utils (4개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[13. Queue & Deque](./collections/Collections-13-QueueDeque.md)** | 대기열 처리 자료구조 | offer, poll, peek, FIFO |
| **[14. PriorityQueue](./collections/Collections-14-PriorityQueue.md)** | 우선순위 큐 | 힙(Heap), 우선순위 정렬 |
| **[15. Stack](./collections/Collections-15-Stack.md)** | LIFO 자료구조 | push, pop, Vector 상속 문제 |
| **[16. Collections 유틸](./collections/Collections-16-CollectionsUtil.md)** | 컬렉션 보조 도구 | sort, binarySearch, synchronized |

### 🔹 4. 모던 자바와 데이터 처리 (Modern Java) — 10개

#### 🚀 Lambda & Functional Interface (3개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[01. Functional Interface](./lambda/Lambda-01-FunctionalInterface.md)** | 람다식과 표준 인터페이스 | Consumer, Supplier, Function, Predicate |
| **[02. Method Reference](./lambda/Lambda-02-MethodReference.md)** | 메서드 참조와 생성자 참조 | `Class::method`, `new::` |
| **[03. Custom Lambda](./lambda/Lambda-03-Custom.md)** | 커스텀 인터페이스와 변수 포획 | @FunctionalInterface, Variable Capture |

#### ✨ Modern Java Features (Java 9~21) (3개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[01. Record](./modern-java/ModernJava-01-Record.md)** | 불변 데이터 클래스 | record, Compact Constructor, DTO |
| **[02. Switch Expression](./modern-java/ModernJava-02-Switch.md)** | 향상된 분기 처리 | arrow syntax, yield, Pattern Matching |
| **[03. Sealed Class](./modern-java/ModernJava-03-Sealed.md)** | 상속 제한 및 계층 제어 | sealed, permits, non-sealed |

#### 🛠️ Modern Data Utilities (4개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[01. Comparator & Comparable](./utils/Util-01-Comparator.md)** | 정렬과 비교 기준 | compare, compareTo, thenComparing |
| **[02. Stream API](./utils/Util-02-Stream.md)** | 데이터 처리 파이프라인 | filter, map, collect, reduce |
| **[03. Optional](./utils/Util-03-Optional.md)** | Null 안전 처리 | ofNullable, orElse, isPresent |
| **[04. 정규표현식](./utils/Util-04-Regex.md)** | 텍스트 패턴 매칭 | Pattern, Matcher, regex |

### 🔹 5. 실무 필수 API (Practical APIs) — 9개

#### 📅 Date & Time (6개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[01. Time API 개요](./datetime/DateTime-01-개요.md)** | Java 8 Time API | LocalDate, ZonedDateTime |
| **[02. Local 클래스](./datetime/DateTime-02-Local.md)** | LocalDate, LocalTime, LocalDateTime | 날짜/시간 기본 |
| **[03. Zoned & Instant](./datetime/DateTime-03-Zoned.md)** | ZonedDateTime, Instant | 타임존, UTC |
| **[04. Period & Duration](./datetime/DateTime-04-Period.md)** | 기간 계산 | 날짜 차이, 시간 차이 |
| **[05. 포맷팅](./datetime/DateTime-05-Formatter.md)** | DateTimeFormatter | 날짜 포맷 변환 |
| **[06. 레거시 vs 신규](./datetime/DateTime-06-Legacy.md)** | Date, Calendar 비교 | 마이그레이션 가이드 |

#### 💾 IO & 입출력 (3개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[01. File 기본](./io/IO-01-File.md)** | 파일 시스템 다루기 | File, Path, Files |
| **[02. 텍스트 파일 입출력](./io/IO-02-Text.md)** | 문자 스트림 (Reader/Writer) | BufferedReader, 인코딩 |
| **[03. 바이트 스트림](./io/IO-03-Binary.md)** | 바이너리 & 객체 직렬화 | BufferedStream, Serializable |

### 🔹 6. 심화 학습 (Advanced) — 10개

#### 🪞 Reflection (리플렉션) (3개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[01. Reflection Basic](./reflection/Reflection-01-Basic.md)** | 런타임 클래스 정보 조작 | Class, Method, Field, Constructor |
| **[02. Annotation](./reflection/Reflection-02-Annotation.md)** | 메타데이터 활용 | @Target, @Retention, @Repeatable |
| **[03. Advanced](./reflection/Reflection-03-Advanced.md)** | 프록시 및 고급 기법 | Dynamic Proxy, MethodHandle, Performance |

#### 🔄 Concurrency & Multithreading (7개)
| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[01. Thread & Runnable](./concurrency/Concurrency-01-Thread.md)** | 스레드 생성과 생명주기 | Thread, Runnable, sleep, join |
| **[02. Synchronization](./concurrency/Concurrency-02-Sync.md)** | 동기화와 락(Lock) 제어 | synchronized, volatile, ReentrantLock |
| **[03. ExecutorService](./concurrency/Concurrency-03-Executor.md)** | 스레드 풀 관리 프레임워크 | ThreadPoolExecutor, submit, shutdown |
| **[04. Concurrent Collections](./concurrency/Concurrency-04-Concurrent.md)** | 스레드 안전 컬렉션 | ConcurrentHashMap, CopyOnWriteArrayList |
| **[05. Atomic Variables](./concurrency/Concurrency-05-Atomic.md)** | 락 없는(Lock-free) 동기화 | AtomicInteger, CAS, ABA Problem |
| **[06. CompletableFuture](./concurrency/Concurrency-06-Future.md)** | 비동기 프로그래밍 패턴 | supplyAsync, thenApply, allOf |
| **[07. Virtual Threads](./concurrency/Concurrency-07-Virtual.md)** | Java 21 가상 스레드 | Virtual Thread, Structured Concurrency |

**총 문서 수: 16 + 8 + 16 + 10 + 9 + 10 = 69개 (확정)**

---

## 📐 문서 구조 (모든 문서 동일 적용)

```markdown
## 🎯 핵심 개념
이 API가 왜 존재하고 어떤 문제를 해결하는가.

## 📌 주요 메서드 정리
실무에서 자주 쓰는 메서드 표 (시그니처 + 설명 + 사용 예).

## 💻 실행 가능한 예제
복사-붙여넣기 즉시 실행 가능한 코드 (3~5개).
- 기본 사용법
- 응용 패턴
- 실무 시나리오

## 🔥 실전 패턴
실무에서 자주 쓰는 코드 패턴 정리.
예: "Map.computeIfAbsent로 카운팅하기", "TreeMap.headMap으로 범위 검색하기"

## 📊 성능 비교 (해당되는 경우)
같은 기능의 여러 방법 비교 — 시간 복잡도 + 마이크로벤치마크.
예: ArrayList vs LinkedList의 add/get/remove 시간 비교

## 🚫 함정 회피 가이드
초보자가 자주 하는 실수.
- "왜 String 비교에 == 쓰면 안 되는가"
- "왜 ArrayList.remove(i)가 무한루프를 만드는가"
- "왜 HashMap에 가변 객체를 키로 쓰면 안 되는가"

## 🔗 관련 API
같이 알아두면 좋은 API와의 관계.

## 📌 핵심 정리 (한 화면 요약)

## 🤔 실전 연습 문제 (+ 해설)
이 API를 진정으로 익혔는지 검증하는 문제 1~3개.
```

---

## 🎨 스타일 가이드

1. **모든 코드는 즉시 실행 가능** — `main` 메서드 또는 JShell에서 바로 돌아가야 함
2. **표 정리 강제** — 메서드 목록은 표로, 비교는 표로 (텍스트 나열 금지)
3. **함정 회피 섹션 강제** — "이 실수를 자주 한다" 패턴을 모든 문서에 1개 이상
4. **Java 21 기준** — 옛날 코드(`new Integer(1)`, `Date`)는 마이그레이션 가이드에서만
5. **코딩 테스트 활용 패턴 명시** — `PriorityQueue`, `TreeMap.floorKey`, `Deque` 등은 알고리즘 활용도 같이
6. **성능 수치는 추상적 표현 금지** — "빠르다" ❌ → "100만 건 기준 X ms" ✅

---

## 🛠️ 코드 환경

- Java 8+ 기본, Java 21까지 핵심 기능 다룸
- 단순 main 메서드 + JUnit 5 (성능 측정 시 JMH 활용 권장)
- 외부 라이브러리 의존성 최소화 (표준 라이브러리 위주)

---

## 🎯 2단계: 작업 순서

전체 구조는 위에 이미 확정되어 있으므로:

1. **디렉토리 생성**: bash 명령어로 다음 폴더 생성
   ```
   string/  math/  arrays/
   generics/  enum/  exception/
   collections/
   lambda/  modern-java/  utils/
   datetime/  io/
   reflection/  concurrency/
   ```
2. **README.md 작성**:
   - 6개 카테고리 표 형식 목차 (책 챕터 느낌으로 카테고리 → 하위 분류 → 문서)
   - 학습 로드맵 (입문 4주 / 실무자 / 코딩 테스트 준비 / 면접 / 빠른 복습)
   - "왜 그렇게 동작하는지" 강조
3. **카테고리별 문서 작성**:
   - 1. 기초 → 2. 타입 안전성 → 3. 자료구조 → 4. 모던 자바 → 5. 실무 필수 → 6. 심화 학습 순
   - 각 문서는 2000~3000 단어 분량
   - **모든 문서는 위 9섹션 구조 준수**
   - **함정 회피 + 실전 연습 문제 강제**

---

## 📚 참고 자료

- **Oracle Java SE Documentation** — docs.oracle.com/en/java/javase/21
- **Effective Java (3rd Edition)** — Joshua Bloch
- **Java SE Tutorial** — docs.oracle.com/javase/tutorial
- **JEPs** (특히 record / sealed / virtual thread)

<JVM Deep Dive README를 여기에 붙여넣기>
<Effective Java README를 여기에 붙여넣기>

위 README들의 비주얼 톤 (배지, 카테고리 분류 표) 유지하되, 깊이는 "쓰는 법" 중심으로.

---

## 💡 핵심 분석 대상

### HashMap 내부 구조 시각화 (Section 3)

```
HashMap<String, Integer> map = new HashMap<>(16, 0.75f);

내부 구조:
table = Node[16]
   ↓ index = hash("apple") & (16 - 1)
   [3] → Node("apple", 1, next=null)
   [7] → Node("banana", 2, next=Node("cat", 3, next=null))   // 해시 충돌 → 체이닝
   [9] → ...
```

→ 해시 충돌 시 체이닝 동작, Java 8부터 8개 이상이면 Tree화되는 메커니즘

### ArrayList vs LinkedList 100만 건 벤치마크 (Section 3)

| 작업 | ArrayList | LinkedList |
|:-----|:---------:|:----------:|
| `add(value)` (끝에 추가) | 5ms | 18ms |
| `add(0, value)` (앞에 추가) | 850ms | 8ms |
| `get(500_000)` (랜덤 접근) | 1ns | 2ms |
| `remove(0)` | 920ms | 1ns |

→ 시간 복잡도가 실측에서 어떻게 나타나는지 직접 측정

### Stream lazy evaluation (Section 4)

```java
Stream.of(1, 2, 3, 4, 5)
    .peek(x -> System.out.println("filter: " + x))
    .filter(x -> x % 2 == 0)
    .peek(x -> System.out.println("map: " + x))
    .map(x -> x * 2)
    .findFirst();

// 출력:
// filter: 1
// filter: 2
// map: 2
// (filter: 3, 4, 5는 실행 안 됨 — short-circuit)
```

→ 중간 연산이 즉시 실행되지 않고 terminal 연산이 필요할 때만 평가되는 메커니즘

---

## 🎯 진행 방식

**구조는 이미 확정되어 있으므로 1단계는 생략하고 바로 2단계(디렉토리 생성)부터 시작.**

진행 순서:
1. 디렉토리 14개 생성 (각 카테고리의 하위 폴더 단위)
2. README.md 작성
3. 카테고리별 문서 작성 (1.기초 → 2.타입안전 → 3.자료구조 → 4.모던자바 → 5.실무 → 6.심화)
4. 각 문서마다 9섹션 구조 강제 준수
5. 함정 회피 + 실전 연습 문제 모든 문서 포함

**시작해줘! 디렉토리 생성 후 README, 그다음 String 01부터.**
