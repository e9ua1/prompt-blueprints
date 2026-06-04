# Kafka Deep Dive 레포지토리 제작 프롬프트

나는 "Kafka Deep Dive" 레포지토리를 만들려고 해.
메시지를 발행하고 소비하는 것과, Kafka가 파티션마다 로그 파일에 어떻게 순차 기록하는지, ISR(In-Sync Replicas)이 어떻게 내구성을 보장하는지, Exactly-Once가 실제로 어떻게 구현되는지를 완전히 파헤치는 레포를 만들 거야.

"@KafkaListener 붙이면 메시지를 받겠지"와 "리밸런싱이 왜 발생하고 처리 중 리밸런싱이 데이터 중복을 일으키는지, acks=all이 실제로 무엇을 보장하는지 아는 것"의 차이를 만드는 레포야.

## 📋 프로젝트 목표

**컨셉**: "메시지를 발행하는 것과, Kafka가 어떻게 순서와 내구성을 보장하는지 아는 것은 다르다"

**핵심 차별화**:
1. 파티션과 로그 — Kafka가 파일에 순차 기록하는 방식이 왜 빠른가
2. 복제와 ISR — acks=all이 실제로 보장하는 것과 보장하지 않는 것
3. 전달 보장 — At-Least-Once / Exactly-Once가 어떻게 구현되는가
4. 리밸런싱 — Consumer Group 재조정이 왜 발생하고 어떻게 최소화하는가

**타겟 독자**:
- Kafka를 메시지 큐로 쓰지만 토픽과 파티션의 차이가 왜 중요한지 모르는 개발자
- acks=1과 acks=all의 차이를 설명 못하는 개발자
- Consumer Lag이 쌓이는 원인을 파악하지 못하는 개발자
- 리밸런싱이 발생할 때마다 중복 메시지를 처리하는 코드를 짜지 못하는 개발자
- Exactly-Once를 원하지만 어떻게 구현해야 하는지 모르는 개발자
- Spring Kafka를 쓰면서 Kafka 내부를 블랙박스로 두는 개발자

**선행 학습**:
- 없음 (이 레포 자체가 독립 완결)
- spring-batch-deep-dive (Kafka + Batch 조합, Chunk 처리와 exactly-once 연결 시 시너지)
- spring-cloud-deep-dive (이벤트 기반 마이크로서비스, Spring Cloud Stream 연결 시 시너지)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Kafka 핵심 아키텍처 (6개 문서)
- Kafka 설계 철학 — 왜 메시지 큐가 아닌 분산 로그인가, MQ(RabbitMQ 등)와 근본적으로 다른 점
- 토픽과 파티션 — 파티션이 병렬성의 기본 단위인 이유, 파티션 수 선택이 처리량과 지연에 미치는 영향
- 브로커 내부 구조 — 로그 세그먼트 파일 구조(.log/.index/.timeindex), 순차 I/O가 빠른 이유, 페이지 캐시 활용
- Producer 내부 동작 — RecordAccumulator 배치, linger.ms와 batch.size 상호작용, 파티션 선택 전략(Key 해시, Round-Robin, Sticky)
- Consumer 내부 동작 — Fetch 루프, max.poll.records, 폴링 간격이 리밸런싱에 미치는 영향
- ZooKeeper vs KRaft — 메타데이터 관리의 진화, KRaft 아키텍처에서 Controller 쿼럼 동작

### Chapter 2: 복제와 내구성 보장 (6개 문서)
- 파티션 복제 — Leader/Follower 구조, Follower가 Leader를 따라가는 방식(Fetch 기반)
- ISR(In-Sync Replicas) — ISR 조건(replica.lag.time.max.ms), ISR 이탈/복귀가 가용성에 미치는 영향
- acks 설정 완전 분해 — acks=0/1/all이 실제로 보장하는 것, acks=all이어도 데이터가 유실될 수 있는 조건
- min.insync.replicas — ISR 최솟값 설정이 가용성과 내구성을 어떻게 트레이드오프하는가
- Leader Election — 파티션 리더가 선출되는 조건, Unclean Leader Election의 데이터 유실 위험
- Log Compaction — 키 기반 최신값 보존 원리, Compaction 대상 세그먼트 선택, Consumer가 Compacted 토픽을 읽는 방식

### Chapter 3: 전달 보장과 멱등성 (5개 문서)
- 전달 보장 3단계 — At-Most-Once/At-Least-Once/Exactly-Once의 구현 방법과 실제 차이
- Producer 멱등성 — enable.idempotence=true 내부 동작, Producer ID + Sequence Number로 중복 제거
- 트랜잭션 Producer — transactional.id, Epoch, Two-Phase Commit으로 멀티 파티션 원자적 쓰기
- Consumer Offset 관리 — 자동 커밋 vs 수동 커밋의 중복/유실 시나리오, commitSync vs commitAsync
- Exactly-Once Semantics(EOS) 전 과정 — Read-Process-Write 패턴에서 EOS를 달성하는 방법

