# Architecture Patterns Deep Dive 레포지토리 제작 프롬프트

나는 "Architecture Patterns Deep Dive" 레포지토리를 만들려고 해.
Controller-Service-Repository 구조를 쓰는 것과, 왜 의존성이 안으로만 향해야 하는지, Hexagonal Architecture가 어떻게 포트와 어댑터로 인프라를 교체 가능하게 만드는지, Clean Architecture와 DDD가 어떻게 연결되는지를 완전히 파헤치는 레포야.

"레이어드 아키텍처를 쓰고 있다"와 "각 레이어의 의존성 방향이 왜 단방향이어야 하는지, 의존성이 역전될 때 무슨 일이 일어나는지, 도메인 레이어가 왜 어떤 프레임워크도 몰라야 하는지 아는 것"의 차이를 만드는 레포야.

## 📋 프로젝트 목표

**컨셉**: "아키텍처는 폴더 구조가 아니다 — 변경이 전파되는 방향을 결정하는 것이다"

**핵심 차별화**:
1. 의존성 방향의 원칙 — 변경이 어디서 시작되어 어디로 전파되는가, 의존성이 역전될 때 생기는 실제 문제
2. Hexagonal Architecture — Port가 어떻게 인프라를 추상화하고, Adapter가 어떻게 Port를 구현하는가
3. Clean Architecture — Uncle Bob의 4원칙을 Spring에서 실제로 구현하는 방법
4. DDD와 통합 — DDD Aggregate가 Clean Architecture의 어느 레이어에 위치하는가, 실전 패키지 구조

**타겟 독자**:
- Controller에서 @Autowired JpaRepository를 직접 쓰는 개발자
- Service가 HttpServletRequest를 파라미터로 받는 코드를 짜는 개발자
- 테스트를 위해 DB와 Spring Context가 항상 필요한 코드를 짜는 개발자
- 레이어드 아키텍처를 쓰지만 레이어 간 의존성이 뒤섞인 개발자
- Hexagonal Architecture를 들어봤지만 Port와 Adapter가 정확히 무엇인지 모르는 개발자
- MSA/DDD를 적용하려는데 패키지 구조를 어떻게 잡을지 모르는 개발자

**선행 학습**:
- spring-core-deep-dive (DI, AOP, Bean — 의존성 역전의 기술적 구현)
- ddd-deep-dive (Domain Model, Aggregate, Repository — 아키텍처의 내용물)
- unit-testing (아키텍처 결정이 테스트 용이성에 미치는 영향)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 아키텍처가 존재하는 이유 (5개 문서)
- 아키텍처의 본질 — 아키텍처는 폴더 구조가 아니라 변경 비용을 낮추는 결정의 집합, 좋은 아키텍처가 선택지를 열어두는 이유 (Uncle Bob: "아키텍처는 결정을 미루는 예술")
- 변경 비용과 의존성 — 의존성이 변경 전파의 경로인 이유, 의존하는 쪽이 의존받는 쪽 변경에 영향받는 원리, Ripple Effect(파급 효과)를 설계로 통제하는 방법
- SOLID 원칙과 아키텍처 — SRP(단일 책임)가 레이어 분리의 이유, OCP(개방-폐쇄)가 Adapter 패턴의 이유, DIP(의존성 역전)가 Clean/Hexagonal의 핵심인 이유
- 전통적 레이어드 아키텍처의 문제 — Controller → Service → Repository 구조가 직관적이지만 의존성이 DB 방향으로 흐르는 문제, 도메인이 영속성에 오염되는 이유, 테스트가 어려워지는 이유
- 아키텍처 선택 기준 — 도메인 복잡도/팀 규모/변경 빈도에 따른 적합한 아키텍처, 과잉 엔지니어링의 위험, 점진적 아키텍처 개선 전략

