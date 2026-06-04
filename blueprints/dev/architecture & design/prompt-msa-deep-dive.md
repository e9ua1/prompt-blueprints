# MSA Deep Dive 레포지토리 제작 프롬프트

나는 "MSA Deep Dive" 레포지토리를 만들려고 해.
서비스를 분리하는 것과, 왜 Database per Service를 써야 하는지, Saga로 분산 트랜잭션을 어떻게 대체하는지, API Gateway가 내부적으로 어떻게 동작하는지를 완전히 파헤치는 레포야.

"서비스를 작게 나눴더니 분산된 모놀리스가 됐다"와 "Bounded Context 기반으로 서비스를 나누고, 동기 호출을 최소화하며, Saga로 데이터 일관성을 보장하는 것"의 차이를 만드는 레포야.

## 📋 프로젝트 목표

**컨셉**: "서비스를 쪼개는 것과, 왜 그 경계에서 쪼개야 하는지 — 그리고 분리 이후 데이터 일관성을 어떻게 보장하는지 아는 것은 다르다"

**핵심 차별화**:
1. 서비스 분해 전략 — Bounded Context 기반 분해, 잘못된 분해가 만드는 분산 모놀리스
2. 통신 패턴 — 동기(REST/gRPC) vs 비동기(Event) 선택 원칙, 결합도와 가용성 트레이드오프
3. 데이터 관리 — Database per Service 원칙, Saga로 분산 트랜잭션 없이 일관성 달성
4. 가용성 설계 — Circuit Breaker, Bulkhead, Retry가 어떻게 장애 전파를 막는가

**타겟 독자**:
- MSA로 전환했지만 서비스 간 직접 DB 접근이 남아있는 개발자
- 서비스가 다운되면 전체 시스템이 멈추는 동기 호출 체인을 설계한 개발자
- 분산 트랜잭션이 필요할 것 같아서 MSA를 못하겠다는 개발자
- API Gateway를 단순 Reverse Proxy로만 쓰는 개발자
- Spring Cloud를 쓰지만 각 컴포넌트가 왜 있는지 모르는 개발자
- 컨테이너와 Kubernetes를 쓰지만 MSA 설계 원칙을 모르는 개발자

**선행 학습**:
- ddd-deep-dive (Bounded Context 기반 서비스 분해 이해 필수)
- kafka-deep-dive (Saga Choreography, Outbox Pattern 구현)
- spring-cloud-deep-dive (Eureka, Gateway, Circuit Breaker 컴포넌트 이해)
- network-deep-dive (REST/gRPC 통신 원리, TLS mTLS 이해)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: MSA 설계 철학과 분해 전략 (6개 문서)
- MSA가 해결하는 문제 — 모놀리스의 실제 문제(독립 배포 불가, 기술 스택 종속, 팀 결합), MSA가 공짜가 아닌 이유(운영 복잡도, 분산 시스템 문제 전부 감수)
- 서비스 분해 원칙 — Bounded Context 기반 분해(DDD 연결), 단일 책임 원칙(SRP), 비즈니스 능력(Business Capability) 기반 분해, 서비스 크기 기준 ("팀이 독립적으로 배포할 수 있는가")
- 분산 모놀리스 안티패턴 — 서비스를 쪼갰지만 강결합이 남은 이유, Shared Database 안티패턴이 왜 최악인가, 동기 호출 체인이 가용성을 기하급수적으로 낮추는 수학적 증명
- 서비스 분해 실전 — 전자상거래 도메인의 서비스 분해 예시, 어떤 기준으로 주문/배송/결제/상품을 나누는가, 서비스 경계 재조정 전략(분리 이후 합치기, 합친 것 다시 나누기)
- MSA 도입 판단 기준 — Conway's Law(조직 구조가 시스템 구조를 결정한다), 팀 규모별 권장 아키텍처, 모놀리스 우선 전략과 Strangler Fig 패턴
- 서비스 자율성 원칙 — 독립 배포 가능성, 독립 확장 가능성, 기술 스택 자율성, 데이터 자율성의 4가지 기준

