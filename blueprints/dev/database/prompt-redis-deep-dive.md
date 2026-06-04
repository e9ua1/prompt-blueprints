# Redis Deep Dive 레포지토리 제작 프롬프트

나는 "Redis Deep Dive" 레포지토리를 만들려고 해.
Redis를 캐시로 쓰는 것과, 데이터가 메모리에서 어떻게 저장되고 만료되는지, Cluster가 어떻게 샤딩을 처리하는지, Cache Stampede가 왜 발생하고 어떻게 막는지를 완전히 파헤치는 레포를 만들 거야.

"@Cacheable 붙이면 Redis에 저장되겠지"와 "직렬화 방식이 왜 성능에 영향을 주는지, maxmemory-policy가 데이터를 어떻게 지우는지 아는 것"의 차이를 만드는 레포야.

## 📋 프로젝트 목표

**컨셉**: "Redis를 캐시로 쓰는 것과, 데이터가 어떻게 저장되고 만료되는지 아는 것은 다르다"

**핵심 차별화**:
1. 단일 스레드 모델 — Redis가 왜 빠른가, 어디서 병목이 발생하는가
2. 데이터 구조 내부 — ziplist/skiplist/SDS 등 실제 인코딩과 메모리 효율
3. 영속성 완전 분해 — RDB fork() 원리, AOF fsync 정책별 내구성 차이
4. 캐싱 패턴의 함정 — Cache Stampede, Hot Key, Cache Coherence 문제

**타겟 독자**:
- @Cacheable을 쓰지만 TTL 만료 시 무슨 일이 일어나는지 모르는 개발자
- Redis를 캐시로만 쓰고 분산 락, Pub/Sub, Stream은 쓸 줄 모르는 개발자
- maxmemory 설정 없이 Redis를 운영하다 OOM을 경험한 개발자
- Cluster와 Sentinel의 차이를 설명 못하는 개발자
- Cache Hit Rate가 낮아도 원인을 찾을 수 없는 개발자
- Spring Data Redis, Spring Cache를 쓰면서 Redis 내부를 블랙박스로 두는 개발자

**선행 학습**:
- 없음 (이 레포 자체가 독립 완결)
- spring-boot-internals (Spring Auto-configuration과 RedisAutoConfiguration 연결 시 시너지)
- spring-data-transaction (Spring Cache 추상화, @Transactional과 Redis 조합 시 시너지)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Redis 내부 아키텍처 (5개 문서)
- Redis 단일 스레드 이벤트 루프 — 왜 빠른가, I/O multiplexing(epoll/kqueue)으로 동시 연결 처리
- 메모리 관리 — jemalloc 할당기, 메모리 단편화(fragmentation) 발생과 해결, maxmemory 정책 8가지
- Redis 객체 시스템 — redisObject 구조체, type과 encoding 분리, 인코딩 자동 전환 조건
- 키 만료 메커니즘 — Lazy 만료 vs Active 만료(주기적 샘플링), 만료 키가 메모리에 남는 경우
- Redis 프로세스 모델 변화 — I/O Threads(Redis 6.0), Threaded I/O가 단일 스레드 모델과 공존하는 방식

### Chapter 2: 데이터 구조 내부 구현 (7개 문서)
- String — SDS(Simple Dynamic String) 구조, 사전 할당으로 재할당 횟수 줄이기, int/embstr/raw 인코딩 전환
- List — ziplist → quicklist 전환 조건(list-max-ziplist-size), 양방향 연결 리스트와 연속 메모리의 트레이드오프
- Hash — ziplist → hashtable 전환 조건, 점진적 rehashing(리해싱 중에도 읽기/쓰기 가능한 이유)
- Set — intset → hashtable 전환 조건, intset이 정수를 압축 저장하는 방식
- Sorted Set — ziplist → skiplist+hashtable 전환 조건, 스킵 리스트가 O(log N)을 보장하는 원리
- Bitmap, HyperLogLog, Geo — 각 자료구조의 내부 인코딩과 실제 메모리 사용량, 적합한 사용 사례
- Stream — Consumer Group 내부 구조, PEL(Pending Entry List), Kafka와의 포지셔닝 비교

### Chapter 3: 영속성 (Persistence) 완전 분해 (5개 문서)
- RDB 스냅샷 — BGSAVE가 fork()로 Copy-On-Write를 활용해 일관성을 보장하는 원리, fork 비용과 메모리 사용
- AOF(Append Only File) — fsync 정책 3가지(always/everysec/no)별 내구성과 성능 트레이드오프, AOF Rewrite 원리
- RDB + AOF 혼합 모드 — Redis 4.0 혼합 포맷의 장단점, 실제 복구 시나리오
- 장애 복구 절차 — RDB/AOF 파일로 데이터를 복원하는 전 과정, 부분 손실 허용 범위 계산
- 영속성 설정 실전 가이드 — 서비스 특성별(캐시 전용 vs 데이터 저장소) 최적 설정 조합

