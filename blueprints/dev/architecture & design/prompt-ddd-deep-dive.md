# DDD Deep Dive 레포지토리 제작 프롬프트

나는 "DDD Deep Dive" 레포지토리를 만들려고 해.
엔티티와 리포지토리를 쓰는 것과, 왜 Aggregate가 일관성 경계인지, Bounded Context가 어떻게 팀과 코드를 분리하는지, Domain Event가 왜 결합도를 낮추는지를 완전히 파헤치는 레포를 만들 거야.

"@Entity 붙이고 JpaRepository 상속하면 DDD겠지"와 "왜 Aggregate Root를 통해서만 내부 객체를 변경해야 하는지, 왜 Value Object는 불변이어야 하는지, 왜 Repository는 Aggregate 단위인지 아는 것"의 차이를 만드는 레포야.

## 📋 프로젝트 목표

**컨셉**: "엔티티와 서비스를 만드는 것과, 도메인의 불변식(Invariant)을 보호하는 객체 설계를 아는 것은 다르다"

**핵심 차별화**:
1. Strategic Design — Bounded Context와 Ubiquitous Language가 왜 코드보다 먼저인가
2. Aggregate 설계 — 왜 Aggregate Root가 일관성 경계이며, 크기를 작게 유지해야 하는 이유
3. Domain Event — 이벤트가 어떻게 Bounded Context 간 결합도를 낮추는가
4. Spring 실전 — JPA Entity와 DDD Entity의 차이, 영속성 무지(Persistence Ignorance) 구현

**타겟 독자**:
- Service 클래스에 모든 비즈니스 로직을 넣고 Entity는 getter/setter만 있는 개발자
- Repository를 테이블당 하나씩 만드는 개발자
- Bounded Context와 마이크로서비스를 동일하게 생각하는 개발자
- Domain Event 없이 서비스 간 직접 호출로 결합도를 높이는 개발자
- MSA를 적용했지만 분산된 모놀리스가 되어버린 코드를 짜는 개발자
- Spring Data JPA를 쓰면서 도메인 모델이 영속성에 오염된 개발자

**선행 학습**:
- spring-data-transaction (JPA/Hibernate 기본 동작 이해 필수)
- spring-core-deep-dive (Bean 생명주기, DI 이해 시 시너지)
- kafka-deep-dive (Domain Event를 Kafka로 발행하는 Outbox Pattern 연결)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: DDD란 무엇인가 — 설계의 철학 (5개 문서)
- DDD가 해결하는 문제 — 소프트웨어 복잡도의 본질, 기술 중심 설계 vs 도메인 중심 설계의 차이, Anemic Domain Model의 함정
- Ubiquitous Language — 왜 개발자와 도메인 전문가가 같은 언어를 써야 하는가, 언어가 코드 구조를 결정하는 방식, 잘못된 언어가 버그를 만드는 사례
- Strategic Design vs Tactical Design — Big Picture(Context Map)부터 시작하는 이유, 전략과 전술의 분리, 언제 어떤 설계 도구를 쓰는가
- DDD 적용 판단 기준 — 모든 프로젝트에 DDD가 답인가, 도메인 복잡도에 따른 아키텍처 선택(단순 CRUD vs 복잡한 도메인), DDD 적용 비용과 편익
- DDD와 기존 레이어드 아키텍처 비교 — Controller → Service → Repository 구조의 문제점, 도메인 레이어가 분리되었을 때의 실제 차이

### Chapter 2: Strategic Design — 큰 그림 그리기 (7개 문서)
- 서브도메인(Subdomain) 분류 — Core/Supporting/Generic Subdomain의 차이, 왜 Core Domain에 집중해야 하는가, 전략적 투자 결정
- Bounded Context 완전 분해 — Context가 경계를 정하는 기준(언어의 경계, 팀의 경계), 하나의 개념이 Context마다 다른 모델을 갖는 이유, Order가 배송 컨텍스트와 결제 컨텍스트에서 다른 이유
- Ubiquitous Language 실전 — 언어가 Bounded Context마다 다를 수 있는 이유, 언어 충돌이 설계 문제의 신호인 이유, 글로시어리(Glossary) 작성법
- Context Map 패턴 완전 분해 — Shared Kernel/Customer-Supplier/Conformist/Anticorruption Layer/Open Host Service/Published Language/Separate Ways 각각의 의미와 선택 기준, 팀 관계가 Context Map을 결정하는 이유
- Anticorruption Layer — 외부 시스템/레거시 시스템의 모델이 도메인 모델을 오염시키지 않도록 하는 방법, ACL이 번역기(Translator)인 이유, Spring에서 ACL 구현 패턴
- 이벤트 스토밍(Event Storming) — 도메인 전문가와 함께 Bounded Context를 발견하는 워크숍 방법, Command/Event/Aggregate/Policy 스티커로 도메인 흐름 시각화
- Bounded Context와 마이크로서비스 — Context ≠ 서비스인 이유, 하나의 Context가 여러 서비스가 될 수 있는 경우, Context가 준비되기 전에 서비스로 분리하면 생기는 문제