### Chapter 2: 서비스 간 통신 패턴 (7개 문서)
- 동기 통신 vs 비동기 통신 — 선택 기준(즉시 응답 필요 여부, 결합도 허용 여부), 동기 통신이 가용성을 낮추는 원리(0.99 * 0.99 * 0.99 = 0.97), 비동기의 Eventual Consistency 비용
- REST API 설계 — 서비스 간 REST API 설계 원칙, API 버저닝 전략(URL 버저닝 vs Header 버저닝), Consumer-Driven Contract Testing으로 Breaking Change 방지
- gRPC 완전 분해 — Protocol Buffers 직렬화가 JSON보다 빠른 이유, HTTP/2 멀티플렉싱으로 연결 효율화, 서버 스트리밍/양방향 스트리밍, gRPC vs REST 선택 기준
- 이벤트 기반 통신 — 이벤트 발행/구독이 결합도를 낮추는 원리, 이벤트 스키마 진화(Backward/Forward Compatibility), Kafka Topic 설계 원칙(서비스당 토픽 vs 이벤트 유형당 토픽)
- API Composition 패턴 — 여러 서비스의 데이터를 조합하는 방법, API Gateway에서 조합하는 방법, BFF(Backend for Frontend) 패턴
- Service Mesh — Envoy Sidecar가 트래픽을 가로채는 원리, mTLS로 서비스 간 암호화, Istio Circuit Breaker가 Hystrix와 다른 점, 언제 Service Mesh가 필요한가
- GraphQL Federation — 여러 서비스의 스키마를 합치는 방법, Supergraph와 Subgraph, REST vs GraphQL 선택 기준

### Chapter 3: 데이터 관리 — Database per Service (6개 문서)
- Database per Service 원칙 — 왜 서비스마다 DB를 분리해야 하는가, Shared Database의 실제 문제(스키마 변경이 다른 서비스를 깨뜨리는 시나리오), 분리 시 발생하는 조인 문제 해결 방법
- 서비스별 DB 기술 선택 — 폴리글랏 영속성(Polyglot Persistence), 주문 서비스 MySQL + 상품 검색 Elasticsearch + 세션 Redis를 함께 쓰는 설계, 기술 선택 기준
- Join 없는 데이터 조회 전략 — API Composition(서비스 조합), CQRS 읽기 모델(다음 레포 연결), 데이터 복제 허용 설계, 각 전략의 일관성 트레이드오프
- 분산 데이터 일관성 — ACID vs BASE, Eventual Consistency를 애플리케이션 레벨에서 보장하는 방법, 일관성 수준 선택 기준(강한 일관성이 필요한 경우)
- 데이터 이관 전략 — 모놀리스 DB에서 서비스별 DB로 점진적 분리하는 방법, Strangler Fig 데이터 이관, 이관 중 데이터 동기화 전략
- 서비스 간 외래키 — 물리적 외래키 대신 애플리케이션 레벨 일관성, 참조 무결성을 이벤트로 보장하는 방법, 고아 데이터 처리 전략

### Chapter 4: Saga — 분산 트랜잭션 없이 일관성 달성 (6개 문서)
- 분산 트랜잭션의 문제 — 2PC(Two-Phase Commit)가 왜 MSA에서 쓸 수 없는가, 가용성과 일관성의 트레이드오프(CAP Theorem), Saga가 2PC를 대체하는 원리
- Choreography Saga — 각 서비스가 이벤트에 반응하여 다음 단계를 진행하는 방법, 주문-결제-재고-배송 Choreography 흐름, 실패 시 보상 트랜잭션 체인
- Orchestration Saga — Saga Orchestrator가 중앙에서 단계를 지시하는 방법, 상태 머신으로 Saga 상태 관리, Orchestrator 장애 시 복구 전략
- Choreography vs Orchestration 선택 — 각각의 결합도/복잡도/디버깅 난이도 비교, 서비스 수와 단계 수에 따른 선택 기준, 실무 권장 패턴
- 보상 트랜잭션(Compensating Transaction) 설계 — 모든 단계가 취소 가능해야 하는 이유, 취소 불가능한 단계 처리(Pivot Transaction), 멱등 보상 트랜잭션 구현
- Saga 실패 처리와 모니터링 — Saga 진행 상태 추적, 중간 실패 시 재시작 전략, Dead Saga 감지와 수동 개입 절차

