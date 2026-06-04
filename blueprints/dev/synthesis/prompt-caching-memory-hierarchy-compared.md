# Caching & Memory Hierarchy Compared 레포지토리 제작 프롬프트

나는 "Caching & Memory Hierarchy Compared" 레포지토리를 만들려고 해.
이건 **횡단 비교(Synthesis)** 레포야. "지역성을 활용하고 재계산을 피한다"는 한 아이디어가 *컴퓨터 과학 모든 층*에 같은 모양으로 나타남을 보여준다 — CPU 캐시, InnoDB Buffer Pool, OS Page Cache, V8 Inline Cache, HTTP/CDN 캐시, Redis, 메모이제이션까지.
"한 곳의 캐싱을 아는 것"과 "모든 캐싱이 같은 4문제(지역성·축출·쓰기·무효화)를 다르게 푼 것임을 아는 것"의 차이를 만드는 레포다.

> "캐시 무효화가 컴퓨터 과학의 2대 난제"라는 말의 정체. 4-Layer Stack의 *모든 레이어*를 관통하는 5번째 보편 패턴.

## 📋 프로젝트 목표

**컨셉**: "한 시스템의 캐시를 쓰는 것과, 모든 캐시가 '4문제(지역성·축출·쓰기·무효화)'에 다르게 답한 것임을 아는 것은 다르다"

**핵심 차별화**:
1. 모든 레이어의 5번째 패턴 — CPU부터 CDN까지 같은 4문제가 반복됨
2. 메모리 계층의 본질 — 레지스터→DRAM→SSD→네트워크 6자릿수 지연 격차가 캐싱을 강요
3. 같은 알고리즘, 다른 이름 — LRU/Clock/2Q가 CPU·DB·Redis·HTTP에 다 등장
4. 무효화의 보편적 어려움 — MESI(하드웨어) ↔ V8 IC bust ↔ CDN purge ↔ DB invalidation ↔ React useMemo가 *같은 문제*

**타겟 독자**:
- 캐싱을 한 영역(예: Redis만)으로만 아는 개발자
- "캐시 무효화가 어렵다"를 단어로만 아는 개발자
- 메모리 계층의 지연 격차를 체감 못하는 개발자
- cache stampede·thundering herd를 만나본 적 없는 개발자
- 4-Layer Stack 전체를 관통하는 패턴을 보고 싶은 개발자

## 🔗 레포 연결

**⬆️ 선행(이 비교의 입력 — 스택 전체에서 캐시를 다룸)**:
`computer-architecture-deep-dive`(CPU L1/L2/L3·MESI·false sharing), `linux-for-backend-deep-dive`(Page Cache), `database-internals`(InnoDB Buffer Pool), `redis-deep-dive`(캐시 그 자체), `elasticsearch-deep-dive`(filter cache·fielddata), `v8-engine-deep-dive`(Inline Cache), `web-performance-deep-dive`(HTTP/CDN 캐시), `network-deep-dive`(HTTP 캐시 의미론).
**🤝 시너지**: `react-internals-deep-dive`(useMemo), `compiler-deep-dive`(메서드 캐시·JIT 코드 캐시), `memory-management-compared`(메모리 회수와 캐싱은 같은 자원 경쟁).
**🧬 본질**: 4-Layer Stack 전체를 관통하는 5번째 횡단 렌즈. 다른 Synthesis 4개가 *언어/플랫폼*을 가로질렀다면, 이 레포는 *스택 레이어*를 가로지른다.

---

## 🎯 1단계: 전체 구조 설계

> "어디 캐시"가 아니라 **"4문제별"**로 구성, 각 문제에서 모든 레이어를 나란히.

