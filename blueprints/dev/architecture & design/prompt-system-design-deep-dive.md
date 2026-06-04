# System Design Deep Dive 레포지토리 제작 프롬프트

나는 "System Design Deep Dive" 레포지토리를 만들려고 해.
Redis, Kafka, MySQL, Elasticsearch를 각각 아는 것과, 실제 서비스를 설계할 때 "왜 이 시점에 캐시를 넣는가", "왜 이 작업은 메시지 큐로 비동기화하는가", "왜 여기서 읽기 전용 복제본을 분리하는가"를 결정하는 것은 완전히 다른 능력이야.

이 레포는 개별 기술을 다시 설명하지 않아. 대신 "어떤 트레이드오프 때문에 어떤 기술을 선택하고, 그것들을 어떻게 조합해서 대규모 시스템을 만드는가"를 다뤄.

## 📋 프로젝트 목표

**컨셉**: "기술을 아는 것과, 수백만 사용자를 위한 시스템을 설계할 수 있는 것은 다르다"

**핵심 차별화**:
1. 설계 결정의 근거 — "이 기술을 선택한 이유"를 항상 트레이드오프로 설명
2. 확장성의 원리 — 수직 확장의 한계가 오는 시점, 수평 확장이 가능하려면 무엇이 바뀌어야 하는가
3. 장애 허용 설계 — 컴포넌트가 실패해도 시스템이 동작하는 설계의 원칙
4. 실제 사례 중심 — URL 단축기, 트위터 타임라인, 유튜브, 채팅 시스템을 실제로 설계

**타겟 독자**:
- Redis를 알지만 "언제 캐시를 넣어야 하는가"를 결정 못하는 개발자
- Kafka를 알지만 "왜 이 작업을 메시지 큐로 해야 하는가"를 설명 못하는 개발자
- DB 샤딩이 무엇인지 알지만 "어떤 키로 샤딩할 것인가"를 결정 못하는 개발자
- 면접에서 "DAU 1000만 서비스를 설계하라"는 질문에 막히는 개발자
- 아키텍처 결정을 할 때 근거 없이 유행하는 기술을 선택하는 개발자
- 단일 서버에서 잘 동작하는 코드가 트래픽이 늘면 왜 망가지는지 모르는 개발자

**선행 학습**:
- 모든 IQ Dev Lab 레포 (각 기술의 내부 원리 이해 후 설계 판단 가능)
- 특히: database-internals, redis-deep-dive, kafka-deep-dive, network-deep-dive

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 시스템 설계의 기초 원칙 (7개 문서)
- 확장성(Scalability) 완전 분해 — 수직 확장(Scale Up)의 물리적 한계, 수평 확장(Scale Out)이 가능하려면 무엇이 달라야 하는가(Stateless), 로드밸런서가 트래픽을 분산하는 방법(Round Robin/Least Connection/IP Hash), Sticky Session의 문제
- 가용성(Availability) 계산 — 99.9%(3 Nine) vs 99.99%(4 Nine)의 실제 다운타임 차이(연간 8.7시간 vs 52분), SLA와 SLO, 단일 장애점(SPOF) 제거 전략, 이중화(Redundancy)의 비용
- 일관성(Consistency) vs 가용성(Availability) — CAP Theorem이 실제 설계 결정에 어떻게 적용되는가, CP 시스템(MySQL Master)과 AP 시스템(Redis Cluster) 선택 기준, Eventual Consistency를 허용하는 설계
- 지연시간(Latency) vs 처리량(Throughput) — P50/P95/P99 지연시간의 의미, 롱테일 지연시간 문제, 처리량을 높이면 지연시간이 올라가는 이유, 배치 처리로 처리량 vs 지연시간 트레이드오프
- 용량 산정(Capacity Estimation) — DAU/MAU로 초당 요청 수 계산, 저장 용량 산정(5년치 데이터), 네트워크 대역폭 계산, Back-of-the-Envelope 계산법과 면접에서의 활용
- 데이터 모델 선택 — RDB(강한 일관성, 복잡한 관계) vs NoSQL(확장성, 유연한 스키마) 선택 기준, 각 NoSQL 유형(Document/Key-Value/Column-Family/Graph) 적합한 사용 사례
- 설계 면접 접근법 — 요구사항 명확화(기능/비기능), 개략적 설계 먼저, 핵심 컴포넌트 상세화, 병목 식별과 해결, 면접관이 원하는 것은 완벽한 답이 아닌 트레이드오프 논의