### Chapter 2: 레이어드 아키텍처 완전 분해 (5개 문서)
- 레이어드 아키텍처의 원칙 — Presentation / Business / Data Access 레이어의 책임, 레이어 간 의존성은 항상 아래 방향, 상위 레이어가 하위 레이어의 구체 클래스에 의존하지 않아야 하는 이유
- 엄격한 레이어 vs 느슨한 레이어 — Strict Layer(인접 레이어만 호출 가능) vs Relaxed Layer(모든 하위 레이어 호출 가능), 각각의 장단점, 실무에서의 타협점
- 레이어드 아키텍처의 함정 — Presentation이 DTO를 Domain까지 그대로 내려보내는 문제, Service 레이어가 비대해지는 Fat Service 문제, 레이어 간 순환 의존성이 생기는 경로
- 레이어드 아키텍처에서 테스트 — Spring Context 없이 Service를 테스트하기 어려운 이유, Mocking의 한계, 아키텍처 변경이 테스트에 미치는 영향
- 레이어드 아키텍처 개선 — DIP로 Data Access 레이어를 역전시키는 방법, Repository 인터페이스를 Business 레이어로 올리는 이유, 개선 전후 의존성 다이어그램 비교

### Chapter 3: Hexagonal Architecture (Ports & Adapters) (8개 문서)
- Hexagonal Architecture의 핵심 아이디어 — Alistair Cockburn의 원래 의도, 애플리케이션을 "내부(도메인)"와 "외부(인프라)"로 분리, 포트가 안과 밖의 계약(Contract)인 이유
- 주도하는 포트 vs 주도받는 포트 — Driving Port(사용자 → 애플리케이션, UseCase 인터페이스), Driven Port(애플리케이션 → 인프라, Repository/MessagePublisher 인터페이스), 왜 포트가 인터페이스인가
- 어댑터의 역할 — Driving Adapter(HTTP Controller, CLI, Test가 UseCase를 호출), Driven Adapter(JPA Repository, KafkaPublisher, InMemoryRepository가 Port를 구현), 어댑터 교체 가능성의 실제 의미
- 의존성 방향 완전 분석 — 모든 의존성이 도메인 방향(안)으로 향하는 이유, 인프라(JPA, Kafka, HTTP)가 도메인에 의존하는 이유, DIP가 의존성 방향을 역전시키는 방법
- Spring으로 Hexagonal 구현 — 패키지 구조(application/domain/infrastructure), UseCase 인터페이스 정의, @Service가 UseCase를 구현, @Repository가 Port를 구현, @Controller가 UseCase를 호출
- 테스트 용이성 향상 — Port 인터페이스 덕분에 InMemory Adapter로 빠른 단위 테스트, Spring Context 없이 도메인 로직 테스트, 어댑터별 독립 테스트
- Hexagonal의 실제 비용 — 인터페이스 증가로 인한 코드량, 간단한 CRUD에 적용 시 과잉 복잡도, 어댑터 레이어 추가로 인한 매핑 비용
- DDD와 Hexagonal 통합 — Aggregate가 도메인 내부에 위치, Repository Port가 Aggregate 경계를 지키는 방법, Domain Event가 Driven Port(MessagePublisher)를 통해 발행

### Chapter 4: Clean Architecture (6개 문서)
- Uncle Bob의 Clean Architecture — 4개 동심원(Entities/Use Cases/Interface Adapters/Frameworks & Drivers), 의존성 규칙(Dependency Rule), Hexagonal과의 공통점과 차이점
- Entities 레이어 — 비즈니스 규칙의 핵심, 어떤 외부 변경에도 영향받지 않아야 하는 이유, DDD Entity/Value Object/Aggregate와의 매핑
- Use Cases 레이어 — 애플리케이션별 비즈니스 규칙, 입출력 포트(Input/Output Boundary), Request Model / Response Model이 DTO와 다른 이유
- Interface Adapters 레이어 — Controller가 Request를 Use Case Input으로 변환, Presenter가 Use Case Output을 ViewModel로 변환, Gateway가 Repository를 구현
- Frameworks & Drivers 레이어 — JPA, Kafka, HTTP, Spring이 가장 바깥에 위치하는 이유, 프레임워크가 교체 가능해야 하는 이유(현실적으로는?)
- Clean Architecture 실전 — Spring Boot에서 모든 레이어를 엄격히 분리한 예시, 현실적 타협(Entity에 JPA 어노테이션 허용 여부), 팀 합의가 필요한 결정들

