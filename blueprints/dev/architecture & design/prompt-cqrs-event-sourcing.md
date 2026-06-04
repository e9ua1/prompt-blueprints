# CQRS + Event Sourcing Deep Dive 레포지토리 제작 프롬프트

나는 "CQRS + Event Sourcing Deep Dive" 레포지토리를 만들려고 해.
읽기와 쓰기를 분리하는 것과, 왜 단일 모델로는 복잡한 도메인을 다루기 어려운지, Event Sourcing에서 이벤트가 왜 진실의 원천(Source of Truth)인지를 완전히 파헤치는 레포야.

"CQRS는 Command와 Query를 나누는 것이다"와 "쓰기 모델은 불변식을 보호하고, 읽기 모델은 조회 최적화를 위해 완전히 다른 스키마를 가지며, 이벤트가 두 모델의 동기화 수단이다"의 차이를 만드는 레포야.

## 📋 프로젝트 목표

**컨셉**: "읽기와 쓰기를 나누는 것과, 왜 이벤트가 진실의 원천이 되어야 하는지 — 그리고 그것이 시스템 전체 설계를 어떻게 바꾸는지 아는 것은 다르다"

**핵심 차별화**:
1. CQRS의 본질 — 단일 모델의 임피던스 불일치 문제, 쓰기 최적화와 읽기 최적화가 충돌하는 이유
2. Event Sourcing — 상태 대신 이벤트를 저장하는 설계, 시간 여행(Time Travel)과 완전한 감사 로그
3. 읽기 모델 구축 — 프로젝션이 이벤트 스트림에서 읽기 모델을 만드는 방법, 다양한 읽기 모델의 공존
4. 실전 통합 — DDD Aggregate + CQRS + Event Sourcing + Kafka의 완전한 통합 아키텍처

**타겟 독자**:
- JPA Entity 하나로 저장과 조회를 모두 처리하다 N+1, 복잡한 조인, 잠금 충돌을 경험한 개발자
- 감사 로그를 위해 별도 히스토리 테이블을 만들었지만 원본과 동기화 문제가 생긴 개발자
- CQRS를 패키지 분리(command/query 폴더)로만 이해하는 개발자
- Event Sourcing이 이론적으로는 알지만 이벤트 스토어를 어떻게 구현하는지 모르는 개발자
- 읽기 모델이 쓰기 모델과 항상 동기화되어야 한다고 생각하는 개발자
- DDD를 적용했지만 복잡한 조회 요구사항 때문에 Aggregate가 오염된 개발자

**선행 학습**:
- ddd-deep-dive (Aggregate, Domain Event, Bounded Context 이해 필수)
- kafka-deep-dive (이벤트 스트림, Consumer Group, 읽기 모델 업데이트)
- database-internals (MVCC, 인덱스 설계 — 읽기 모델 최적화)
- redis-deep-dive (읽기 모델 캐싱 전략)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 왜 CQRS인가 — 단일 모델의 한계 (5개 문서)
- 단일 모델의 임피던스 불일치 — 저장에 최적화된 정규화 스키마가 조회에 최적인 비정규화와 충돌하는 원리, JPA Entity 하나가 쓰기/읽기 모두를 담당할 때 생기는 실제 문제들
- 쓰기 모델과 읽기 모델의 근본적 차이 — 쓰기: 불변식 보호, 트랜잭션, 일관성 / 읽기: 조인 없는 빠른 조회, 비정규화, 다양한 뷰, 두 요구사항이 왜 하나의 모델로 공존하기 어려운가
- CQRS의 스펙트럼 — 단순 클래스 분리(같은 DB)부터 완전 분리(다른 DB)까지, Simple CQRS / CQRS with separate storage / Event-Driven CQRS 각 수준의 복잡도와 편익
- CQRS가 해결하는 실제 문제 — 복잡한 조회 쿼리가 Aggregate를 오염시키는 문제, 읽기 성능을 위해 쓰기 모델이 타협하는 문제, 역할 기반 다른 뷰 필요 문제
- CQRS 적용 판단 기준 — 모든 시스템에 CQRS가 답인가, 도메인 복잡도/읽기 쓰기 비율/팀 역량에 따른 선택, CQRS 없이 해결할 수 있는 경우