### Chapter 2: 핵심 인프라 컴포넌트 (7개 문서)
- DNS와 로드밸런서 — DNS가 지역별 트래픽을 분산하는 방법(GeoDNS), L4 vs L7 로드밸런서 차이, HAProxy vs Nginx vs AWS ALB, 로드밸런서 자체가 SPOF가 되지 않으려면
- CDN(Content Delivery Network) — 정적 자원을 엣지 서버에 캐싱하는 원리, Push CDN vs Pull CDN, CDN이 없으면 이미지 서비스가 왜 느린가, Cache-Control 헤더와 CDN 갱신 전략
- 캐싱 계층 설계 — 캐시를 어디에 놓는가(클라이언트/CDN/API Gateway/애플리케이션/DB), 캐시 히트율과 DB 부하의 관계, 캐시 갱신 전략(TTL/Event-Driven/Write-Through), 분산 캐시 구성(Redis Cluster)
- 메시지 큐와 비동기 처리 — 어떤 작업을 비동기로 분리해야 하는가(이메일 발송, 이미지 리사이징, 결제 후처리), 메시지 큐가 없으면 트래픽 급증 시 무슨 일이 생기는가(백프레셔 없는 동기 처리), Kafka vs RabbitMQ 선택 기준
- 데이터베이스 확장 — 읽기 복제본으로 읽기 트래픽 분산, 파티셔닝(테이블 내 분할) vs 샤딩(서버 간 분할), 샤딩 키 선택의 중요성(Hot Shard 문제), CQRS와 DB 분리
- 검색 인프라 — 풀텍스트 검색에 DB LIKE 쿼리가 안 되는 이유, Elasticsearch 도입 시점(검색 기능이 핵심인 경우), DB와 Elasticsearch 데이터 동기화 전략
- 스토리지 계층 — Block Storage vs Object Storage vs File Storage 선택 기준, S3 같은 Object Storage가 이미지/영상 저장에 적합한 이유, CDN + Object Storage 조합

### Chapter 3: 실전 설계 — 기본 서비스 (7개 문서)
- URL 단축 서비스(bit.ly) — 10억 개 URL 저장, 초당 10만 리디렉션, 단축 키 생성 전략(랜덤 vs Base62 vs Counter 기반), 충돌 처리, DB 설계, 캐싱(핫 URL), 커스텀 URL 지원, 분석 기능 추가
- 키-값 저장소 설계 — 분산 해시 테이블, 일관성 해싱(Consistent Hashing)으로 노드 추가/삭제 시 리밸런싱 최소화, 복제(Replication) 전략, 가용성 vs 일관성 선택, Gossip Protocol로 노드 상태 전파
- 웹 크롤러 설계 — 수십억 페이지를 크롤링하는 분산 크롤러, 중복 URL 감지(BloomFilter), 크롤링 우선순위(Page Rank 기반), Robots.txt 준수, 스케일아웃 전략
- 분산 ID 생성기 — 순서 보장이 필요한 경우 vs 불필요한 경우, UUID(충돌 없지만 정렬 불가), 데이터베이스 Auto Increment(단일 SPOF), Snowflake ID(Twitter 방식 — 타임스탬프+서버ID+시퀀스), 각 방식의 트레이드오프
- Rate Limiter 설계 — Token Bucket vs Leaky Bucket vs Fixed Window vs Sliding Window 알고리즘 비교, 분산 환경에서 Rate Limiter(Redis + Lua Script), API Gateway에서의 구현, 클라이언트 응답 헤더 설계
- 알림 시스템 — Push(iOS APNs/Android FCM)/SMS/이메일 채널, 메시지 큐로 알림 발송 비동기화, 재시도 전략, 알림 중복 방지(멱등성), 사용자 설정(알림 거부) 처리
- 검색 자동완성(Typeahead) — 트라이(Trie) 자료구조로 접두사 검색, 실시간 검색어 업데이트 vs 배치 업데이트, 검색어 필터링(금지어), 분산 트라이(Trie Sharding), 캐싱 전략

### Chapter 4: 실전 설계 — 대규모 서비스 (7개 문서)
- 유튜브/넷플릭스 설계 — 영상 업로드(멀티파트 + Object Storage), 인코딩 파이프라인(여러 해상도/코덱으로 비동기 변환), CDN으로 영상 스트리밍, 어댑티브 비트레이트(ABR), 추천 시스템 연결
- 트위터 타임라인 설계 — Fan-out on Write(팔로워에게 즉시 배포) vs Fan-out on Read(조회 시 계산) 트레이드오프, 셀러브리티 계정(팔로워 1000만)의 Fan-out 문제, 하이브리드 전략, Redis Sorted Set으로 타임라인 저장
- Facebook 뉴스피드 설계 — Edge Rank 알고리즘, 친구/그룹/페이지 게시물 랭킹, 실시간 업데이트 vs 배치 처리, 광고 인서팅 포인트, 읽기 최적화 vs 쓰기 최적화 트레이드오프
- 채팅 시스템(Slack/WhatsApp) — 실시간 메시지 전달(WebSocket vs Long Polling vs SSE), 온라인/오프라인 상태 관리, 1:1 채팅 vs 그룹 채팅 차이, 메시지 저장(HBase/Cassandra 선택 이유), 메시지 순서 보장
- Google Drive / Dropbox 설계 — 파일 청크 업로드(대용량 파일 처리), 중복 청크 감지(해시 기반), 델타 동기화(변경된 청크만), 파일 버저닝, 동시 편집 충돌 해결
- 대규모 검색 엔진 — 웹 크롤러 + 인덱서 + 검색 서버 파이프라인, 역색인 분산 저장(Elasticsearch 클러스터), 페이지랭크 계산, 광고 경매 시스템, 검색 결과 개인화
- 라이브 스트리밍 플랫폼(트위치) — RTMP 스트림 수신, 트랜스코딩 파이프라인, HLS 세그먼트 생성, CDN 분산, 낮은 지연시간(Low Latency) vs 안정성 트레이드오프, 채팅 동시성(초당 수만 메시지)

