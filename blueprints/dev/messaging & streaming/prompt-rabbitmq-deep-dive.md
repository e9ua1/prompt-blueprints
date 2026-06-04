# RabbitMQ Deep Dive 레포지토리 제작 프롬프트

나는 "RabbitMQ Deep Dive" 레포지토리를 만들려고 해.
메시지를 발행하고 소비하는 것과, AMQP의 Exchange/Binding/Queue 라우팅이 어떻게 동작하는지, Kafka와 근본적으로 어떤 철학이 다른지, Dead Letter Exchange가 왜 메시지를 유실 없이 처리하는지를 완전히 파헤치는 레포야.

"@RabbitListener 붙이면 메시지를 받겠지"와 "Direct/Topic/Fanout/Headers Exchange가 메시지를 어떻게 라우팅하는지, Acknowledgement 모드가 메시지 내구성에 어떤 영향을 미치는지, Prefetch Count가 왜 Consumer 처리량의 핵심인지 아는 것"의 차이를 만드는 레포야.

## 📋 프로젝트 목표

**컨셉**: "메시지를 보내는 것과, 어떤 Exchange로 어떤 Routing Key를 써서 어떤 Queue에 전달되는지 — 그리고 소비자가 실패했을 때 메시지가 어디로 가는지 아는 것은 다르다"

**핵심 차별화**:
1. AMQP 프로토콜 — Exchange/Binding/Queue의 3단계 라우팅이 Kafka의 Topic 직접 발행과 근본적으로 다른 이유
2. Kafka vs RabbitMQ — 언제 무엇을 선택해야 하는가, 메시지 소비 철학의 차이(Push vs Pull, 삭제 vs 보관)
3. 신뢰성 패턴 — Publisher Confirm, Consumer Ack, Dead Letter Exchange로 메시지 유실 없는 처리
4. 실무 패턴 — Work Queue, Pub/Sub, RPC, Priority Queue, Delayed Message 각각의 구현

**타겟 독자**:
- autoAck=true로 설정해놓고 Consumer 실패 시 메시지가 사라지는 문제를 경험한 개발자
- Exchange 없이 Queue에 직접 발행하고 있어서 라우팅 유연성이 없는 개발자
- Prefetch Count를 설정하지 않아서 한 Consumer가 모든 메시지를 독점하는 개발자
- RabbitMQ와 Kafka 중 무엇을 선택해야 하는지 기준이 없는 개발자
- Dead Letter Queue 없이 실패한 메시지를 그냥 버리는 개발자
- Spring AMQP를 쓰지만 내부 동작을 블랙박스로 두는 개발자

**선행 학습**:
- kafka-deep-dive (Kafka 내부 원리 이해 후 RabbitMQ와 비교 필수)
- network-deep-dive (TCP 기반 AMQP 연결 이해)
- spring-boot-internals (Auto-configuration과 RabbitAutoConfiguration 연결)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: RabbitMQ 아키텍처와 AMQP 프로토콜 (6개 문서)
- RabbitMQ가 해결하는 문제 — 동기 호출 체인의 결합도 문제, Producer가 Consumer의 존재를 몰라도 되는 설계, 트래픽 버퍼로서의 메시지 큐, RabbitMQ vs Kafka의 근본적 철학 차이 (메시지 브로커 vs 분산 로그)
- AMQP 프로토콜 완전 분해 — AMQP 0-9-1 프레임 구조(Method/Header/Body), Channel 멀티플렉싱(하나의 TCP 연결에 여러 채널), Connection vs Channel 비용 차이, AMQP의 3단계 라우팅(Exchange → Binding → Queue)
- Exchange 완전 분해 — Exchange가 메시지를 받아 어떤 Queue로 보낼지 결정하는 라우팅 컴포넌트, Binding이 Exchange와 Queue를 연결하는 규칙, Routing Key의 역할, Default Exchange(이름 없는 Exchange)의 동작
- Queue 내부 구조 — Queue의 FIFO 구조, 메시지 저장 방식(메모리 vs 디스크), Queue 속성(Durable/Exclusive/AutoDelete), Quorum Queue vs Classic Queue(내구성과 가용성 차이)
- 가상 호스트(vHost) — vHost로 브로커 내 논리적 분리, 팀/환경별 vHost 설계 전략, 권한 관리
- RabbitMQ 클러스터링 — 브로커 클러스터 구성, Queue 미러링(Mirrored Queue) vs Quorum Queue, 클러스터에서 네트워크 파티션(Split-Brain) 처리 전략