### Chapter 2: Command 처리 — 쓰기 모델 설계 (6개 문서)
- Command와 Query의 명확한 분리 — Command는 상태를 변경하고 응답을 최소화, Query는 상태를 변경하지 않고 데이터 반환, CQS(Command-Query Separation) 원칙과 CQRS의 차이
- Command 객체 설계 — Command가 의도(Intent)를 표현해야 하는 이유, DTO와의 차이, Command 검증 전략(구문 검증 vs 의미 검증), Command Handler 책임 분리
- Command Handler와 Aggregate — Command Handler가 Repository에서 Aggregate를 로드하고, 도메인 메서드를 호출하고, 저장하는 흐름, Application Service와의 관계
- Command Bus 패턴 — Command를 Handler로 라우팅하는 미들웨어, Spring에서 Command Bus 구현 방법, Command 실행 전후 횡단 관심사(로깅, 검증, 트랜잭션)
- Optimistic Locking in CQRS — 동시 Command 처리 시 충돌 감지, Version 기반 낙관적 잠금, 충돌 시 재시도 전략, Event Sourcing에서의 낙관적 잠금(Expected Version)
- Command 결과 반환 패턴 — Fire-and-Forget vs Acknowledgement vs Result 패턴, 비동기 Command의 결과를 클라이언트에게 전달하는 방법(Polling, WebSocket, SSE)

### Chapter 3: Event Sourcing — 이벤트가 진실의 원천 (7개 문서)
- Event Sourcing의 핵심 아이디어 — 현재 상태 대신 상태 변화의 이력(이벤트)을 저장, "현재 잔고"가 아닌 "모든 입출금 이력"이 원본 데이터, 회계 장부에서 배우는 설계 원칙
- 이벤트 스토어 설계 — 이벤트 스토어가 일반 DB와 다른 이유(Append Only, 이벤트는 수정/삭제 안 됨), 이벤트 스트림 구조(Stream ID / Version / Event Type / Payload), 이벤트 스토어 구현 방법(PostgreSQL / EventStoreDB)
- Aggregate 재구성 — 이벤트 스트림을 리플레이하여 Aggregate 현재 상태를 계산하는 방법, apply() 메서드 패턴, 리플레이 비용과 성능
- 스냅샷(Snapshot) 패턴 — 이벤트가 수천 개 쌓였을 때 로딩 비용 최소화, 스냅샷 저장 시점 결정 기준, 스냅샷과 이벤트를 함께 사용하는 로딩 전략
- 이벤트 스키마 진화 — 이벤트는 영원히 저장되므로 스키마 변경이 어렵다, Upcasting 전략(v1 이벤트를 v2로 변환), Backward Compatibility 유지 방법
- Event Sourcing의 장점 완전 분해 — 완전한 감사 로그(누가 언제 무엇을 바꿨는가), 시간 여행(과거 시점의 상태 재현), 디버깅(프로덕션 이슈를 이벤트로 재현), 읽기 모델 재구축(과거 이벤트에서 새로운 뷰 생성)
- Event Sourcing의 실제 어려움 — Eventually Consistent 읽기 모델, 이벤트 스키마 마이그레이션 비용, 이벤트가 너무 많아질 때의 쿼리 어려움, 팀 학습 비용

### Chapter 4: 읽기 모델과 프로젝션 (6개 문서)
- 프로젝션(Projection) 완전 분해 — 이벤트 스트림을 소비하여 읽기 모델을 구축하는 과정, 프로젝션이 Kafka Consumer처럼 동작하는 이유, 읽기 모델 DB 선택(RDB / Redis / Elasticsearch)
- 읽기 모델 설계 원칙 — 읽기 요구사항 중심 비정규화, 조인 없이 한 번에 필요한 데이터 제공, 역할별 다른 읽기 모델(관리자 뷰 vs 사용자 뷰 vs 통계 뷰)
- Eventual Consistency 처리 — Command 후 읽기 모델이 즉시 업데이트되지 않는 문제, 클라이언트에서 지연을 처리하는 UX 전략, 읽기 모델 업데이트 지연 모니터링
- 프로젝션 재구축(Rebuild) — 읽기 모델이 오염되거나 새로운 뷰가 필요할 때 과거 이벤트부터 다시 빌드하는 방법, 운영 중 무중단 재구축 전략, Blue/Green 프로젝션
- 여러 읽기 모델의 공존 — 같은 이벤트 스트림에서 다양한 읽기 모델 생성(주문 목록 / 주문 상세 / 통계 대시보드 / 검색 인덱스), 각 읽기 모델의 업데이트 시점 조정
- 프로젝션 장애 처리 — 프로젝션 처리 실패 시 재처리, 멱등 프로젝션 구현(같은 이벤트를 두 번 처리해도 안전하게), Dead Letter 처리