### Chapter 5: 데이터 집약적 시스템 패턴 (5개 문서)
- 데이터 파이프라인 설계 — 배치 처리(Hadoop MapReduce) vs 스트림 처리(Kafka Streams/Flink), Lambda Architecture(배치+스트림 혼합), Kappa Architecture(스트림만), 선택 기준
- 대규모 데이터 집계 — 광고 클릭 집계(실시간 vs 배치), 시계열 DB(InfluxDB/TimescaleDB), 사전 집계(Pre-aggregation) vs 실시간 집계, 정확성 vs 속도 트레이드오프
- 분산 캐시 심화 — 캐시 일관성 문제(Cache Invalidation이 어려운 이유), 분산 환경에서 캐시 갱신 전략, Cache Stampede 방지(Mutex/PER), Hot Key 분산(Local Cache + Redis)
- 샤딩 전략 심화 — 범위 기반 샤딩(날짜/알파벳)의 핫스팟 문제, 해시 기반 샤딩의 리밸런싱 문제, 일관성 해싱으로 리밸런싱 최소화, Vitess(MySQL 샤딩 미들웨어) 아키텍처
- 글로벌 분산 시스템 — 다중 데이터센터 배포, 지역 간 데이터 복제(비동기 복제의 일관성 문제), 사용자를 가까운 DC로 라우팅(GeoDNS), 재해 복구(DR) 전략

### Chapter 6: 신뢰성과 장애 대응 (5개 문서)
- 장애 허용 설계 원칙 — Chaos Engineering(의도적 장애 주입), Circuit Breaker가 없으면 장애가 전파되는 시나리오, Bulkhead로 장애 격리, Timeout 없이 서비스를 호출하면 생기는 자원 고갈
- 데이터 일관성 보장 — 분산 환경에서 2PC 없이 일관성 달성(Saga), 멱등성(Idempotency) 설계(같은 요청이 두 번 와도 안전하게), Exactly-Once 처리 구현 패턴
- 모니터링과 경보 — SLI(Service Level Indicator) 측정 항목 선택(에러율/지연시간/처리량), SLO 설정 기준, 경보 피로(Alert Fatigue) 방지, Runbook 작성
- 재해 복구(DR) 전략 — RPO(데이터 손실 허용 시간)와 RTO(복구 목표 시간) 설정, 백업 전략(풀 백업/증분 백업/연속 복제), 복구 절차 자동화, 정기 DR 훈련
- 포스트모텀(Post-Mortem) — 장애 발생 후 원인 분석 방법, 비난 없는 문화(Blameless), 재발 방지 액션 아이템 도출, 5-Why 분석법

### Chapter 7: 설계 결정 프레임워크 (4개 문서)
- 아키텍처 결정 기록(ADR) — 기술 선택의 이유를 문서화하는 방법, 미래의 나와 팀을 위한 컨텍스트 보존, ADR 템플릿과 예시(왜 Kafka를 선택했는가, 왜 MySQL을 PostgreSQL 대신 썼는가)
- 기술 부채 관리 — 의도적 부채(빠른 출시를 위한 타협)와 비의도적 부채, 부채 측정과 상환 우선순위, 점진적 리팩터링 전략
- 스타트업부터 대기업까지 성장 경로 — 단일 서버 → 읽기 복제본 분리 → 캐시 도입 → 샤딩 → 마이크로서비스 각 단계의 트리거(사용자 수/트래픽/팀 규모)
- 비용 최적화 — 클라우드 비용이 폭발하는 이유(오버프로비저닝, 비효율적 쿼리), Auto Scaling 설계, Reserved Instance vs Spot Instance 선택, 스토리지 계층화(Hot/Warm/Cold)

---