### Chapter 3: Tactical Design — 도메인 모델 설계 (8개 문서)
- Entity vs Value Object — 식별자(Identity)로 구분하는 Entity, 값(Value)으로 구분하는 Value Object, 왜 Money/Address/Email은 Value Object여야 하는가, equals() 구현의 차이
- Value Object 설계 원칙 — 불변성(Immutability)이 왜 필수인가, 자기 검증(Self-Validation) 원칙, Value Object가 행동을 가질 수 있는 이유, Primitive Obsession 안티패턴
- Aggregate 설계 — Aggregate가 일관성 경계(Consistency Boundary)인 이유, Root를 통해서만 내부를 변경해야 하는 이유, Aggregate 경계 설정의 실전 기준
- Aggregate 크기 결정 — 작은 Aggregate를 선호해야 하는 이유(트랜잭션 충돌 최소화), Aggregate 간 참조는 ID로만 하는 이유, 큰 Aggregate의 실제 문제(락 경합, 로딩 비용)
- Domain Service vs Application Service — 도메인 서비스가 필요한 경우(단일 Entity로 표현할 수 없는 도메인 로직), Application Service의 역할(오케스트레이션, 트랜잭션 경계), 두 서비스를 혼동했을 때 생기는 문제
- Repository 설계 원칙 — Repository는 Aggregate 단위인 이유, 컬렉션처럼 동작해야 하는 이유(Add/Remove/Find), 테이블당 Repository의 문제점, 도메인 레이어가 Repository 인터페이스를 소유하는 이유
- Factory 패턴 — 복잡한 Aggregate 생성을 Factory로 분리하는 이유, 생성자 vs Factory Method vs Factory 클래스 선택 기준, 불변식 검증을 생성 시점에 강제하는 방법
- Domain Event 설계 — 이벤트가 과거형 이름을 가져야 하는 이유(OrderPlaced, PaymentCompleted), 이벤트가 포함해야 할 정보(최소한의 데이터 vs 풍부한 데이터), 이벤트 버저닝 전략

### Chapter 4: Domain Event와 Bounded Context 통합 (6개 문서)
- Domain Event가 결합도를 낮추는 원리 — 이벤트 발행자가 구독자를 모르는 설계, 동기 직접 호출 vs 이벤트 기반 통신의 일관성 트레이드오프
- 이벤트 발행 패턴 — Aggregate 내부에서 이벤트를 수집하고 Application Service에서 발행하는 방법, Spring의 ApplicationEventPublisher vs Kafka 발행 선택 기준
- Outbox Pattern 완전 분해 — DB 트랜잭션과 이벤트 발행의 원자성 문제, Outbox 테이블로 Dual Write 문제 해결, Polling Publisher vs CDC(Debezium) 구현 비교
- Saga Pattern — 분산 트랜잭션 없이 여러 Bounded Context에 걸친 비즈니스 프로세스 조율, Choreography Saga vs Orchestration Saga 선택 기준, 보상 트랜잭션(Compensating Transaction) 설계
- Context 간 데이터 동기화 — Event Sourcing 없이 Context 간 읽기 모델 동기화, Eventual Consistency 허용 설계, 정합성 검증 전략
- Anti-Corruption Layer 실전 구현 — 외부 API 응답을 도메인 모델로 변환하는 Translator, Mapper 패턴, 외부 모델 변경이 도메인 모델에 전파되지 않도록 하는 방법

### Chapter 5: Spring + JPA로 DDD 구현하기 (7개 문서)
- Persistence Ignorance 원칙 — 도메인 모델이 JPA를 몰라야 하는 이유, @Entity 어노테이션이 도메인 모델을 오염시키는 방법, 실용적 타협과 순수 분리의 트레이드오프
- JPA Entity와 DDD Entity 분리 패턴 — Domain Object와 JPA Entity를 완전히 분리하는 방법, Mapper로 변환하는 비용, 두 모델이 섞였을 때의 문제
- Value Object JPA 매핑 — @Embedded와 @Embeddable로 Value Object 매핑, 불변 Value Object를 JPA와 함께 쓰는 방법, AttributeConverter 활용
- Aggregate JPA 매핑 — CascadeType.ALL의 함정, Aggregate 경계가 JPA Fetch 전략을 결정하는 이유, LazyLoading이 Aggregate를 깨는 방법
- Repository 구현 — Spring Data JPA Repository를 도메인 Repository 인터페이스의 구현체로 사용하는 방법, 인프라 레이어에서 JpaRepository 상속, 도메인 레이어에는 인터페이스만
- Domain Event Spring 통합 — AbstractAggregateRoot로 이벤트 수집, @DomainEvents와 @AfterDomainEventsPublication, save() 이후 자동 이벤트 발행
- 테스트 전략 — 도메인 모델 단위 테스트(인프라 없이), Application Service 통합 테스트, Aggregate 불변식 테스트 패턴