### Chapter 4: 복제와 고가용성 (5개 문서)
- Master-Replica 복제 — PSYNC 프로토콜 내부, 전체 재동기화 vs 부분 재동기화 판단 기준
- Redis Sentinel — 자동 장애감지(Quorum), Failover 절차, 클라이언트가 새 Master를 찾는 방법
- Redis Cluster — 해시 슬롯 16384개 분산 원리, 클러스터 노드 간 Gossip 프로토콜
- 클러스터 리샤딩과 마이그레이션 — CLUSTER SLOTS 재배치, 슬롯 이동 중 요청 처리(ASK/MOVED 리다이렉션)
- 복제 일관성과 트레이드오프 — WAIT 명령어로 복제 확인, 동기 복제가 없는 Redis에서 데이터 유실 허용 범위

### Chapter 5: 캐싱 패턴과 고급 활용 (6개 문서)
- 캐싱 전략 비교 — Cache-Aside vs Write-Through vs Write-Back vs Read-Through 각각의 구현과 일관성
- Cache Stampede(Cache Thundering Herd) — 동시 캐시 미스로 DB 과부하, Mutex Lock/PER(Probabilistic Early Recomputation) 해결법
- Hot Key 문제 — 특정 키에 트래픽 집중, 읽기 Hot Key와 쓰기 Hot Key의 해결 방법 차이
- 분산 락(Distributed Lock) — SET NX EX 구현, Redlock 알고리즘 원리, Redlock의 논란(Martin Kleppmann vs Antirez)
- Pub/Sub vs Stream — 각각의 메시지 보관 여부, 소비자 그룹 지원 여부, 실무 선택 기준
- Pipeline과 MULTI/EXEC — 네트워크 왕복 최소화, 트랜잭션과 파이프라인의 차이(원자성 보장 범위)

### Chapter 6: 성능 튜닝과 운영 (5개 문서)
- SLOWLOG와 병목 찾기 — slowlog-log-slower-than 설정, OBJECT ENCODING으로 메모리 인코딩 확인
- Lua 스크립트 — 원자적 복합 연산 구현, EVALSHA로 스크립트 캐싱, Lua 스크립트 블로킹 주의사항
- 메모리 최적화 실전 — 데이터 구조별 threshold 튜닝(hash-max-ziplist-entries 등), OBJECT FREQ로 접근 패턴 분석
- Redis 모니터링 — INFO 섹션별 핵심 지표, redis-exporter + Prometheus + Grafana 대시보드 구성
- 운영 중 발생하는 문제 패턴 — OOM 발생, 복제 지연 누적, 클러스터 FAIL 상태, fork 지연으로 인한 latency spike

### Chapter 7: Spring과 Redis 통합 (4개 문서)
- Spring Data Redis — RedisTemplate 직렬화 전략(JdkSerializationRedisSerializer vs Jackson2JsonRedisSerializer vs StringRedisSerializer) 비교
- Spring Cache 추상화 — @Cacheable/@CacheEvict/@CachePut 내부 AOP 동작, CacheManager 구성
- Spring Session — HttpSession을 Redis에 저장하는 방식, 세션 직렬화 전략, 세션 클러스터링 구현
- 분산 환경 패턴 — Redisson을 활용한 분산 락/세마포어, Spring Batch와 Redis ItemReader 조합

---

각 챕터는 **4~7개 문서**로 구성해줘. 총 **37개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 실무에서 중요한가
## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)
## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)
## 🔬 내부 동작 원리 (Redis 소스 레벨 분석, 메모리 레이아웃)
## 💻 실전 실험 (redis-cli, DEBUG OBJECT, OBJECT ENCODING 등)
## 📊 성능/비용 비교 (인코딩별 메모리, 직렬화 방식별 속도 등)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 모든 데이터 구조 설명은 "실제 메모리 레이아웃이 어떻게 생겼는가"로 연결
2. redis-cli로 바로 재현할 수 있는 실험 포함 (OBJECT ENCODING, DEBUG OBJECT, MEMORY USAGE)
3. 각 설계 결정의 이유 명시 ("왜 single-threaded인가", "왜 skiplist인가")
4. Spring 애플리케이션 관점 연결 — 직렬화 전략 선택이 왜 중요한지, @Cacheable의 키 생성 전략
5. 운영 장애 시나리오 → redis-cli 진단 명령어 세트