### Chapter 5: 가용성 패턴 — 장애 격리와 복원력 (6개 문서)
- Circuit Breaker 완전 분해 — Closed/Open/Half-Open 상태 전이, 실패율 임계값 계산, Resilience4j Circuit Breaker 내부 슬라이딩 윈도우, Spring Cloud CircuitBreaker 통합
- Bulkhead 패턴 — 스레드 풀 격리로 하나의 서비스 장애가 전체를 멈추지 않도록, Semaphore 기반 vs Thread Pool 기반 Bulkhead, Resilience4j 구현
- Retry와 Exponential Backoff — 재시도가 장애를 악화시키는 Thundering Herd 문제, Jitter 추가로 재시도 분산, 멱등성 보장 없이 재시도할 때의 위험
- Timeout 전략 — 적절한 Timeout 설정 방법, 타임아웃 없이 서비스 호출하면 생기는 자원 고갈, Timeout + Retry + Circuit Breaker 조합 전략
- Fallback 전략 — 서비스 장애 시 기본값 반환, 캐시된 데이터 반환, 기능 축소 운영(Graceful Degradation), Fallback 체인 설계
- 헬스체크와 자가 치유 — Spring Boot Actuator 헬스체크, Kubernetes Liveness/Readiness Probe 차이, 서비스 자동 복구 시나리오

### Chapter 6: API Gateway와 서비스 디스커버리 (5개 문서)
- API Gateway 역할 완전 분해 — 단순 리버스 프록시가 아닌 이유, 인증/인가 오프로딩, Rate Limiting, 요청 변환, 라우팅 로직, Spring Cloud Gateway 필터 체인 내부
- 서비스 디스커버리 — 왜 하드코딩된 URL이 MSA에서 안 되는가, Client-Side Discovery(Eureka) vs Server-Side Discovery(Kubernetes Service), Ribbon 로드밸런서 동작
- Rate Limiting 구현 — Token Bucket / Sliding Window 알고리즘, Gateway에서 Redis로 분산 Rate Limiting, 사용자별/서비스별 한도 설정
- 인증/인가 아키텍처 — Gateway에서 JWT 검증, 각 서비스는 토큰 신뢰, OAuth2 + OIDC로 외부 인증, 서비스 간 인증(Service Account, mTLS)
- BFF(Backend for Frontend) 패턴 — 웹/모바일/서드파티 클라이언트별 최적화된 API 레이어, API 조합 책임, GraphQL을 BFF로 사용하는 방법

### Chapter 7: 관찰 가능성과 운영 (5개 문서)
- 분산 추적(Distributed Tracing) — Trace ID / Span ID로 요청이 여러 서비스를 거치는 흐름 추적, OpenTelemetry + Zipkin/Jaeger 구성, 느린 서비스 핀포인트 방법
- 중앙화 로깅 — 각 서비스 로그를 Elasticsearch로 수집하는 방법, Trace ID로 요청 전체 로그 조합, ELK Stack / EFK Stack 구성
- 메트릭 모니터링 — 서비스별 Prometheus 메트릭 수집, MSA의 핵심 지표(응답 시간, 에러율, 처리량, 포화도 — RED 방법론), Grafana 대시보드
- 배포 전략 — Blue/Green 배포로 다운타임 없이 배포, Canary 배포로 점진적 트래픽 이전, Feature Flag로 기능 ON/OFF, Kubernetes Rolling Update
- 운영 중 발생하는 문제 패턴 — 순환 의존성 감지, 카스케이드 장애 분석, 분산 데드락, Saga 실패 추적과 수동 개입

---

총 **41개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 MSA에서 중요한가
## 😱 흔한 실수 (Before — 잘못된 MSA 설계)
## ✨ 올바른 접근 (After — 올바른 MSA 패턴)
## 🔬 내부 동작 원리 (컴포넌트 내부 동작, 알고리즘 분석)
## 💻 실전 구현 (Spring Cloud + Kafka + Docker 기반)
## 📊 패턴 비교 (Choreography vs Orchestration, REST vs gRPC 등)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 분산 시스템의 실패 시나리오 항상 포함 — "이 서비스가 죽으면 무슨 일이 생기는가"
2. 가용성 계산 항상 포함 — 동기 호출 3단계면 0.99^3 = 97% 가용성
3. Spring Cloud 컴포넌트와 연결 — spring-cloud-deep-dive 레포 참조 명시
4. Kubernetes 환경 고려 — Service Mesh, Ingress, Health Check
5. 모놀리스에서 MSA로의 점진적 전환 경로 포함