### Chapter 6: DDD 실전 프로젝트 — 전자상거래 도메인 (5개 문서)
- 도메인 분석 — 전자상거래의 Bounded Context 식별(주문/상품/회원/배송/결제), Context Map 작성, 각 Context의 핵심 도메인 모델
- 주문 Aggregate 설계 — Order/OrderLine/OrderStatus의 불변식, 주문 상태 전이 로직을 Aggregate 안에 캡슐화하는 방법, 결제 완료 전 배송 시작 방지
- Bounded Context 간 통합 — 주문 완료 이벤트가 배송/재고/결제 Context로 전파되는 흐름, Saga로 주문 취소 시 재고 복원 구현
- 읽기 모델 분리 — 주문 목록 조회를 위한 별도 읽기 모델, CQRS와의 연결(다음 레포 연결 포인트), 쓰기 모델과 읽기 모델의 동기화 전략
- DDD 리팩터링 — 기존 Transaction Script 코드를 DDD로 점진적 리팩터링하는 전략, Strangler Fig 패턴으로 레거시 개선

### Chapter 7: DDD 안티패턴과 트레이드오프 (5개 문서)
- Anemic Domain Model 안티패턴 — Entity가 데이터 컨테이너가 되는 문제, Service에 모든 로직이 쌓이는 이유, 리팩터링 전략
- Aggregate 설계 실수 — 너무 큰 Aggregate(God Object), Aggregate 간 직접 객체 참조, 트랜잭션 범위가 Aggregate를 넘어가는 설계
- Bounded Context 경계 실수 — 너무 세분화된 Context(분산 모놀리스), 너무 큰 Context(모놀리스와 다를 바 없음), 경계 재조정 전략
- DDD 과잉 적용 — CRUD에 DDD를 적용했을 때의 과잉 복잡도, 도메인 복잡도에 따른 적절한 설계 수준 선택
- 실무에서의 타협 — 완벽한 DDD는 없다, JPA 오염 허용 범위, 팀 역량과 도메인 복잡도에 따른 점진적 적용 전략

---

총 **43개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 실무에서 중요한가
## 😱 흔한 실수 (Before — DDD 없이 짜는 코드)
## ✨ 올바른 접근 (After — DDD로 짠 코드)
## 🔬 내부 동작 원리 (왜 이 설계가 더 나은가 — 불변식, 캡슐화, 결합도 분석)
## 💻 실전 코드 (Spring + JPA 기반 구현, 단위 테스트 포함)
## 📊 설계 비교 (Anemic vs Rich Domain Model, 직접 호출 vs 이벤트 등)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 모든 개념은 실제 코드로 Before/After 비교 — 추상적 설명 없이 구체적 코드 대비
2. 불변식(Invariant)을 항상 명시 — "이 설계가 어떤 비즈니스 규칙을 보호하는가"
3. 잘못된 설계의 실제 버그 시나리오 포함 — "이렇게 짜면 언제 장애가 나는가"
4. Spring Data JPA와 연결 — 도메인 설계 결정이 JPA 설정에 어떻게 영향을 미치는가
5. 다음 레포(MSA, CQRS)와의 연결 지점 명시

**실험 환경**:
```yaml
# docker-compose.yml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: ddd_shop
    ports:
      - "3306:3306"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
    ports:
      - "9092:9092"

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
```