**실험 환경**:
```yaml
# docker-compose.yml
services:
  redis:
    image: redis:7.0
    command: redis-server /etc/redis/redis.conf
    volumes:
      - ./redis.conf:/etc/redis/redis.conf
      - redis-data:/data
    ports:
      - "6379:6379"

  redis-replica:
    image: redis:7.0
    command: redis-server --replicaof redis 6379
    depends_on:
      - redis

  redis-sentinel:
    image: redis:7.0
    command: redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./sentinel.conf:/etc/redis/sentinel.conf
    depends_on:
      - redis
      - redis-replica

  redis-exporter:
    image: oliver006/redis_exporter
    environment:
      REDIS_ADDR: redis:6379
    ports:
      - "9121:9121"

volumes:
  redis-data:
```

```conf
# redis.conf 핵심 설정
maxmemory 256mb
maxmemory-policy allkeys-lru
save 900 1
save 300 10
appendonly yes
appendfsync everysec
slowlog-log-slower-than 10000
```

```bash
# 실험용 공통 명령어
# 인코딩 확인
redis-cli OBJECT ENCODING key_name

# 메모리 사용량
redis-cli MEMORY USAGE key_name

# 만료 키 통계
redis-cli INFO keyspace

# 슬로우 로그
redis-cli SLOWLOG GET 10

# 실시간 명령 모니터링 (주의: 운영 환경 부하)
redis-cli MONITOR

# 메모리 분석
redis-cli --bigkeys
redis-cli --memkeys
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**:
   - mysql-deep-dive README 스타일 참고
   - "단순 캐시 사용을 넘어 Redis 내부를 이해한다"는 차별화 강조
   - Spring Boot/Spring Cache/Spring Session과의 연결 지점 명시
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료 섹션

README 작성 시 다음을 Reference로 포함해줘:
- Redis 공식 문서 — https://redis.io/docs/
- Redis 소스 코드 (GitHub) — https://github.com/redis/redis
- Redis in Action (Josiah L. Carlson)
- Designing Data-Intensive Applications (Martin Kleppmann) — Redis 관련 챕터
- Antirez 블로그 — http://antirez.com (Redis 설계 결정 이유 원문)

---

## 💡 핵심 분석 대상

```
Redis 요청 처리 흐름:

클라이언트 (Spring RedisTemplate)
  │
  ▼ TCP 연결 (커넥션 풀 — Lettuce/Jedis)
  ▼
Redis 이벤트 루프 (단일 스레드)
  ├── epoll/kqueue로 소켓 이벤트 감지
  ├── 명령어 파싱 (RESP 프로토콜)
  ├── 명령어 실행 (O(1) ~ O(N))
  │     ├── 데이터 구조 접근 (메모리)
  │     ├── 키 만료 확인 (Lazy Expiry)
  │     └── AOF 쓰기 (appendfsync 정책에 따라)
  └── 응답 전송
        │
        ▼ (복제 활성화 시)
        Replica에게 명령어 전파 (비동기)

@Cacheable 동작 흐름:
  @Cacheable("products") 호출
    → CacheAspect.invoke()
    → CacheManager.getCache("products")
    → RedisCache.get(cacheKey)
    → RedisTemplate.opsForValue().get(serializedKey)
    → 캐시 미스 → 실제 메서드 실행
    → RedisCache.put(key, value) with TTL
    → 직렬화 (설정된 RedisSerializer)
    → Redis SET key value EX ttl

Cache Stampede 발생:
  TTL 만료 순간 → 100개 스레드 동시 캐시 미스
  → 100개 DB 쿼리 동시 발생
  → DB 과부하

  해결책 비교:
  1. Mutex Lock: 첫 번째 스레드만 DB 조회 → 나머지 대기
     단점: 락 대기 지연
  2. PER: TTL 만료 전 확률적 조기 갱신
     장점: 락 없이 자연스러운 분산
  3. 외부 캐시 갱신 (별도 프로세스): 애플리케이션에서 캐시 미스 없음

메모리 관리:
  maxmemory 설정 없음 → OOM Killer가 Redis 프로세스 종료
  maxmemory 설정 있음 → maxmemory-policy에 따라 키 제거
    noeviction: 쓰기 거부 (에러 반환)
    allkeys-lru: 전체 키 중 최근 미사용 키 제거
    volatile-lru: TTL 있는 키 중 최근 미사용 키 제거
    → 서비스 특성에 따른 정책 선택이 핵심
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (37개 목표)
- Docker Compose 실험 환경 (Master + Replica + Sentinel 포함)
- Spring Boot/Spring Cache와 연결되는 지점 명시 (어느 문서에서 어느 레포와 연결하는지)

**준비됐으면 1단계 구조 설계부터 시작해줘!**