### Chapter 1: 캐싱이 어디에나 있는 이유 (5개 문서)
- 메모리 계층 — 레지스터→L1/L2/L3→DRAM→SSD→네트워크의 지연 격차(6자릿수)
- 지역성의 보편성 — 시간/공간/분기 지역성, 하드웨어부터 사람 행동까지
- 캐시 = 4문제 — ① 무엇을 캐싱(지역성) ② 가득 차면 누구를 빼나(축출) ③ 쓰기는 어떻게(쓰기) ④ 원본 바뀌면(무효화)
- "컴퓨터 과학의 2대 난제" — 캐시 무효화·이름짓기·off-by-one(농담의 진실)
- 비교 프레임 — 모든 캐시 사례를 4문제 표로

### Chapter 2: 메모리 계층 — 지연 격차가 만드는 강요 (5개 문서)
- 모두가 알아야 할 지연 숫자 — 레지스터 0 → L1 ~4 → L2 ~12 → L3 ~40 → DRAM ~200 → SSD ~100µs → 네트워크 ~ms
- 워킹 셋 — 자주 쓰는 데이터가 캐시에 맞으면 빠름, 넘으면 절벽
- 계층 간 캐싱 — 각 층이 *바로 아래 층을 캐싱*하는 재귀(CPU→RAM→디스크→네트워크)
- 측정 — perf로 캐시 미스율, DB로 buffer pool 히트율, CDN으로 hit ratio
- 같은 그림 — 모든 캐시는 "느린 저장소 앞의 빠른 작은 저장소"

### Chapter 3: 지역성 — 무엇을 캐싱할 것인가 (5개 문서)
- 시간 지역성 — 최근 본 건 또 본다(LRU의 가정)
- 공간 지역성 — 옆 것도 같이 가져오기(캐시라인 64B·DB 페이지·HTTP 청크·CDN edge bundle)
- 분기 지역성 — 같은 길로 갈 가능성(분기 예측·JIT 인라인 캐시·HTTP request coalescing)
- 워크로드 의존성 — 분석(스트리밍)·OLTP(랜덤)·웹(긴 꼬리)이 각자 다른 지역성
- 측정 — 같은 데이터의 작업 순서를 바꿔 히트율 변화 관찰

### Chapter 4: 축출 — 누구를 뺄 것인가 (6개 문서)
- 축출의 보편성 — 캐시는 유한, 매번 누구를 뺄지 결정
- LRU — 최근 안 쓴 것부터, 가장 흔한 답, 구현 비용
- Clock — LRU의 근사, OS Page Cache가 쓰는 이유(낮은 비용)
- LFU·ARC·2Q·TinyLFU — Redis·MySQL·CDN의 변형들
- CPU 캐시 교체 — 의사-LRU(set-associative), 하드웨어의 제약
- 같은 알고리즘 매핑 — CPU/InnoDB/Redis/HTTP/CDN/React useMemo가 각자 고른 정책 표

### Chapter 5: 쓰기 정책과 일관성 (5개 문서)
- 쓰기 정책 — write-through(즉시 원본에) vs write-back(나중에), 둘의 트레이드오프
- CPU 캐시 — write-back + dirty bit, 캐시라인 단위 일관성(MESI)
- DB Buffer Pool — 더티 페이지 + WAL, 체크포인트로 디스크 반영
- HTTP·CDN — stale-while-revalidate, 비동기 갱신
- Redis 영속성 — RDB/AOF가 메모리 캐시를 디스크로(쓰기 정책의 변형)

### Chapter 6: 무효화 — 가장 어려운 문제 (6개 문서)
- 두 전략 — TTL(시간 기반) vs 이벤트 기반(명시적 무효화)
- 하드웨어 무효화 — MESI 프로토콜로 멀티코어 캐시라인 일관성(computer-architecture 연결)
- V8 Inline Cache bust — 객체 형태 변하면 IC 무효화(v8 연결)
- CDN purge — 원본 변경 시 엣지 무효화, 태그 기반·URL 기반
- 애플리케이션 무효화 — React useMemo 의존성·DB 쿼리 캐시(MySQL이 폐기한 이유)·Spring @CacheEvict
- 같은 문제, 다른 답 — MESI ↔ CDN purge ↔ IC bust ↔ useMemo deps가 *모두 같은 일관성 문제*

