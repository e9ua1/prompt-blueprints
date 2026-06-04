# Java Design Patterns 레포지토리 제작 프롬프트

나는 "Java Design Patterns" 레포지토리를 만들려고 해.
GoF 패턴부터 아키텍처·동시성·Modern Java 패턴까지, **단순 패턴 나열이 아닌 "왜 이 설계가 필요한가" 의 문제-해결 구조로** 47개 패턴을 다루는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "단순한 코드 작성을 넘어, 확장 가능하고 유지보수하기 쉬운 설계로"

**핵심 차별화**:
1. 패턴 정의 나열 ❌ → **문제 상황 → 패턴 적용 → Before/After 비교** ✅
2. 다이어그램만 보여주는 책의 한계를 넘어, **모두 실행 가능한 Java 코드** 로 작성
3. **47개 패턴을 7개 카테고리** 로 묶어 GoF 23개에 더해 아키텍처·엔터프라이즈·동시성·Modern Java까지 확장
4. 패턴 간 관계 비교 (Adapter vs Decorator vs Proxy / Strategy vs Template Method 등)
5. **안티패턴 가이드** — "이 패턴을 잘못 쓰면 어떻게 되는가"

**타겟 독자**:
- 패턴 이름은 알지만 언제 써야 할지 모르는 개발자
- 디자인 패턴을 외웠지만 실제 코드에 적용 못 하는 개발자
- "Decorator vs Proxy 차이가 뭐예요" 면접 질문에 막히는 개발자
- Spring 프레임워크가 어떤 패턴을 쓰는지 궁금한 개발자
- 아키텍처 레벨로 시야를 확장하고 싶은 시니어 지망생
- 동시성 패턴을 처음 접하는 백엔드 개발자

**기존 레포와의 관계**:
- **Object (조영호)**: 책임 주도 설계 → 패턴이 등장하는 설계 문제로 자연스럽게 연결
- **Effective Java**: 모범 사례와 패턴이 어떻게 맞물리는가
- **Spring Core Deep Dive**: Spring이 사용하는 패턴들 (Proxy, Strategy, Template Method 등)
- **Modern Java in Action**: Functional Interface, Stream Pipeline 같은 Modern Java 패턴

---

## 🎯 1단계: 전체 구조 (이미 확정 — 변경 금지)

이 레포는 **7개 카테고리, 총 47개 패턴 문서**로 구성된다. 카테고리·패턴 제목·핵심 키워드는 아래에 고정되어 있으며 추가/삭제/순서 변경 없이 이대로 작성한다.

### 🔹 1. 생성 패턴 (Creational Patterns) — 6개
> 객체 생성 메커니즘을 다루는 패턴

| Pattern | 문제 상황 | 핵심 개념 |
|:--------|----------|-----------|
| **[1. Singleton](./creational/01-Singleton.md)** | 전역 인스턴스가 필요할 때 | Thread-safe, Enum, Lazy Initialization |
| **[2. Factory Method](./creational/02-FactoryMethod.md)** | 객체 생성 로직을 분리하고 싶을 때 | 인터페이스 기반, 확장성 |
| **[3. Builder](./creational/03-Builder.md)** | 복잡한 객체를 단계별로 생성할 때 | Fluent API, Immutable |
| **[4. Prototype](./creational/04-Prototype.md)** | 객체를 복제해서 생성할 때 | Clone, Deep Copy |
| **[5. Abstract Factory](./creational/05-AbstractFactory.md)** | 관련 객체군을 생성할 때 | 제품군, 플랫폼 독립 |
| **[6. Object Pool](./creational/06-ObjectPool.md)** | 비용이 큰 객체를 재사용할 때 | Connection Pool, Thread Pool |

### 🔹 2. 구조 패턴 (Structural Patterns) — 7개
> 클래스와 객체를 조합하는 패턴

| Pattern | 문제 상황 | 핵심 개념 |
|:--------|----------|-----------|
| **[1. Adapter](./structural/01-Adapter.md)** | 호환되지 않는 인터페이스를 연결할 때 | Wrapper, 레거시 통합 |
| **[2. Decorator](./structural/02-Decorator.md)** | 동적으로 기능을 추가할 때 | 상속 대신 조합, Wrapping |
| **[3. Proxy](./structural/03-Proxy.md)** | 객체 접근을 제어할 때 | Lazy Loading, 권한 체크 |
| **[4. Composite](./structural/04-Composite.md)** | 트리 구조로 객체를 다룰 때 | 부분-전체, 재귀적 구조 |
| **[5. Facade](./structural/05-Facade.md)** | 복잡한 서브시스템을 단순화할 때 | 통합 인터페이스, 결합도 감소 |
| **[6. Bridge](./structural/06-Bridge.md)** | 추상화와 구현을 분리할 때 | 다차원 확장, 독립적 변경 |
| **[7. Flyweight](./structural/07-Flyweight.md)** | 많은 객체를 효율적으로 관리할 때 | 공유, 메모리 절약 |