```java
// 핵심 분석 대상 — Before (Anemic Domain Model)
@Entity
public class Order {
    @Id private Long id;
    private String status;
    private List<OrderLine> lines;
    // getter/setter만 있음
}

@Service
public class OrderService {
    public void placeOrder(OrderRequest request) {
        Order order = new Order();
        order.setStatus("PENDING");
        // 모든 비즈니스 로직이 Service에
        if (request.getItems().isEmpty()) throw new Exception("...");
        // ...
    }
}

// After (Rich Domain Model with DDD)
public class Order {  // Aggregate Root
    private OrderId id;
    private CustomerId customerId;
    private OrderStatus status;
    private List<OrderLine> lines;
    private List<DomainEvent> events = new ArrayList<>();

    // 불변식: 주문은 최소 1개 이상의 상품을 가져야 한다
    // 불변식: 취소된 주문에는 상품을 추가할 수 없다
    public static Order place(CustomerId customerId, List<OrderLineRequest> lines) {
        if (lines.isEmpty()) throw new EmptyOrderException();
        Order order = new Order(customerId, lines);
        order.events.add(new OrderPlaced(order.id, customerId));
        return order;
    }

    public void addLine(ProductId productId, int quantity, Money price) {
        if (this.status != OrderStatus.PENDING)
            throw new OrderNotModifiableException();
        this.lines.add(new OrderLine(productId, quantity, price));
    }

    public Money totalAmount() {
        return lines.stream()
            .map(OrderLine::subtotal)
            .reduce(Money.ZERO, Money::add);
    }
}
```

---

## 💡 핵심 분석 대상

```
DDD 설계 전체 흐름:

1. Strategic Design (큰 그림)
   비즈니스 도메인 분석
     ↓
   Subdomain 식별 (Core / Supporting / Generic)
     ↓
   Bounded Context 경계 설정 (언어의 경계, 팀의 경계)
     ↓
   Context Map 작성 (컨텍스트 간 관계 정의)

2. Tactical Design (구현)
   각 Bounded Context 내부:
     ├── Aggregate (일관성 경계)
     │     ├── Aggregate Root (진입점)
     │     ├── Entity (식별자로 구분)
     │     └── Value Object (값으로 구분, 불변)
     ├── Domain Service (Aggregate로 표현 불가한 도메인 로직)
     ├── Repository (Aggregate 단위, 컬렉션처럼)
     ├── Factory (복잡한 Aggregate 생성)
     └── Domain Event (Context 간 통신)

3. Bounded Context 통합
   주문 Context  →  [OrderPlaced 이벤트]  →  배송 Context
   주문 Context  →  [OrderPlaced 이벤트]  →  재고 Context
   주문 Context  →  [OrderPlaced 이벤트]  →  결제 Context

Aggregate 불변식 보호:
   외부 → Order.addLine() (Aggregate Root의 메서드만 허용)
   ❌ 금지: order.getLines().add(new OrderLine()) // 불변식 우회
   ✅ 허용: order.addLine(productId, qty, price)  // 불변식 검증 포함

   왜 Aggregate Root를 통해서만?
   Order.addLine() 내부:
     → status 검사 (취소된 주문에는 추가 불가)
     → 최대 수량 검사
     → 재고 확인 신호 발송
   직접 접근하면 이 검증들이 모두 우회됨

Value Object vs Entity:
   Money — 100원짜리 A와 100원짜리 B는 같다 (값이 같으면 동일)
   Order  — 주문 #1과 주문 #2는 내용이 같아도 다르다 (ID로 구분)

   Money를 Entity로 만들면?
     → money.setAmount(200) // 불변성 파괴
     → 다른 Order가 같은 Money 인스턴스 공유 → 버그 발생

Outbox Pattern:
   문제: DB 저장과 이벤트 발행을 어떻게 원자적으로?
   ❌ 잘못된 방법:
     transaction.begin()
     repository.save(order)       // DB 성공
     kafka.send(OrderPlaced)      // Kafka 실패 → 이벤트 유실!
     transaction.commit()

   ✅ Outbox Pattern:
     transaction.begin()
     repository.save(order)       // Order 저장
     outbox.save(OrderPlaced)     // 같은 DB 트랜잭션에 이벤트 저장
     transaction.commit()
     // 별도 프로세스가 Outbox 테이블 폴링 → Kafka 발행
     // 또는 Debezium이 binlog 감지 → Kafka 발행
```

---

## 📚 참고 자료

- Domain-Driven Design: Tackling Complexity in the Heart of Software (Eric Evans) — "The Blue Book"
- Implementing Domain-Driven Design (Vaughn Vernon) — "The Red Book"
- Domain-Driven Design Distilled (Vaughn Vernon)
- Patterns, Principles, and Practices of Domain-Driven Design (Scott Millett & Nick Tune)
- 에릭 에반스 DDD 공식 참고 자료 — https://dddcommunity.org/
- Martin Fowler의 DDD 관련 글 — https://martinfowler.com/tags/domain driven design.html
- Udi Dahan Blog — https://udidahan.com/ (Aggregate, Domain Event 설계)

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~8개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (43개 목표)
- Spring + JPA + Kafka 실험 환경
- kafka-deep-dive(Outbox Pattern), cqrs-event-sourcing(다음 레포), msa-deep-dive와 연결되는 지점 명시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