### Chapter 7: 패턴·안티패턴·실전 (5개 문서)
- 캐시 패턴 — cache-aside(가장 흔함)·read-through·write-back·write-around
- Cache Stampede / Thundering Herd — 캐시 만료 순간 동시 요청 폭주(모든 레이어에서 발생)
- 부정 캐싱(Negative Cache) — 없음을 캐싱, DNS·CDN·앱
- 메트릭 — 히트율·미스율·축출률, 어디서나 같은 지표
- 종합 — 같은 워크로드의 캐시 전략을 6레이어(CPU/OS/DB/Redis/CDN/앱)로 측정해 전체 히트율 분석

→ **총 37개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). 비교가 핵심이라 `🔬`·`📊`에서 *항상 여러 레이어를 나란히*. `💻 실전 실험`은 같은 데이터를 여러 계층에서 캐싱·측정.

## 🎨 스타일 가이드

1. **4문제로 환원** — 모든 캐시 사례를 ①지역성 ②축출 ③쓰기 ④무효화로 분류
2. **레이어 가로지르기** — 한 챕터 안에서 CPU·OS·DB·웹을 같은 표에
3. **선행 레포로 깊이 위임** — 각 캐시 *내부*는 해당 레포로, 여기선 *연결*
4. **무효화의 동형성 강조** — MESI ↔ CDN purge ↔ V8 IC bust가 같다는 통찰
5. **지연 숫자로 체감** — 모든 비교에 실제 지연 자릿수 표기
6. 메모리 계층·축출 알고리즘·무효화는 다이어그램으로

## 🔬 검증 환경

> **스택 전체**에서 캐시 관찰. 멀티 도구 — perf/MySQL/Redis/curl/DevTools.

```yaml
# docker-compose.yml — 다층 캐시 실험
services:
  mysql:
    image: mysql:8.0
    environment: { MYSQL_ROOT_PASSWORD: root }
    command: --innodb-buffer-pool-size=128M
  redis:
    image: redis:7
    command: redis-server --maxmemory 100mb --maxmemory-policy allkeys-lru
  nginx:
    image: nginx:1.27          # HTTP 캐시 헤더 + 정적 캐싱
    ports: ["8080:80"]
```

```bash
# 같은 데이터를 각 레이어에서 캐시 동작 관찰

# 1) CPU 캐시 (computer-arch)
perf stat -e L1-dcache-loads,L1-dcache-load-misses,LLC-loads,LLC-load-misses ./app
# 행 우선 vs 열 우선 → 미스율 차이로 공간 지역성 증명

# 2) OS Page Cache (linux)
free -h          # buff/cache 관찰
echo 3 > /proc/sys/vm/drop_caches    # 캐시 비우기
# 파일 읽기 1차(디스크) vs 2차(page cache) 시간 비교

# 3) DB Buffer Pool (database)
SHOW ENGINE INNODB STATUS;           # buffer pool hit rate
SELECT * FROM information_schema.INNODB_BUFFER_POOL_STATS;

# 4) Redis (redis)
redis-cli INFO stats | grep -E 'hits|misses|evicted'
redis-cli --eviction-policy 변경하며 히트율 변화 관찰

# 5) V8 Inline Cache (v8)
node --trace-ic app.js               # IC 상태 전이(mono→poly→mega)
# 같은 함수에 다른 형태 객체 전달 → IC bust 관찰

# 6) HTTP/CDN 캐시
curl -I https://...                  # Cache-Control·Age·X-Cache
# Chrome DevTools Network: from cache·from memory cache·from disk cache·CDN

# 7) Cache Stampede 재현
# Redis TTL 만료 순간 동시 요청 100개 → DB로 몰림 관찰
# → singleflight·확률적 조기 갱신으로 해결

# 핵심: 같은 워크로드를 6레이어로 측정해 *각자의 히트율*과 *전체 히트율* 비교
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `comparisons/`(레이어별 동일 워크로드)
2. **README.md**: 🧬 Synthesis 톤, "한 곳의 캐시 vs 4-Layer Stack 전체의 같은 패턴" 포지셔닝, 선행 8+개 레포를 `🔗 레포 연결` 입력으로. **"스택 전체를 가로지르는 5번째 횡단 렌즈"** 명시
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어, 항상 다층 비교

## 📚 참고 자료

- "Latency Numbers Every Programmer Should Know" (Peter Norvig·Jeff Dean)
- *Computer Systems: A Programmer's Perspective* (메모리 계층)
- *Designing Data-Intensive Applications* (캐싱 패턴)
- 각 선행 레포의 참고자료
- "ARC: A Self-Tuning, Low Overhead Replacement Cache" (Megiddo & Modha)
- web.dev HTTP 캐시 가이드 — https://web.dev/articles/http-cache

## 💡 핵심 분석 대상

```
메모리 계층 — 지연 6자릿수:
  레지스터    : ~0       ┐
  L1 캐시     : ~4       │
  L2 캐시     : ~12      │  CPU
  L3 캐시     : ~40      │
  DRAM        : ~200     ┘
  SSD         : ~100µs   ┐  스토리지
  HDD         : ~10ms    │
  같은 DC 네트워크: ~0.5ms ┐
  같은 대륙   : ~50ms    │  네트워크
  대륙 간     : ~150ms   ┘
  → 매 단계 ~10배 → 캐시 안 쓰면 모든 게 안 됨