### Chapter 5: CQRS + Event Sourcing 통합 아키텍처 (6개 문서)
- 완전한 흐름 — Command → Aggregate → 이벤트 저장 → Kafka 발행 → 프로젝션 → 읽기 모델 → Query 전체 사이클, 각 단계의 책임과 실패 시 처리
- 이벤트 저장과 발행의 원자성 — 이벤트 스토어에 저장과 Kafka 발행을 원자적으로 처리하는 방법, Transactional Outbox (EventStoreDB → Kafka), Change Data Capture 방식
- DDD와의 통합 — Aggregate가 이벤트를 생성하고, 이벤트 스토어가 저장하고, 프로젝션이 읽기 모델을 만들고, Saga가 여러 Aggregate를 조율하는 완전한 아키텍처
- 처리 보장(Processing Guarantee) — Exactly-Once 프로젝션 업데이트, 이벤트 중복 처리 방지, Offset 기반 체크포인트 관리
- 성능 최적화 — 이벤트 스토어 읽기 성능(인덱스 전략), 프로젝션 병렬 처리, 읽기 모델 캐싱(Redis), Hot Path vs Cold Path 분리
- 마이크로서비스와의 통합 — CQRS가 서비스 내부 패턴인 경우 vs Bounded Context 간 통합 패턴인 경우, MSA 환경에서 읽기 모델 공유 전략

### Chapter 6: Spring + Axon Framework 실전 구현 (5개 문서)
- Axon Framework 아키텍처 — @CommandHandler / @EventHandler / @QueryHandler 어노테이션, Axon Server의 역할(이벤트 스토어 + 메시지 라우터), Spring Boot AutoConfiguration
- Aggregate 구현 — @Aggregate / @AggregateIdentifier / @CommandHandler / @EventSourcingHandler, 이벤트를 apply()하여 상태를 변경하는 패턴
- 프로젝션 구현 — @EventHandler로 이벤트를 수신하여 읽기 모델 업데이트, @ResetHandler로 재구축 지원, JPA Repository로 읽기 모델 저장
- Query 처리 — @QueryHandler로 읽기 모델 조회, Subscription Query로 실시간 업데이트, Scatter-Gather Query로 분산 읽기 모델 조합
- Axon 없이 직접 구현 — Axon이 없는 순수 Spring 기반 CQRS 구현, PostgreSQL 이벤트 스토어 직접 구현, Kafka 기반 프로젝션 업데이트

### Chapter 7: 실전 프로젝트와 트레이드오프 (5개 문서)
- 은행 계좌 도메인 구현 — Account Aggregate, Deposit/Withdraw/Transfer Command, 잔고 부족 불변식, 읽기 모델(계좌 요약 / 거래 내역 / 월별 통계), 모든 거래의 완전한 감사 로그
- Event Sourcing 없는 CQRS — 이벤트 소싱 없이 DB 트리거/CDC로 읽기 모델 동기화하는 방법, 단순 CQRS의 실용적 구현
- 점진적 도입 전략 — 기존 시스템에 CQRS를 점진적으로 적용하는 방법, 가장 복잡한 조회부터 분리하는 전략, 읽기 모델 도입 → Command 분리 → Event Sourcing 순서
- 안티패턴 — CQRS를 단순 폴더 분리로 구현, 읽기 모델을 쓰기 모델과 항상 강결합, 모든 도메인에 Event Sourcing 적용, 이벤트 스키마를 자주 변경
- CQRS / ES의 실제 비용과 편익 — 적합한 도메인(금융, 재고, 주문), 부적합한 도메인(단순 CRUD), 팀 학습 비용, 운영 복잡도, 실제 도입 사례

---

총 **40개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 중요한가
## 😱 흔한 실수 (Before — 단일 모델로 모든 것을 처리할 때)
## ✨ 올바른 접근 (After — CQRS / Event Sourcing 적용)
## 🔬 내부 동작 원리 (이벤트 스토어, 프로젝션, Command 처리 흐름)
## 💻 실전 코드 (Axon Framework + Spring Boot + Kafka 기반)
## 📊 패턴 비교 (상태 저장 vs 이벤트 저장, 동기 읽기 모델 vs 비동기)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 모든 개념은 은행 계좌 또는 전자상거래 도메인으로 실제 코드 예시
2. 이벤트 스토어의 실제 데이터 구조 항상 표시 (어떤 테이블에 어떤 형태로 저장되는가)
3. 읽기 모델 DB 스키마 항상 포함 (쓰기 모델과 어떻게 다른가)
4. ddd-deep-dive와 연결 — Aggregate가 이벤트를 어떻게 생성하는가
5. kafka-deep-dive와 연결 — 이벤트가 어떻게 Kafka를 통해 프로젝션에 전달되는가