### Chapter 4: Consumer Group과 리밸런싱 (5개 문서)
- Consumer Group 내부 동작 — Group Coordinator, Consumer Group 상태 머신(Empty→PreparingRebalance→Stable)
- 리밸런싱 발생 조건 — heartbeat 누락, max.poll.interval.ms 초과, Consumer 추가/제거 시 발생 흐름
- 리밸런싱 전략 비교 — Eager vs Cooperative(Incremental) 리밸런싱 차이, Stop-The-World 문제
- 리밸런싱 중 중복 처리 — 처리 중 리밸런싱이 발생하면 왜 중복 메시지가 생기는가, 멱등 처리 패턴
- Consumer Lag — lag이 쌓이는 원인 분석, Consumer 처리 속도 향상 전략

### Chapter 5: 성능 튜닝과 모니터링 (5개 문서)
- Producer 처리량 최적화 — batch.size, linger.ms, buffer.memory, compression.type(gzip/snappy/lz4/zstd) 비교
- Consumer 처리량 최적화 — fetch.min.bytes, fetch.max.wait.ms, max.partition.fetch.bytes 튜닝
- 파티션 핫스팟 — 특정 파티션에 메시지 집중 현상 원인과 Custom Partitioner 구현
- Kafka 모니터링 — JMX 메트릭 핵심 지표, kafka-exporter + Prometheus + Grafana 대시보드 구성
- 운영 중 발생하는 문제 패턴 — Under-Replicated Partitions, Consumer Group 무한 리밸런싱, 오프셋 리셋 절차

### Chapter 6: Kafka Streams와 이벤트 기반 설계 (5개 문서)
- Kafka Streams 아키텍처 — Topology, StreamTask, 내부 상태 저장소(RocksDB) + Changelog 토픽
- KTable과 KStream — 각각의 의미(스냅샷 vs 이벤트 스트림), KStream-KTable 조인 원리
- 윈도우 연산 — Tumbling/Hopping/Sliding/Session Window 차이, 지연 이벤트(Late Arrival) 처리
- Exactly-Once in Kafka Streams — processing.guarantee=exactly_once_v2 내부 동작
- 이벤트 기반 아키텍처 패턴 — Outbox Pattern으로 DB 트랜잭션과 Kafka 발행을 원자적으로 처리

### Chapter 7: Spring과 Kafka 통합 (5개 문서)
- Spring Kafka 기초 — KafkaTemplate, @KafkaListener의 내부 AOP 동작, ContainerFactory 구성
- 에러 처리와 재시도 — DefaultErrorHandler, SeekToCurrentErrorHandler, Dead Letter Topic(DLT) 패턴
- 트랜잭션 처리 — @Transactional과 KafkaTransactionManager 조합, DB 트랜잭션과 Kafka 트랜잭션의 경계
- Spring Cloud Stream — 함수형 프로그래밍 모델, 바인더 추상화, Kafka Binder 내부 설정
- Spring Batch + Kafka — Kafka ItemReader/ItemWriter 구현, Chunk 처리와 Exactly-Once 달성

---

각 챕터는 **4~7개 문서**로 구성해줘. 총 **37개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 실무에서 중요한가
## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)
## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)
## 🔬 내부 동작 원리 (Kafka 브로커/클라이언트 내부 분석)
## 💻 실전 실험 (kafka-console-producer/consumer, kafka-topics, 재현 시나리오)
## 📊 성능/비용 비교 (acks 설정별 처리량, 압축 방식별 비교 등)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 추상적 설명 금지 — 모든 원리는 "실제로 어떤 파일에, 어떤 형태로 기록되는가"로 연결
2. kafka-console-producer/consumer와 kafka-topics CLI로 바로 재현할 수 있는 실험 포함
3. 각 설계 결정의 이유 명시 ("왜 파티션인가", "왜 Pull 방식인가", "왜 offset을 Consumer가 관리하는가")
4. Spring 애플리케이션 관점 연결 — @KafkaListener 설정 옵션이 내부 메커니즘과 어떻게 연결되는지
5. 운영 장애 시나리오 → kafka-consumer-groups, kafka-log-dirs 진단 명령어 세트

**실험 환경**:
```yaml
# docker-compose.yml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  kafka-1:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-1:9092
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_LOG_DIRS: /var/kafka-logs
    ports:
      - "9092:9092"

  kafka-2:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-2:9093
    ports:
      - "9093:9092"

  kafka-3:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-3:9094
    ports:
      - "9094:9092"

  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    depends_on:
      - kafka-1
    environment:
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka-1:9092
    ports:
      - "8081:8081"

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    depends_on:
      - kafka-1
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka-1:9092
    ports:
      - "8080:8080"

  kafka-exporter:
    image: danielqsj/kafka-exporter
    command: --kafka.server=kafka-1:9092
    ports:
      - "9308:9308"
```