**실험 환경**:
```yaml
# docker-compose.yml
version: '3.8'
services:
  # 주문 서비스
  order-service:
    build: ./order-service
    ports:
      - "8081:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql-order:3306/order_db
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: http://eureka:8761/eureka

  # 결제 서비스
  payment-service:
    build: ./payment-service
    ports:
      - "8082:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql-payment:3306/payment_db
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092

  # API Gateway
  api-gateway:
    image: springcloud/spring-cloud-gateway:latest
    ports:
      - "8080:8080"

  # Eureka Server
  eureka:
    image: springcloud/eureka:latest
    ports:
      - "8761:8761"

  # 주문 DB
  mysql-order:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: order_db
      MYSQL_ROOT_PASSWORD: root

  # 결제 DB (Database per Service!)
  mysql-payment:
    image: mysql:8.0
    environment:
      MYSQL_DATABASE: payment_db
      MYSQL_ROOT_PASSWORD: root

  # Kafka (Saga 이벤트)
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on: [zookeeper]
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  # 분산 추적
  zipkin:
    image: openzipkin/zipkin:latest
    ports:
      - "9411:9411"

  # Prometheus + Grafana
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
```

---

## 💡 핵심 분석 대상

```
주문 처리 전체 MSA 흐름:

클라이언트
  │
  ▼ POST /api/orders
API Gateway (8080)
  ├── JWT 검증 (인증 오프로딩)
  ├── Rate Limiting 확인 (Redis 기반)
  └── 라우팅 → Order Service (8081)
        │
        ▼ OrderService.placeOrder()
        Order DB에 저장 (MySQL - order_db)
        Outbox 테이블에 OrderPlaced 이벤트 저장
        트랜잭션 커밋
        │
        ▼ Outbox Poller → Kafka 발행
        Topic: order-events
        Event: { "type": "OrderPlaced", "orderId": "...", "amount": 10000 }
        │
        ├──────────────────────────────────┐
        ▼                                  ▼
Payment Service 수신               Inventory Service 수신
결제 처리                          재고 차감
PaymentCompleted 발행              StockReserved 발행
또는 PaymentFailed 발행            또는 StockInsufficient 발행
        │
        ▼
Order Service 수신
PaymentCompleted → 주문 확정
PaymentFailed → 주문 취소 (보상 트랜잭션)

Choreography Saga 실패 시나리오:
  OrderPlaced → Payment 성공 → StockReserved 성공
  → Shipping 실패 → ShippingFailed 이벤트
  → Order Service: 보상 → PaymentRefundRequested
  → Payment Service: 환불 → StockReleaseRequested
  → Inventory Service: 재고 복원

Circuit Breaker 동작:
  Order → Payment 동기 호출 100번 중 60번 실패
  → Resilience4j: OPEN 상태 전환 (이후 호출 즉시 실패 반환)
  → 5초 후 HALF-OPEN: 3번 테스트 호출
  → 성공 → CLOSED 복귀
  → 실패 → OPEN 유지

가용성 계산:
  단일 서비스: 99.9% 가용성
  동기 3단계 호출: 0.999^3 = 99.7% 가용성
  비동기 이벤트: 이벤트 브로커 가용성에만 의존 → 독립적
```

---

## 📚 참고 자료

- Microservices Patterns (Chris Richardson) — Saga, CQRS, 분산 데이터 패턴의 바이블
- Building Microservices, 2nd Edition (Sam Newman)
- Designing Distributed Systems (Brendan Burns)
- Chris Richardson의 microservices.io — https://microservices.io/patterns/
- Martin Fowler — https://martinfowler.com/microservices/
- Netflix Tech Blog — https://netflixtechblog.com/

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (41개 목표)
- Docker Compose 멀티 서비스 실험 환경
- ddd-deep-dive / kafka-deep-dive / spring-cloud-deep-dive와 연결되는 지점 명시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