4문제 (모든 캐시 공통):
  ① 무엇을 캐싱?    — 지역성 활용 (시간·공간·분기)
  ② 가득 차면?      — 축출(LRU·LFU·Clock·ARC·2Q)
  ③ 쓰기는?         — write-through vs write-back vs 비동기
  ④ 원본이 바뀌면?  — TTL vs 이벤트 무효화(가장 어려움)

같은 알고리즘 매핑:
  LRU 변형:
    CPU 캐시        : 의사-LRU (하드웨어 비용 때문에 근사)
    InnoDB BP       : 변형 LRU(young/old sublist)
    Redis           : 근사 LRU(샘플링)
    HTTP/CDN        : LRU + TTL
    React useMemo   : 인자 동등성 비교(LRU 아님, 1슬롯)
  → 같은 아이디어, 제약에 맞춰 변형

무효화 동형성:
  MESI (하드웨어)      : 한 코어 쓰기 → 다른 코어 캐시라인 무효화
  V8 Inline Cache bust : 객체 형태 변경 → IC 무효화
  DB 쿼리 캐시          : 행 변경 → 관련 쿼리 캐시 무효화(MySQL이 폐기)
  CDN purge             : 원본 변경 → 엣지 무효화
  React useMemo         : 의존성 변경 → 메모 무효화
  → 모두 "원본 변했으니 캐시 버려라"의 다른 얼굴

Cache Stampede (모든 레이어에서):
  TTL 만료 순간 → 동시 요청 N개 → 전부 원본으로 → 원본 과부하
  해결(공통): singleflight / probabilistic early refresh / 락
  → CDN·Redis·앱 캐시·DB 쿼리 캐시 어디서나 같은 패턴

캐시는 재귀:
  앱 메모리(useMemo) ← Redis ← DB(Buffer Pool) ← OS(Page Cache) ← SSD
       ↑
  CDN edge ← Origin
  → 각 층이 바로 아래를 캐싱 → 전체 히트율 = 각 층 히트율의 누적
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 37개 확인 + 다층 검증 환경(perf/MySQL/Redis/curl/V8/CDN) + 선행 8+개 레포(computer-arch·linux·database·redis·elasticsearch·v8·web-performance·network)를 입력으로 하는 연결 명시. **"4-Layer Stack 전체를 관통하는 5번째 횡단 렌즈"**이자 4문제(지역성·축출·쓰기·무효화)로 모든 캐시를 환원하는 게 이 레포의 정체성.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