```bash
# 실험용 공통 명령어
# 토픽 생성 (복제 팩터 3, 파티션 3)
kafka-topics --bootstrap-server localhost:9092 \
  --create --topic orders \
  --replication-factor 3 \
  --partitions 3

# ISR 상태 확인
kafka-topics --bootstrap-server localhost:9092 \
  --describe --topic orders

# Consumer Group Lag 확인
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --describe --group order-consumer-group

# 오프셋 리셋 (재처리)
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group order-consumer-group \
  --topic orders \
  --reset-offsets --to-earliest --execute

# 로그 세그먼트 파일 확인
kafka-log-dirs --bootstrap-server localhost:9092 \
  --topic-list orders \
  --describe
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**:
   - mysql-deep-dive README 스타일 참고
   - "단순 메시지 발행/소비를 넘어 Kafka 내부를 이해한다"는 차별화 강조
   - Spring Kafka/Spring Batch/Spring Cloud Stream과의 연결 지점 명시
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료 섹션

README 작성 시 다음을 Reference로 포함해줘:
- Kafka 공식 문서 — https://kafka.apache.org/documentation/
- Kafka: The Definitive Guide, 2nd Edition (Neha Narkhede, Gwen Shapira, Todd Palino)
- Designing Data-Intensive Applications (Martin Kleppmann) — Kafka 관련 챕터
- Confluent Blog — https://www.confluent.io/blog/
- Kafka Improvement Proposals (KIPs) — https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals

---

## 💡 핵심 분석 대상

```
Kafka 메시지 처리 전체 흐름:

Producer (Spring KafkaTemplate)
  │
  ▼ 직렬화 (Serializer 설정에 따라)
  ▼ 파티션 결정 (Key Hash / Round-Robin / Sticky)
  ▼ RecordAccumulator 배치 (linger.ms / batch.size)
  ▼
브로커 수신
  ├── Leader 파티션 로그 세그먼트에 순차 기록 (.log 파일)
  ├── 인덱스 파일 갱신 (.index, .timeindex)
  └── Follower에게 복제 (ISR 기준)
        │
        ▼ acks 설정에 따라 응답 시점 다름
        acks=0: 브로커 응답 기다리지 않음
        acks=1: Leader 기록 완료 시
        acks=all: ISR 전체 기록 완료 시

Consumer (Spring @KafkaListener)
  │
  ▼ Group Coordinator에서 파티션 할당
  ▼ Fetch 루프 (max.poll.records 단위)
  ▼ 처리 로직 실행
  ▼ Offset 커밋 (자동 or 수동)
  │
  ※ 처리 중 리밸런싱 발생 시:
    처리한 메시지의 오프셋 미커밋 → 다음 Consumer가 재처리
    = At-Least-Once (중복 발생 가능)
    해결: 멱등 처리 로직 + 트랜잭션 또는 Cooperative 리밸런싱

Exactly-Once 달성 흐름:
  Producer:
    enable.idempotence=true (중복 제거)
    + transactional.id 설정
    + beginTransaction() → send() → commitTransaction()
  Consumer:
    isolation.level=read_committed (커밋된 메시지만 읽기)
  = 프로듀서 중복 제거 + 원자적 멀티 파티션 쓰기 + 소비자 미커밋 무시

리밸런싱 발생 시나리오:
  Consumer A가 1000ms 처리 중
    → max.poll.interval.ms = 300ms 초과
    → Consumer A가 그룹에서 제거됨
    → 리밸런싱 시작 → A의 파티션 재배정
    → 이미 처리 중이던 메시지 재처리 = 중복

  해결:
  1. max.poll.interval.ms 늘리기 (긴 처리 허용)
  2. max.poll.records 줄이기 (한 번에 적게 처리)
  3. Cooperative Rebalancing (incremental.cooperative.rebalancing)

Consumer Lag 누적:
  원인 분석 트리:
  ├── 파티션 수 < Consumer 수 → 일부 Consumer 유휴
  ├── 처리 로직이 느림 → max.poll.records 줄이기
  ├── DB 호출이 병목 → 배치 처리로 DB 왕복 감소
  └── 리밸런싱 반복 → heartbeat 간격 최적화
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (37개 목표)
- Docker Compose 실험 환경 (3-브로커 클러스터 + Schema Registry + Kafka UI 포함)
- Spring Kafka/Spring Batch/Spring Cloud Stream과 연결되는 지점 명시 (어느 문서에서 어느 레포와 연결하는지)

**준비됐으면 1단계 구조 설계부터 시작해줘!**