### Chapter 2: Exchange 유형과 라우팅 패턴 (6개 문서)
- Direct Exchange — Routing Key가 Binding Key와 정확히 일치할 때 라우팅, 1:1 라우팅과 1:N 라우팅(같은 Routing Key에 여러 Queue 바인딩), 사용 사례(특정 서비스로 메시지 전달)
- Topic Exchange — 와일드카드 패턴(* = 단어 하나, # = 0개 이상의 단어)으로 유연한 라우팅, Routing Key를 계층적으로 설계하는 방법(order.payment.failed), 구독 패턴의 유연성
- Fanout Exchange — Routing Key 무시하고 바인딩된 모든 Queue에 브로드캐스트, Pub/Sub 패턴 구현, 이벤트 하나를 여러 서비스가 처리하는 설계
- Headers Exchange — Routing Key 대신 메시지 헤더로 라우팅, x-match=all(모든 헤더 일치) vs x-match=any(하나 이상 일치), Headers Exchange가 Direct/Topic보다 유연한 경우
- Dead Letter Exchange(DLX) — 메시지가 Dead Letter가 되는 조건(소비 실패/TTL 만료/Queue 가득 참), DLX로 실패한 메시지를 별도 Queue로 라우팅, Dead Letter Queue 모니터링과 재처리 전략
- 라우팅 패턴 설계 가이드 — 도메인 이벤트를 Exchange로 설계하는 방법(OrderExchange → order.placed, order.cancelled), 마이크로서비스 간 이벤트 라우팅 설계, Exchange 재사용 vs 서비스별 Exchange 분리

### Chapter 3: 메시지 신뢰성 — 유실 없는 처리 (7개 문서)
- 메시지 유실의 세 지점 — Publisher → Broker 전송 중 실패, Broker Queue 저장 중 장애, Consumer 처리 중 실패 — 각 지점별 해결 방법
- Publisher Confirm — Broker가 메시지 수신을 확인하는 비동기 확인 메커니즘, confirmSelect()로 Confirm 모드 활성화, 개별 Confirm vs 배치 Confirm, Mandatory 플래그(라우팅 불가 메시지 반환)
- 메시지 영속성(Persistence) — Queue Durable + Message DeliveryMode=2(persistent)의 조합, 메시지가 디스크에 fsync되는 시점, 영속성이 처리량에 미치는 영향, Lazy Queue로 메모리 절약
- Consumer Acknowledgement 완전 분해 — autoAck=true의 위험(처리 전 메시지 삭제), 수동 Ack(basicAck), Nack(basicNack)와 Reject(basicReject) 차이, requeue=true vs false 선택 기준
- 재시도 전략 설계 — 처리 실패 시 즉시 requeue의 무한 루프 문제, Exponential Backoff 구현(TTL + Dead Letter + 재발행), 재시도 횟수 제한, 최종 실패 메시지의 Dead Letter Queue 처리
- Prefetch Count(QoS) 완전 분해 — Prefetch Count가 1이면 Consumer가 처리 중인 메시지 1개만 받는 이유, Prefetch Count가 크면 처리량이 높지만 불균등 분배 문제, 최적 Prefetch Count 산정 방법
- 트랜잭션 vs Publisher Confirm — AMQP 트랜잭션(txSelect/txCommit)의 성능 비용, Publisher Confirm이 트랜잭션보다 선호되는 이유, 완전한 메시지 처리 보장 패턴(Outbox + Publisher Confirm)

### Chapter 4: 실무 메시징 패턴 (6개 문서)
- Work Queue(경쟁 소비자) — 여러 Consumer가 하나의 Queue를 공유하여 병렬 처리, Round-Robin 분배의 기본 동작, Prefetch Count로 공정한 분배, 처리 시간이 긴 작업의 병렬화
- Publish/Subscribe — Fanout Exchange로 브로드캐스트, 각 Consumer가 자신만의 임시 Queue를 생성, Consumer 추가/제거 시 자동 확장/축소, 로그 시스템, 알림 시스템 구현
- RPC(Remote Procedure Call) — Reply-To Queue와 Correlation ID로 요청-응답 패턴 구현, 동기 RPC의 한계와 타임아웃 처리, RabbitMQ를 RPC 백엔드로 쓰는 경우와 그러지 않는 경우
- Priority Queue — Queue에 x-max-priority 설정으로 메시지 우선순위 처리, 높은 우선순위 메시지가 먼저 소비되는 원리, 우선순위 큐 설계 시 주의사항(낮은 우선순위 메시지 굶주림)
- Delayed/Scheduled Message — RabbitMQ Delayed Message Plugin 또는 TTL + DLX 조합으로 지연 메시지 구현, 정확한 지연 보장 한계, 스케줄링 서비스(Quartz)와의 비교
- Saga 패턴 구현 — Choreography Saga에서 RabbitMQ 사용, 각 서비스가 이벤트를 발행하고 구독하는 흐름, 보상 트랜잭션 이벤트 라우팅, Orchestration Saga에서 Orchestrator의 명령 발행

### Chapter 5: 성능 튜닝과 운영 (5개 문서)
- 처리량 최적화 — Batch Publishing, Publisher Confirm 비동기 처리, Consumer 병렬화(동시 Consumer 수), 메시지 압축, 적절한 Prefetch Count 설정
- 메모리 관리 — RabbitMQ 메모리 임계값(vm_memory_high_watermark), 메모리 초과 시 Publisher 차단(Flow Control), Lazy Queue로 메시지를 디스크에 저장하여 메모리 절약
- 모니터링 핵심 지표 — Queue Depth(메시지 적체), Consumer Utilisation(소비자 효율), Publish Rate vs Deliver Rate, 연결/채널 수, RabbitMQ Management UI와 Prometheus Exporter
- 운영 중 발생하는 문제 패턴 — Queue 메시지 무한 적체(Consumer 처리 속도 < 발행 속도), Unacknowledged 메시지 누적(Prefetch 크고 처리 느림), 연결 폭풍(서버 재시작 시 수백 개 클라이언트 동시 재연결)
- 클러스터 운영 — 노드 추가/제거, Queue 균등 분산, 노드 장애 시 Quorum Queue 자동 복구, Rolling Upgrade 절차

### Chapter 6: RabbitMQ vs Kafka — 언제 무엇을 선택하는가 (4개 문서)
- 근본적 철학 차이 완전 분해 — RabbitMQ: 메시지 브로커(Consumer가 처리하면 삭제), Kafka: 분산 로그(메시지를 보관, Consumer가 오프셋으로 읽기), Push 기반 vs Pull 기반 소비 방식의 트레이드오프
- 사용 사례 비교 — RabbitMQ가 적합한 경우(복잡한 라우팅, 요청-응답 패턴, 메시지 우선순위, 작업 큐, 낮은 지연시간), Kafka가 적합한 경우(이벤트 소싱, 로그 집계, 대용량 스트리밍, 재처리 필요, 여러 Consumer 그룹)
- 성능 특성 비교 — RabbitMQ: 낮은 지연시간(마이크로초~밀리초), Kafka: 높은 처리량(초당 수백만 메시지), 각각의 최적 메시지 크기와 규모
- 함께 쓰는 경우 — RabbitMQ(작업 큐, 실시간 알림) + Kafka(이벤트 스트림, 감사 로그)를 동시에 운영하는 아키텍처, 각 미들웨어의 책임 분리

### Chapter 7: Spring AMQP 통합 (4개 문서)
- Spring AMQP 아키텍처 — RabbitTemplate(발행), @RabbitListener(소비), RabbitAdmin(Exchange/Queue/Binding 자동 생성), SimpleMessageListenerContainer vs DirectMessageListenerContainer 차이
- 메시지 직렬화 전략 — Jackson2JsonMessageConverter(JSON), SimpleMessageConverter(String/byte[]), 커스텀 MessageConverter, 메시지 헤더에 타입 정보 포함 방법
- 에러 처리와 재시도 — RetryTemplate으로 Consumer 재시도, RejectAndDontRequeueRecoverer로 최종 실패 처리, DeadLetterPublishingRecoverer로 DLX 연동, @RabbitListener concurrency 설정
- Spring Boot Auto-configuration — RabbitAutoConfiguration 동작 방식, application.yml로 Exchange/Queue/Binding 선언, Test에서 EmbeddedRabbitMQ 활용

---

총 **38개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 RabbitMQ에서 중요한가
## 😱 흔한 실수 (Before — RabbitMQ 내부를 모를 때의 설정)
## ✨ 올바른 접근 (After — 올바른 설정과 패턴)
## 🔬 내부 동작 원리 (AMQP 프로토콜, 라우팅 알고리즘, 메모리 관리)
## 💻 실전 코드 (Spring AMQP + RabbitMQ Management UI 실험)
## 📊 Kafka vs RabbitMQ 비교 (같은 문제를 어떻게 다르게 해결하는가)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 모든 Exchange 설명은 실제 메시지 흐름 다이어그램 포함
2. Kafka와의 비교를 항상 포함 — "Kafka였다면 어떻게 했을까"
3. RabbitMQ Management UI에서 확인 가능한 실험 포함
4. 메시지 유실 시나리오 항상 포함 — "이 설정이 없으면 언제 메시지가 사라지는가"
5. Spring AMQP 코드 항상 포함 — 설정이 내부 동작과 어떻게 연결되는가

**실험 환경**:
```yaml
# docker-compose.yml
services:
  rabbitmq:
    image: rabbitmq:3.12-management
    ports:
      - "5672:5672"   # AMQP
      - "15672:15672" # Management UI
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin
    volumes:
      - ./rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
      - rabbitmq-data:/var/lib/rabbitmq
    healthcheck:
      test: rabbitmq-diagnostics check_port_connectivity
      interval: 10s

  # 클러스터 노드 2
  rabbitmq-node2:
    image: rabbitmq:3.12-management
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: admin
      RABBITMQ_ERLANG_COOKIE: secret-cookie
    depends_on:
      - rabbitmq

  rabbitmq-exporter:
    image: kbudde/rabbitmq-exporter:latest
    environment:
      RABBIT_URL: http://rabbitmq:15672
      RABBIT_USER: admin
      RABBIT_PASSWORD: admin
    ports:
      - "9419:9419"

  spring-app:
    build: .
    environment:
      SPRING_RABBITMQ_HOST: rabbitmq
      SPRING_RABBITMQ_PORT: 5672
      SPRING_RABBITMQ_USERNAME: admin
      SPRING_RABBITMQ_PASSWORD: admin
    depends_on:
      rabbitmq:
        condition: service_healthy

volumes:
  rabbitmq-data:
```

---

## 💡 핵심 분석 대상

```
=== AMQP 라우팅 전체 흐름 ===

Producer (Spring RabbitTemplate)
  │
  ▼ exchange: "order.exchange"
    routingKey: "order.payment.failed"
    body: { orderId: 123, reason: "..." }
  │
  ▼ RabbitMQ Exchange: order.exchange (Topic)
  │
  Binding 규칙 검색:
  ├── "order.#"          → payment-service-queue   ✅ 일치
  ├── "order.payment.*"  → audit-service-queue      ✅ 일치
  ├── "order.shipped"    → shipping-service-queue   ❌ 불일치
  └── "#.failed"         → alert-service-queue      ✅ 일치
  │
  ▼ 일치하는 Queue로 메시지 복사 (팬아웃)
  payment-service-queue → Consumer A
  audit-service-queue   → Consumer B
  alert-service-queue   → Consumer C

=== Kafka vs RabbitMQ 핵심 차이 ===

메시지 소비 후:
  RabbitMQ: 메시지 삭제 (Queue에서 제거)
    → Consumer가 처리하면 끝
    → 과거 메시지 재처리 불가
    → Consumer 수에 독립적 (Queue 하나를 N개 Consumer가 공유)

  Kafka: 메시지 보관 (보관 기간 설정까지)
    → Consumer Group이 오프셋으로 추적
    → 과거 메시지 재처리 가능 (오프셋 리셋)
    → Consumer Group 추가 시 독립적 소비 가능

소비 방식:
  RabbitMQ: Push (Broker → Consumer로 밀어넣기)
    → Consumer가 처리 속도를 Prefetch로 조절
    → 낮은 지연시간에 유리

  Kafka: Pull (Consumer가 Broker에서 가져오기)
    → Consumer가 직접 속도 조절
    → 높은 처리량에 유리

선택 기준:
  RabbitMQ가 적합한 경우:
    ✅ 복잡한 라우팅 (조건에 따라 다른 서비스로)
    ✅ 요청-응답 패턴 (RPC)
    ✅ 메시지 우선순위 필요
    ✅ 낮은 지연시간 (< 10ms)
    ✅ 메시지 처리 후 삭제되어야 함

  Kafka가 적합한 경우:
    ✅ 이벤트 소싱 / 감사 로그 (메시지 보관)
    ✅ 대용량 스트리밍 (초당 수백만)
    ✅ 여러 Consumer Group이 독립적으로 소비
    ✅ 재처리가 필요한 경우
    ✅ 이벤트 기반 아키텍처

=== 메시지 유실 방지 완전 패턴 ===

Publisher 측:
  1. Publisher Confirm으로 Broker 수신 확인
  2. Mandatory=true로 라우팅 불가 메시지 반환
  3. Outbox Pattern으로 DB 저장 + 발행 원자성

Broker 측:
  1. Queue Durable=true
  2. Message DeliveryMode=PERSISTENT
  3. Quorum Queue로 클러스터 내구성

Consumer 측:
  1. autoAck=false (수동 Ack)
  2. 처리 완료 후 basicAck
  3. 처리 실패 시 basicNack(requeue=false)
  4. DLX로 실패 메시지 → Dead Letter Queue
  5. Dead Letter Queue에서 재처리 또는 알림
```

---

## 📚 참고 자료

- RabbitMQ 공식 문서 — https://www.rabbitmq.com/documentation.html
- AMQP 0-9-1 프로토콜 스펙 — https://www.amqp.org/
- RabbitMQ in Depth (Gavin Roy)
- Spring AMQP 공식 문서 — https://docs.spring.io/spring-amqp/docs/current/reference/html/
- CloudAMQP Blog — https://www.cloudamqp.com/blog/ (실전 운영 사례)
- Michael Klishin(RabbitMQ 핵심 개발자) 블로그 — https://blog.rabbitmq.com/

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (38개 목표)
- Docker Compose 실험 환경 (RabbitMQ 클러스터 + Management UI + Exporter)
- kafka-deep-dive와 비교 연결 지점 명시 (어느 문서에서 Kafka와 대비하는지)

**준비됐으면 1단계 구조 설계부터 시작해줘!**