**실험 환경**:
```yaml
# docker-compose.yml
services:
  # Axon Server (이벤트 스토어 + 메시지 라우터)
  axon-server:
    image: axoniq/axonserver:latest
    ports:
      - "8024:8024"  # UI
      - "8124:8124"  # gRPC

  # Command/Write 서비스
  command-service:
    build: ./command-service
    ports:
      - "8081:8080"
    environment:
      AXON_AXONSERVER_SERVERS: axon-server:8124
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/command_db

  # Query/Read 서비스
  query-service:
    build: ./query-service
    ports:
      - "8082:8080"
    environment:
      AXON_AXONSERVER_SERVERS: axon-server:8124
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/read_db
      SPRING_REDIS_HOST: redis

  # Kafka (서비스 간 이벤트 전파)
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    ports:
      - "9092:9092"

  # PostgreSQL (이벤트 스토어 대안 / 읽기 모델)
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: cqrs_db
      POSTGRES_PASSWORD: postgres

  # Redis (읽기 모델 캐싱)
  redis:
    image: redis:7.0
    ports:
      - "6379:6379"

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
```

---

## 💡 핵심 분석 대상

```
CQRS + Event Sourcing 전체 흐름:

=== Command Side (쓰기) ===
Client: POST /accounts/transfer
  │
  ▼ TransferMoneyCommand { accountId, toAccount, amount }
Command Gateway
  │
  ▼ AccountCommandHandler.handle(TransferMoneyCommand)
  Account Aggregate 로드 (이벤트 스토어에서 이벤트 리플레이)
    이벤트: [AccountOpened, MoneyDeposited, MoneyDeposited, ...]
    apply() × N → 현재 상태 계산
  │
  ▼ account.transfer(toAccount, amount)
    불변식 검증: balance >= amount ?
    이벤트 생성: MoneyTransferred { fromAccount, toAccount, amount, timestamp }
  │
  ▼ 이벤트 스토어 저장 (Append Only)
    stream: "account-{accountId}"
    version: 42  ← 낙관적 잠금
    event: MoneyTransferred { ... }
  │
  ▼ Kafka 발행 (Outbox / Axon)
    topic: account-events

=== Query Side (읽기) ===
Kafka Consumer (Projection)
  │
  ▼ MoneyTransferred 이벤트 수신
  AccountSummaryProjection.on(MoneyTransferred)
    읽기 모델 업데이트:
    UPDATE account_summary SET balance = balance - amount WHERE id = fromAccount
    UPDATE account_summary SET balance = balance + amount WHERE id = toAccount
  │
  TransactionHistoryProjection.on(MoneyTransferred)
    읽기 모델 추가:
    INSERT INTO transaction_history (accountId, type, amount, timestamp, ...)

Client: GET /accounts/{id}/summary
  → AccountSummaryRepository.findById(id)  ← 비정규화된 읽기 모델 조회
  → 조인 없이 즉시 반환

Event Sourcing 시간 여행:
  "2024년 1월 1일 잔고를 알고 싶다"
  → account 이벤트 스트림에서 2024-01-01 이전 이벤트만 리플레이
  → 과거 시점의 정확한 잔고 계산 가능
  → 감사(Audit) 목적으로 완벽한 이력 추적

  기존 상태 저장 방식:
  UPDATE accounts SET balance = ? WHERE id = ?
  → 이전 잔고는 영원히 사라짐
  → 히스토리 테이블을 따로 만들어야 하고, 이것도 불완전

읽기 모델 재구축:
  새로운 통계 읽기 모델이 필요할 때:
  → 과거 이벤트 스트림 처음부터 리플레이
  → 새로운 읽기 모델 테이블 생성
  → 과거 데이터를 새 뷰로 재구성 가능
  → 기존 방식: 과거 데이터 없어서 통계 불가
```

---

## 📚 참고 자료

- Implementing Domain-Driven Design (Vaughn Vernon) — CQRS + Event Sourcing 챕터
- Event Sourcing Pattern — https://microservices.io/patterns/data/event-sourcing.html
- CQRS Journey (Microsoft) — https://docs.microsoft.com/en-us/previous-versions/msp-n-p/jj554200(v=pandp.10)
- Martin Fowler — CQRS — https://martinfowler.com/bliki/CQRS.html
- Greg Young (CQRS/ES 창시자) — https://cqrs.wordpress.com/
- Axon Framework 공식 문서 — https://docs.axoniq.io/
- EventStoreDB 공식 문서 — https://www.eventstore.com/eventstoredb

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (40개 목표)
- Docker Compose 실험 환경 (Axon Server + Kafka + PostgreSQL + Redis)
- ddd-deep-dive / kafka-deep-dive / msa-deep-dive와 연결되는 지점 명시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