### 🔹 3. 행위 패턴 (Behavioral Patterns) — 11개
> 객체 간 책임과 알고리즘 분배 패턴

| Pattern | 문제 상황 | 핵심 개념 |
|:--------|----------|-----------|
| **[1. Strategy](./behavioral/01-Strategy.md)** | 알고리즘을 교체 가능하게 할 때 | 정책 분리, 런타임 교체 |
| **[2. Observer](./behavioral/02-Observer.md)** | 이벤트를 구독/발행할 때 | 일대다 의존, Pub/Sub |
| **[3. Template Method](./behavioral/03-TemplateMethod.md)** | 알고리즘 골격을 정의할 때 | Hook 메서드, 제어 역전 |
| **[4. Command](./behavioral/04-Command.md)** | 요청을 객체로 캡슐화할 때 | Undo/Redo, 트랜잭션 |
| **[5. Iterator](./behavioral/05-Iterator.md)** | 컬렉션을 순회할 때 | 내부 구조 감춤, 표준화 |
| **[6. State](./behavioral/06-State.md)** | 상태에 따라 행위가 변할 때 | 상태 기계, 조건문 제거 |
| **[7. Chain of Responsibility](./behavioral/07-ChainOfResponsibility.md)** | 요청을 순차적으로 처리할 때 | 책임 연쇄, 파이프라인 |
| **[8. Mediator](./behavioral/08-Mediator.md)** | 객체 간 상호작용을 중재할 때 | 결합도 감소, 중앙 제어 |
| **[9. Memento](./behavioral/09-Memento.md)** | 객체 상태를 저장/복원할 때 | 스냅샷, 이력 관리 |
| **[10. Visitor](./behavioral/10-Visitor.md)** | 구조와 연산을 분리할 때 | Double Dispatch, 확장성 |
| **[11. Interpreter](./behavioral/11-Interpreter.md)** | 언어 문법을 해석할 때 | AST, 파서 |

### 🔹 4. 아키텍처 패턴 (Architectural Patterns) — 8개
> 시스템 레벨 구조 패턴

| Pattern | 문제 상황 | 핵심 개념 |
|:--------|----------|-----------|
| **[1. Layered Architecture](./architectural/01-LayeredArchitecture.md)** | 계층을 분리할 때 | Presentation-Business-Data |
| **[2. MVC](./architectural/02-MVC.md)** | UI 로직을 분리할 때 | Model-View-Controller |
| **[3. Repository](./architectural/03-Repository.md)** | 데이터 접근을 추상화할 때 | Domain Model 보호 |
| **[4. Hexagonal](./architectural/04-Hexagonal.md)** | 도메인을 격리할 때 | Ports & Adapters, DDD |
| **[5. MVVM](./architectural/05-MVVM.md)** | 데이터 바인딩이 필요할 때 | ViewModel, Reactive |
| **[6. Event-Driven](./architectural/06-EventDriven.md)** | 비동기 통신이 필요할 때 | Message Queue, 느슨한 결합 |
| **[7. Microservices](./architectural/07-Microservices.md)** | 독립 배포가 필요할 때 | Service Mesh, API Gateway |
| **[8. MVP](./architectural/08-MVP.md)** | View를 수동적으로 만들 때 | Presenter, Testability |

### 🔹 5. 엔터프라이즈 패턴 (Enterprise Patterns) — 5개
> 비즈니스 로직 구현 패턴

| Pattern | 문제 상황 | 핵심 개념 |
|:--------|----------|-----------|
| **[1. DTO](./enterprise/01-DTO.md)** | 계층 간 데이터를 전송할 때 | 직렬화, API 응답 |
| **[2. DAO](./enterprise/02-DAO.md)** | DB 접근을 캡슐화할 때 | CRUD, SQL 격리 |
| **[3. Service Layer](./enterprise/03-ServiceLayer.md)** | 비즈니스 로직을 분리할 때 | Transaction, Orchestration |
| **[4. Unit of Work](./enterprise/04-UnitOfWork.md)** | 트랜잭션을 관리할 때 | 변경 추적, 일괄 처리 |
| **[5. Specification](./enterprise/05-Specification.md)** | 비즈니스 규칙을 캡슐화할 때 | 조합 가능, 재사용 |