### Chapter 5: 아키텍처 패턴 비교와 선택 (5개 문서)
- Layered vs Hexagonal vs Clean 비교 — 의존성 방향, 테스트 용이성, 복잡도, 팀 학습 비용, 도메인 순수성 관점에서의 비교표
- 패턴 선택 기준 — 도메인 복잡도(단순 CRUD vs 복잡한 비즈니스), 팀 규모, 외부 시스템 교체 가능성 요구, 테스트 자동화 수준에 따른 선택 가이드
- 혼합 아키텍처 — Layered를 기본으로 하되 핵심 도메인에만 Hexagonal 적용, 점진적 리팩터링 경로, 일관성 유지 전략
- 아키텍처 결정 기록(ADR) — 아키텍처 결정을 문서화하는 방법, ADR 템플릿, 팀이 아키텍처 결정에 합의하는 프로세스
- MSA와 아키텍처 패턴 — 서비스 내부에 적용하는 아키텍처 패턴, 서비스 크기에 따른 아키텍처 선택, DDD Bounded Context와 아키텍처 패턴의 조합

### Chapter 6: 패키지 구조와 모듈화 (5개 문서)
- 패키지 구조의 원칙 — 레이어 기반 구조 vs 기능(Feature) 기반 구조 vs 컴포넌트 기반 구조, 각각의 장단점과 수정 용이성
- Spring Boot 실전 패키지 구조 — Hexagonal Architecture 기반 권장 패키지 구조, 모듈 간 의존성 강제 도구(ArchUnit), 잘못된 의존성 자동 감지
- 멀티 모듈 프로젝트 — Gradle 멀티 모듈로 레이어 간 의존성을 컴파일 시점에 강제, domain 모듈이 infrastructure 모듈을 참조할 수 없도록 설정
- ArchUnit으로 아키텍처 테스트 — 의존성 규칙을 테스트 코드로 강제, "도메인 레이어는 Spring 어노테이션을 사용하지 않는다" 규칙, CI에서 아키텍처 규칙 검증
- 코드 가이드라인 수립 — 팀이 아키텍처 원칙을 코드 리뷰에서 체크하는 기준, 신규 기능 추가 시 어느 레이어에 코드를 작성할지 결정 트리

### Chapter 7: 실전 프로젝트 — 아키텍처 리팩터링 (5개 문서)
- 레거시 코드 분석 — Controller에 비즈니스 로직, Service에 SQL, Entity에 비즈니스 로직 혼재, 의존성 그래프로 문제 시각화
- 점진적 리팩터링 전략 — Strangler Fig 패턴으로 레거시를 유지하면서 새 아키텍처 도입, 하나의 기능부터 Hexagonal로 전환, 테스트 커버리지 확보 후 리팩터링
- 도메인 레이어 분리 — Service에서 도메인 로직 추출 → Domain Object로, Entity에서 비즈니스 메서드 추가, UseCase 인터페이스 도입
- 인프라 레이어 분리 — Repository 인터페이스를 도메인으로, JPA Repository를 Adapter로 이동, Kafka 발행을 Port로 추상화
- 전후 비교 — 리팩터링 전후 의존성 다이어그램, 테스트 실행 속도 비교(Spring Context 필요 vs 불필요), 새 기능 추가 시 변경 범위 비교

---

총 **39개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 아키텍처 결정이 중요한가
## 😱 흔한 실수 (Before — 아키텍처 원칙 없이 짜는 코드)
## ✨ 올바른 접근 (After — 아키텍처 원칙 적용)
## 🔬 내부 원리 (왜 이 구조가 변경 비용을 낮추는가 — 의존성 분석)
## 💻 실전 코드 (Spring Boot 기반, 패키지 구조 포함)
## 📊 패턴 비교 (Layered vs Hexagonal vs Clean, 아키텍처별 의존성 다이어그램)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 모든 개념은 의존성 다이어그램으로 시각화 — 화살표 방향이 핵심
2. Before/After 패키지 구조와 코드 항상 비교
3. ArchUnit 테스트 코드 항상 포함 — "이 아키텍처 규칙을 어떻게 강제하는가"
4. ddd-deep-dive와 연결 — DDD 개념이 각 레이어의 어디에 위치하는가
5. 테스트 관점 항상 포함 — "이 아키텍처로 테스트가 얼마나 빨라지는가"

**실험 환경**:
```yaml
# docker-compose.yml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: architecture_demo
    ports:
      - "3306:3306"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    ports:
      - "9092:9092"

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
```