총 **42개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 설계 목표와 요구사항
## 📊 규모 산정 (DAU, 초당 요청 수, 저장 용량)
## 🏗️ 개략적 설계 (High-Level Architecture)
## 🔬 핵심 컴포넌트 상세 설계
## ⚡ 병목 식별과 해결
## 🔄 데이터 흐름 (주요 시나리오별)
## ⚖️ 트레이드오프 (왜 이 기술을 선택했는가)
## 🚀 확장 전략 (10배 트래픽이 오면 어떻게 대응하는가)
## 📌 핵심 결정 요약
## 🤔 심화 질문 (면접에서 나올 수 있는 후속 질문 + 답)
```

**스타일 가이드**:
1. 모든 설계는 DAU → 초당 요청 수 → 저장 용량 계산부터 시작
2. 기술 선택에는 반드시 "왜 이 기술인가, 대안은 무엇인가, 어떤 트레이드오프인가" 포함
3. 다른 IQ Dev Lab 레포 연결 — "Redis를 쓰는 이유는 redis-deep-dive의 단일 스레드 모델 참고"
4. ASCII 아키텍처 다이어그램 항상 포함
5. 면접 컨텍스트 포함 — "면접관이 이 질문을 할 때 기대하는 답변 방향"

**용량 산정 예시**:
```
URL 단축 서비스 예시:
DAU: 100만 명
읽기:쓰기 = 10:1 (단축 URL 생성보다 리디렉션이 10배 많음)

초당 쓰기: 1,000,000 / 86,400 ≈ 12 req/s
초당 읽기: 12 × 10 = 120 req/s

5년치 URL 저장:
12 req/s × 86,400 × 365 × 5 = 1.89억 개
URL 하나당 500 bytes → 1.89억 × 500 = 약 95GB

캐시:
읽기 요청의 20%가 전체의 80% → Pareto
120 req/s × 86,400 × 20% = 207만 건/일 → 약 1GB

대역폭:
읽기: 120 req/s × 500 bytes = 60KB/s (문제 없음)
```

---

## 💡 핵심 분석 대상

```
트위터 타임라인 Fan-out 설계 의사결정:

사용자 A(팔로워 1000명)가 트윗 작성
  │
  방법 1: Fan-out on Write (Push Model)
  ├── 트윗 저장 → 팔로워 1000명의 타임라인에 즉시 배포
  ├── 장점: 읽기 O(1) — 타임라인 캐시 그냥 읽으면 됨
  ├── 단점: 쓰기 비용 O(팔로워 수)
  └── 문제: 셀러브리티(팔로워 1000만) → 1000만 번 배포 → DB/캐시 폭발

  방법 2: Fan-out on Read (Pull Model)
  ├── 트윗 저장만 → 팔로워가 타임라인 요청 시 계산
  ├── 장점: 쓰기 O(1)
  ├── 단점: 읽기 O(팔로우 수) — 1000명 팔로우 시 1000개 쿼리
  └── 문제: DAU 1억 × 타임라인 조회 → DB 쿼리 폭발

  방법 3: 하이브리드 (실제 트위터 방식)
  ├── 일반 사용자: Fan-out on Write (팔로워 < 10만)
  ├── 셀러브리티: Fan-out on Read (팔로워 >= 10만)
  └── 타임라인 조회: 캐시된 타임라인 + 셀러브리티 트윗 실시간 병합

  설계 결정의 핵심:
  → 읽기/쓰기 트레이드오프
  → 사용자 유형에 따른 다른 전략
  → 정답은 없고 트레이드오프만 있다

용량 산정 프레임워크:
  1. 사용자 기반: DAU, MAU
  2. 읽기:쓰기 비율
  3. 초당 요청 수: DAU / 86,400 × 평균 액션 수
  4. 저장 용량: 초당 쓰기 × 86,400 × 365 × N년 × 레코드 크기
  5. 캐시 크기: 일일 읽기 × 20% × 레코드 크기
  6. 네트워크 대역폭: 초당 읽기 × 응답 크기
```

---

## 📚 참고 자료

- System Design Interview – An Insider's Guide (Alex Xu) Vol.1 & Vol.2
- Designing Data-Intensive Applications (Martin Kleppmann) — 분산 시스템의 바이블
- Building Microservices, 2nd Edition (Sam Newman)
- The Art of Scalability (Martin Abbott, Michael Fisher)
- High Scalability 블로그 — http://highscalability.com/ (실제 기업 아키텍처 사례)
- AWS Architecture Center — https://aws.amazon.com/architecture/
- Netflix Tech Blog — https://netflixtechblog.com/
- Uber Engineering Blog — https://www.uber.com/en-KR/blog/engineering/

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (42개 목표)
- IQ Dev Lab 다른 레포들과 연결되는 지점 명시 (어느 설계 결정에서 어느 레포 참조)

**준비됐으면 1단계 구조 설계부터 시작해줘!**