### 🔹 6. Modern Java 패턴 (Modern Java Patterns) — 4개
> Java 8+ 기능 활용 패턴

| Pattern | 문제 상황 | 핵심 개념 |
|:--------|----------|-----------|
| **[1. Functional Interface](./modern/01-FunctionalInterface.md)** | 함수를 일급 객체로 다룰 때 | Lambda, Method Reference |
| **[2. Stream Pipeline](./modern/02-StreamPipeline.md)** | 데이터를 선언적으로 처리할 때 | filter-map-collect, 지연 평가 |
| **[3. Optional Chaining](./modern/03-OptionalChaining.md)** | Null을 안전하게 처리할 때 | ofNullable, orElse |
| **[4. Sealed Classes](./modern/04-SealedClasses.md)** | 타입을 제한할 때 | Pattern Matching, 타입 안전 |

### 🔹 7. 동시성 패턴 (Concurrency Patterns) — 6개
> 멀티스레딩 환경의 패턴

| Pattern | 문제 상황 | 핵심 개념 |
|:--------|----------|-----------|
| **[1. Thread Pool](./concurrency/01-ThreadPool.md)** | 스레드를 효율적으로 관리할 때 | ExecutorService, 재사용 |
| **[2. Producer-Consumer](./concurrency/02-ProducerConsumer.md)** | 생산/소비를 분리할 때 | BlockingQueue, 버퍼 |
| **[3. Reader-Writer Lock](./concurrency/03-ReaderWriterLock.md)** | 읽기/쓰기를 분리할 때 | 동시 읽기, 배타적 쓰기 |
| **[4. Double-Checked Locking](./concurrency/04-DoubleCheckedLocking.md)** | Singleton을 최적화할 때 | volatile, 성능 |
| **[5. Active Object](./concurrency/05-ActiveObject.md)** | 비동기 메서드를 호출할 때 | Actor Model, 메시지 |
| **[6. Future/Promise](./concurrency/06-FuturePromise.md)** | 비동기 결과를 처리할 때 | CompletableFuture, 콜백 |

**총 패턴 수: 6 + 7 + 11 + 8 + 5 + 4 + 6 = 47개 (확정)**

---

## 📐 패턴 문서 구조 (모든 문서 동일 적용)

```markdown
## 🎯 문제 상황 (이 패턴이 필요한 실제 시나리오)
## 📌 핵심 개념 (구조 + 참여자 + 협력 방식)
## 💻 구현 예제 (Before — 패턴 미적용 / After — 패턴 적용)
## 🔥 실전 사례 (JDK / Spring / 라이브러리에서 실제로 쓰이는 곳)
## 🔗 관련 패턴 비교 (Decorator vs Proxy 같은 헷갈리는 패턴 차이)
## ⚡ 장단점 (트레이드오프 분석)
## 🚫 안티패턴 (이 패턴을 잘못 쓰면 어떻게 되는가)
## 🎯 적용 기준 (언제 써야 하고 언제 쓰지 말아야 하는가)
## 📌 핵심 정리 (한 화면 요약)
## 🤔 생각해볼 문제 (+ 해설)
```

---

## 🎨 스타일 가이드

1. **모든 패턴은 "문제 상황"으로 시작** — "Singleton 패턴은 ~이다"가 아니라 "전역 설정이 필요한데 인스턴스가 두 개 생기면 이런 문제가 발생한다"
2. **Before/After 코드는 같은 도메인으로** — "패턴이 없을 때 어떻게 못생겼는지" 와 "패턴 적용 후 어떻게 깔끔해졌는지" 한 화면에 비교
3. **JDK / Spring 실전 사례 강제** — 모든 패턴은 표준 라이브러리나 Spring에서 어디에 쓰이는지 명시
4. **헷갈리는 패턴 비교 강제** — Adapter vs Decorator vs Proxy / Strategy vs Template Method / Factory vs Abstract Factory 등은 표 비교
5. **Java 21 + Modern 기능** 으로 작성 (record, Sealed Class, Pattern Matching, Switch Expression)
6. **클래스 다이어그램은 Mermaid 사용** — 외부 이미지 의존성 없이 GitHub에서 바로 렌더링

---

## 🛠️ 코드 환경

- Java 21 (record, Sealed, Pattern Matching 적극 활용)
- Maven 또는 Gradle 8.x
- 실행 가능한 main 메서드 또는 JUnit 5 테스트로 검증
- Spring Boot 예제는 별도 모듈로 분리

---

## 🎯 2단계: 작업 순서

전체 구조는 위에 이미 확정되어 있으므로:

1. **디렉토리 생성**: bash 명령어로 7개 카테고리 폴더 생성
   ```
   creational/
   structural/
   behavioral/
   architectural/
   enterprise/
   modern/
   concurrency/
   ```
2. **README.md 작성**:
   - "47가지 디자인 패턴" 강조
   - 7개 카테고리 표 형식 목차
   - 학습 로드맵 (입문 4주 / 실무자 4주 / 면접 / 아키텍트 / 빠른 복습)
   - "일반 자료 vs 이 레포" 비교
   - 패턴 간 관계 비교 표 (Adapter vs Decorator vs Proxy 등)
3. **카테고리별 문서 작성**:
   - 생성 → 구조 → 행위 → 아키텍처 → 엔터프라이즈 → Modern Java → 동시성 순
   - 각 문서는 2500~3500 단어 분량
   - **모든 문서는 위 10섹션 구조 준수**
   - **모든 패턴은 Before/After 코드 + JDK·Spring 실전 사례 포함**

---

## 📚 참고 자료

- **Design Patterns: Elements of Reusable Object-Oriented Software** — Gamma, Helm, Johnson, Vlissides (GoF)
- **Head First Design Patterns** — Eric & Elisabeth Freeman
- **Refactoring Guru** — refactoring.guru/design-patterns
- **Patterns of Enterprise Application Architecture** — Martin Fowler (PoEAA, 엔터프라이즈 패턴)
- **Java Concurrency in Practice** — Brian Goetz (동시성 패턴)
- **Implementing Domain-Driven Design** — Vaughn Vernon (Hexagonal, Specification)

<Object README를 여기에 붙여넣기>
<Effective Java README를 여기에 붙여넣기>

위 README들의 비주얼 톤 (배지, 표, 카테고리 분류) 유지.

---

## 💡 핵심 분석 대상

### Singleton — Java 21 기준 권장 구현 비교

```java
// ❌ Lazy Initialization with synchronized — 성능 저하
public static synchronized Singleton getInstance() { ... }

// ❌ Double-Checked Locking — JMM 이해 필요, 실수 위험
public static Singleton getInstance() {
    if (instance == null) {
        synchronized (Singleton.class) {
            if (instance == null) instance = new Singleton();
        }
    }
}

// ✅ Initialization-on-Demand Holder — Lazy + Thread-safe + 빠름
private static class Holder { static final Singleton INSTANCE = new Singleton(); }
public static Singleton getInstance() { return Holder.INSTANCE; }

// ✅ Enum — 가장 안전 (직렬화·리플렉션 공격 방어)
public enum Singleton { INSTANCE; }
```

→ 4가지 구현의 트레이드오프, JVM이 클래스 로딩을 어떻게 보장하는지 추적 (Concurrency Section의 Double-Checked Locking과 연결)

### Strategy vs Template Method (Behavioral Section)

```java
// Strategy — 합성, 런타임 교체 가능
class Sorter {
    private SortStrategy strategy;
    void sort(List<?> list) { strategy.sort(list); }
}

// Template Method — 상속, 컴파일 타임 고정
abstract class AbstractSorter {
    final void sort(List<?> list) {
        prepare();
        doSort(list);  // 추상 메서드, 서브클래스가 구현
        cleanup();
    }
    protected abstract void doSort(List<?> list);
}
```

→ 같은 "알고리즘 변경" 문제를 다르게 해결하는 두 패턴의 트레이드오프 비교

### Spring이 사용하는 패턴 매핑

| Spring 기능 | 사용 패턴 |
|:------------|:---------|
| `JdbcTemplate.execute()` | Template Method |
| `BeanFactory` / `ApplicationContext` | Factory Method, Singleton |
| `@Transactional` 프록시 | Proxy, Decorator |
| `HandlerInterceptor` 체인 | Chain of Responsibility |
| `ApplicationEventPublisher` | Observer |
| `ConversionService` | Strategy |
| `EnableAutoConfiguration` | Builder + Factory |

---

## 🎯 진행 방식

**구조는 이미 확정되어 있으므로 1단계는 생략하고 바로 2단계(디렉토리 생성)부터 시작.**

진행 순서:
1. 디렉토리 7개 생성
2. README.md 작성 (47개 패턴 표 + 학습 로드맵)
3. 카테고리별 문서 작성 (생성 → 구조 → 행위 → 아키텍처 → 엔터프라이즈 → Modern → 동시성)
4. 각 문서마다 10섹션 구조 강제 준수
5. Before/After + JDK/Spring 실전 사례 + 관련 패턴 비교 강제

**시작해줘! 디렉토리 생성 후 README, 그다음 Creational의 Singleton부터.**