```java
// 핵심 분석 대상

// ❌ Before: 레이어드 아키텍처, 의존성이 위에서 아래로
@RestController
class OrderController {
    @Autowired OrderService service;  // Service에 의존
}
@Service
class OrderService {
    @Autowired OrderJpaRepository repo;  // JPA에 직접 의존 (인프라!)
    @Autowired KafkaTemplate kafka;      // Kafka에 직접 의존 (인프라!)
}
// 도메인이 인프라를 알고 있다 → 도메인 테스트에 DB, Kafka 필요

// ✅ After: Hexagonal Architecture, 의존성이 안으로만
// === Domain Layer (아무것도 모름) ===
public class Order { /* 순수 Java, 어노테이션 없음 */ }
public interface OrderRepository { /* Port — 인터페이스만 */ }
public interface OrderEventPublisher { /* Port — 인터페이스만 */ }

// === Application Layer ===
public interface PlaceOrderUseCase {  // Input Port
    OrderId placeOrder(PlaceOrderCommand command);
}
@Service
class PlaceOrderService implements PlaceOrderUseCase {
    private final OrderRepository repo;     // Port에만 의존
    private final OrderEventPublisher pub;  // Port에만 의존
    // JPA, Kafka를 전혀 모름!
}

// === Infrastructure Layer ===
@Repository
class JpaOrderRepository implements OrderRepository { // Adapter
    private final SpringDataOrderJpaRepository jpa;
}
@Component
class KafkaOrderEventPublisher implements OrderEventPublisher { // Adapter
    private final KafkaTemplate kafka;
}

// 테스트:
class PlaceOrderServiceTest {
    OrderRepository repo = new InMemoryOrderRepository();  // Adapter 교체!
    OrderEventPublisher pub = new InMemoryOrderEventPublisher();
    PlaceOrderService service = new PlaceOrderService(repo, pub);
    // Spring Context 없이 빠른 단위 테스트!
}
```

---

## 💡 핵심 분석 대상

```
의존성 방향 비교:

=== 레이어드 아키텍처 ===
Controller → Service → JpaRepository → DB
                 ↓
             KafkaTemplate → Kafka

문제: 도메인(Service)이 인프라(JPA, Kafka)를 안다
      JPA 변경 → Service 변경 필요
      단위 테스트에 DB, Kafka 필요

=== Hexagonal Architecture ===
               ↓ 의존
Controller → PlaceOrderUseCase(Interface) ← PlaceOrderService
                                               ↓ 의존
                               OrderRepository(Interface)
                                    ↑ 구현
                               JpaOrderRepository → DB

핵심: 모든 화살표가 도메인(중앙)을 향함
      JPA 변경 → JpaOrderRepository만 변경
      도메인 테스트에 DB 불필요 (InMemory Adapter 사용)

=== Clean Architecture 의존성 규칙 ===
Frameworks & Drivers  (Spring, JPA, Kafka)
      ↓ 의존
Interface Adapters    (Controller, JpaRepository, KafkaPublisher)
      ↓ 의존
Use Cases             (PlaceOrderUseCase, FindOrderQuery)
      ↓ 의존
Entities              (Order, OrderLine, Money)

규칙: 안쪽 레이어는 바깥쪽 레이어를 모른다
      Entities는 Use Cases를 모른다
      Use Cases는 Controller를 모른다
      Entities는 JPA를 모른다

ArchUnit으로 규칙 강제:
  @Test
  void domainShouldNotDependOnInfrastructure() {
      noClasses()
          .that().resideInAPackage("..domain..")
          .should().dependOnClassesThat()
          .resideInAPackage("..infrastructure..")
          .check(importedClasses);
  }

  @Test
  void domainShouldNotUseSpringAnnotations() {
      noClasses()
          .that().resideInAPackage("..domain..")
          .should().beAnnotatedWith(Component.class)
          .orShould().beAnnotatedWith(Service.class)
          .check(importedClasses);
  }
```

---

## 📚 참고 자료

- Clean Architecture: A Craftsman's Guide to Software Structure and Design (Robert C. Martin)
- Get Your Hands Dirty on Clean Architecture (Tom Hombergs) — Spring 실전 구현 최고 참고서
- Hexagonal Architecture 원문 — https://alistair.cockburn.us/hexagonal-architecture/
- Martin Fowler — PresentationDomainDataLayering — https://martinfowler.com/bliki/PresentationDomainDataLayering.html
- ArchUnit — https://www.archunit.org/
- Making Architecture Matter (Martin Fowler) — https://youtu.be/DngAZyWMGR0

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~8개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (39개 목표)
- Docker Compose 실험 환경
- ddd-deep-dive / unit-testing / spring-core-deep-dive와 연결되는 지점 명시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
