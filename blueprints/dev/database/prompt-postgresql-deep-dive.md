# PostgreSQL Deep Dive 레포지토리 제작 프롬프트

나는 "PostgreSQL Deep Dive" 레포지토리를 만들려고 해.
MySQL을 알고 PostgreSQL을 쓰는 것과, 왜 PostgreSQL의 MVCC가 MySQL과 근본적으로 다른지, VACUUM이 왜 존재해야 하는지, GIN 인덱스가 왜 전문 검색에 B-Tree보다 적합한지를 완전히 파헤치는 레포야.

"PostgreSQL도 SQL 쓰면 되는 거 아닌가"와 "Dead Tuple이 왜 생기고 VACUUM이 어떻게 회수하는지, TOAST가 큰 값을 어떻게 저장하는지, PostgreSQL의 MVCC가 왜 Undo Log가 아닌 Heap 내부 버전으로 구현되는지 아는 것"의 차이를 만드는 레포야.

## 📋 프로젝트 목표

**컨셉**: "MySQL을 아는 것과, PostgreSQL이 MVCC를 완전히 다른 방식으로 구현하고 그 선택이 어떤 트레이드오프를 만드는지 아는 것은 다르다"

**핵심 차별화**:
1. MVCC 구현 차이 — MySQL(Undo Log) vs PostgreSQL(Heap 내부 버전), 각 방식이 만드는 Dead Tuple과 VACUUM의 필요성
2. 인덱스 다양성 — B-Tree/Hash/GiST/GIN/BRIN/Bloom 각각의 내부 구조와 적합한 사용 사례
3. TOAST — 큰 값(텍스트, JSONB, 배열)을 투명하게 압축/분리 저장하는 방법
4. PostgreSQL만의 기능 — JSONB, 배열, 전문 검색, 윈도우 함수, CTE 내부 동작

**타겟 독자**:
- MySQL만 써왔는데 PostgreSQL을 사용해야 하는 개발자
- VACUUM을 끄거나 무시하다가 테이블 부풀음(Table Bloat) 문제를 경험한 개발자
- JSONB를 쓰지만 GIN 인덱스를 걸지 않아 느린 JSON 쿼리를 날리는 개발자
- PostgreSQL의 파티셔닝이 MySQL과 어떻게 다른지 모르는 개발자
- pg_stat_activity, pg_locks를 보고 원인을 분석 못하는 개발자
- 복잡한 분석 쿼리(Window Function, CTE, LATERAL JOIN)를 모르는 개발자

**선행 학습**:
- database-internals (B-Tree, MVCC, WAL 기초 — PostgreSQL과 비교를 위해 필수)
- mysql-deep-dive (MySQL과의 대비로 PostgreSQL의 차별점 이해)
- linux-for-backend-deep-dive (Page Cache, shared_buffers와 OS 캐시의 관계)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: PostgreSQL 아키텍처와 MySQL과의 근본 차이 (5개 문서)
- PostgreSQL vs MySQL 아키텍처 비교 — 프로세스 기반(PostgreSQL, 연결당 프로세스 fork) vs 스레드 기반(MySQL), Shared Memory(shared_buffers)와 OS Page Cache의 이중 캐시 구조, Connection Pooling(PgBouncer)이 필수인 이유
- Storage Manager와 페이지 구조 — 8KB 페이지(Heap Page) 구조, pd_lower/pd_upper/pd_special 레이아웃, 튜플(행) 저장 방식, MySQL InnoDB 페이지와의 구조 비교
- PostgreSQL MVCC vs MySQL MVCC — MySQL: 변경 시 Undo Log에 이전 버전 저장 / PostgreSQL: Heap 안에 이전 버전 튜플을 그대로 보존, Dead Tuple이 Heap에 남는 이유, 각 방식의 장단점
- WAL(Write-Ahead Log) 완전 분해 — WAL이 PostgreSQL에서 동작하는 방식, WAL 레코드 구조, Checkpoint, WAL 기반 복제(Streaming Replication), MySQL Binary Log와의 차이
- 트랜잭션 ID(XID)와 가시성 — 모든 튜플에 xmin/xmax가 있는 이유, 트랜잭션이 어떤 버전의 튜플을 볼 수 있는지 결정하는 가시성 규칙, XID Wraparound 문제와 예방

### Chapter 2: MVCC 심화와 VACUUM (7개 문서)
- Dead Tuple 완전 분해 — UPDATE/DELETE 시 기존 튜플을 즉시 삭제하지 않는 이유(다른 트랜잭션이 여전히 읽을 수 있음), Dead Tuple이 쌓이면 발생하는 Table Bloat, 페이지 당 살아있는 튜플 비율 감소가 Full Table Scan 비용을 올리는 이유
- VACUUM 내부 동작 — VACUUM이 Dead Tuple을 회수하는 단계별 흐름, Free Space Map(FSM) 업데이트, Visibility Map 업데이트, VACUUM이 Lock을 최소화하는 방법(HOT Update 활용)
- VACUUM FULL vs VACUUM — VACUUM: Dead Tuple 공간 재사용 가능하게 표시(테이블 크기 유지), VACUUM FULL: 실제 파일 크기 축소(AccessExclusiveLock 필요, 운영 중 사용 주의), 각 사용 시점
- Autovacuum 튜닝 — Autovacuum이 동작하는 조건(autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor), 테이블 크기별 튜닝 전략, 대형 테이블에서 Autovacuum이 따라오지 못하는 문제, 수동 VACUUM 스케줄링
- HOT(Heap Only Tuple) Update — 인덱스가 있는 컬럼을 UPDATE하지 않을 때 인덱스 갱신 없이 같은 페이지 내에서 버전 체인 구성, HOT 업데이트 조건과 성능 이점, Fillfactor 설정이 HOT을 늘리는 이유
- XID Wraparound — 32비트 트랜잭션 ID가 40억 개를 넘으면 과거 트랜잭션이 미래로 보이는 문제, VACUUM이 오래된 튜플을 Freeze하는 이유(xmin을 2로 설정), Wraparound 발생 시 DB 강제 종료(PostgreSQL 안전 장치), 모니터링과 예방
- 격리 수준과 스냅샷 — PostgreSQL의 Snapshot Isolation 구현, REPEATABLE READ가 MySQL보다 강한 이유(Phantom Read 방지), Serializable Snapshot Isolation(SSI)으로 직렬화 가능성 보장

### Chapter 3: 인덱스 완전 분해 (8개 문서)
- B-Tree 인덱스 심화 — PostgreSQL B-Tree가 MySQL B+Tree와 다른 점, 인덱스 페이지 구조, Index-Only Scan이 가능한 조건(Visibility Map 사용), Partial Index(WHERE 절이 있는 인덱스)의 활용
- Hash 인덱스 — 등가 비교(=)에만 사용, B-Tree보다 빠른 등가 조회, WAL 로깅 문제(PostgreSQL 10 이전), 범위 쿼리 지원 안 됨, Hash 인덱스가 적합한 경우
- GiST(Generalized Search Tree) — 기하 데이터(Point, Box, Polygon), 전문 검색, 범위 데이터 인덱싱, R-Tree를 GiST로 구현하는 방법, PostGIS와의 조합
- GIN(Generalized Inverted Index) — JSONB, 배열, 전문 검색의 원소를 역색인하는 방법, GIN이 B-Tree보다 JSONB 검색에 빠른 이유, GIN Pending List와 Fastupdate 옵션, GIN 인덱스 크기
- BRIN(Block Range INdex) — 물리적으로 정렬된 데이터(타임스탬프, 로그 ID)에 최적화, 블록 범위마다 min/max 값만 저장하는 초경량 인덱스, 테이블의 0.1% 크기로 효과적인 필터링, BRIN이 B-Tree를 대체할 수 있는 경우
- Bloom 필터 인덱스 — 여러 컬럼의 등가 비교를 하나의 인덱스로, 확률적 자료구조(False Positive 있음), Bloom 필터 인덱스 크기가 B-Tree보다 작은 이유, 적합한 사용 사례
- 인덱스 선택 가이드 — 쿼리 패턴별 최적 인덱스 선택 결정 트리, Partial Index로 인덱스 크기 줄이기, Expression Index(함수 기반 인덱스), 인덱스 유지 비용 vs 조회 이득 트레이드오프
- 실행 계획 분석 심화 — EXPLAIN ANALYZE의 각 노드(SeqScan/IndexScan/IndexOnlyScan/BitmapHeapScan/NestedLoop/HashJoin/MergeJoin) 의미, 플래너가 인덱스를 무시하는 조건, 통계 정보(pg_stats) 갱신과 플래너 정확도

### Chapter 4: TOAST와 대용량 데이터 (5개 문서)
- TOAST 완전 분해 — Oversized-Attribute Storage Technique, 8KB 페이지에 들어가지 않는 큰 값을 별도 TOAST 테이블에 저장하는 방법, 압축(pglz/lz4)과 분리 저장 전략, TOAST가 투명한 이유(애플리케이션에서 인식 불필요)
- JSONB 내부 저장 — JSONB가 JSON과 다른 이유(파싱 후 바이너리 형태로 저장), 키 정렬 저장으로 O(log N) 키 접근, JSONB 컬럼 크기와 TOAST 임계값, GIN 인덱스로 JSONB 내부 검색
- 배열(Array) 타입 — PostgreSQL 네이티브 배열 저장 방식, GIN 인덱스로 배열 원소 검색(ANY/ALL/@@), 배열 조작 함수(unnest, array_agg), 정규화 vs 배열 선택 기준
- 전문 검색(Full Text Search) — tsvector/tsquery 내부 표현, to_tsvector로 토큰 생성, GIN 인덱스로 전문 검색 가속, 한국어 전문 검색(pg_bigm), Elasticsearch와 비교
- Large Object vs TOAST — 파일 저장에 Large Object(lo_*) vs TOAST 선택 기준, pg_largeobject 테이블 내부, 파일 시스템 vs DB 저장 트레이드오프

### Chapter 5: 고급 SQL과 분석 기능 (6개 문서)
- 윈도우 함수(Window Function) 완전 분해 — OVER(PARTITION BY ... ORDER BY ...) 내부 실행 방식, 각 행마다 집계하면서 행을 유지하는 원리, ROW_NUMBER/RANK/DENSE_RANK/LAG/LEAD 동작, 윈도우 프레임(ROWS vs RANGE)
- CTE(Common Table Expression) 심화 — WITH 절이 최적화 울타리(Optimization Fence)인 이유(PostgreSQL 12 이전), MATERIALIZED vs NOT MATERIALIZED, 재귀 CTE로 트리/그래프 순회, CTE와 서브쿼리 성능 비교
- 파티셔닝 완전 분해 — Range/List/Hash 파티셔닝, 파티션 프루닝(Partition Pruning)으로 쿼리가 일부 파티션만 스캔, 파티션 인덱스 전략, PostgreSQL 파티셔닝 vs MySQL 파티셔닝 차이, 파티션 키 선택
- LATERAL JOIN — 서브쿼리가 왼쪽 테이블의 각 행을 참조하는 방법, 각 사용자의 최근 N개 주문 조회(LATERAL + LIMIT), LATERAL이 필요한 패턴
- Upsert와 SKIP LOCKED — INSERT ... ON CONFLICT DO UPDATE(Upsert) 내부 동작, 동시 Upsert 처리 시 충돌 해결, SELECT ... FOR UPDATE SKIP LOCKED로 큐 구현(작업자 여러 개가 동시에 작업을 가져가는 패턴)
- 집계 함수 심화 — GROUPING SETS/CUBE/ROLLUP으로 다차원 집계, FILTER 절로 조건부 집계, 통계 집계 함수(percentile_cont, corr), 대용량 집계 최적화

### Chapter 6: 복제와 고가용성 (5개 문서)
- Streaming Replication 완전 분해 — Primary가 WAL을 Standby에 스트리밍하는 방법, 동기(Synchronous) vs 비동기(Asynchronous) 복제 선택 기준, Replication Slot으로 WAL 보존(Standby가 느릴 때 Primary WAL 삭제 방지)
- Hot Standby — 복제 중인 Standby에서 읽기 쿼리 처리, 읽기 트래픽 분산, Standby의 쿼리가 Primary의 VACUUM과 충돌하는 문제(hot_standby_feedback 설정)
- 로직 복제(Logical Replication) — WAL 레벨 변경(physical → logical), 테이블 단위 선택적 복제, 이기종 버전 간 복제, CDC(Change Data Capture) 활용(Debezium + PostgreSQL logical replication)
- Patroni 고가용성 — Patroni가 etcd/Consul로 Leader Election을 구현하는 방법, 자동 장애조치(Failover) 절차, Split-Brain 방지(Fencing), Spring 애플리케이션에서 자동 연결 전환(PgBouncer + Patroni)
- Connection Pooling 심화 — PgBouncer Session Mode vs Transaction Mode vs Statement Mode 차이, Prepared Statement와 Transaction Mode 충돌, 애플리케이션 연결 풀(HikariCP)과 PgBouncer 조합

### Chapter 7: 성능 튜닝과 운영 (5개 문서)
- postgresql.conf 핵심 설정 — shared_buffers(총 RAM의 25%), effective_cache_size(OS 캐시 힌트), work_mem(정렬/해시 조인 메모리), maintenance_work_mem(VACUUM/인덱스 생성 메모리), max_connections와 PgBouncer
- 쿼리 성능 진단 — pg_stat_statements(느린 쿼리 누적 통계), auto_explain으로 느린 쿼리 실행 계획 자동 로깅, pg_stat_activity로 실행 중인 쿼리 확인, pg_locks로 잠금 대기 분석
- 인덱스 사용률 분석 — pg_stat_user_indexes로 인덱스 스캔 횟수 확인, 사용되지 않는 인덱스 식별, 인덱스 부풀음(Index Bloat) 측정과 REINDEX CONCURRENTLY
- 운영 중 발생하는 문제 패턴 — Long Running Transaction이 VACUUM을 막는 문제, Autovacuum이 따라오지 못하는 경우, 테이블 잠금 대기로 인한 연결 누적, Checkpoint 과부하(checkpoint_completion_target 튜닝)
- Spring + PostgreSQL 최적화 — HikariCP 최적 설정, R2DBC PostgreSQL 드라이버, Spring Data JPA와 PostgreSQL 특화 기능(JSONB 쿼리, 배열 타입), Flyway/Liquibase 마이그레이션 전략

---

총 **41개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 PostgreSQL에서 중요한가
## 😱 흔한 실수 (Before — MySQL 방식 그대로 적용하거나 PostgreSQL 특성을 무시)
## ✨ 올바른 접근 (After — PostgreSQL에 최적화된 방식)
## 🔬 내부 동작 원리 (소스 레벨 분석, 페이지 구조, 알고리즘)
## 💻 실전 실험 (psql, EXPLAIN ANALYZE, pg_stat_* 뷰 활용)
## 📊 MySQL과 비교 (같은 문제를 어떻게 다르게 해결하는가)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. MySQL과 비교를 항상 포함 — "MySQL은 이렇게 하는데 PostgreSQL은 왜 다르게 하는가"
2. psql 명령어와 pg_stat_* 시스템 뷰로 바로 재현할 수 있는 실험 포함
3. 물리적 저장 구조 항상 포함 — "실제로 디스크에 어떤 형태로 저장되는가"
4. database-internals 레포와 연결 — 기초 개념은 해당 레포 참조, PostgreSQL 특화 내용만
5. Spring Data JPA/R2DBC와 연결 — PostgreSQL 설정이 JPA 동작에 어떻게 영향 미치는가

**실험 환경**:
```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: pg_deep_dive
      POSTGRES_PASSWORD: postgres
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=ko_KR.UTF-8"
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    command: >
      postgres
      -c shared_buffers=256MB
      -c effective_cache_size=1GB
      -c work_mem=16MB
      -c maintenance_work_mem=128MB
      -c log_min_duration_statement=100
      -c log_autovacuum_min_duration=0
      -c track_io_timing=on
      -c track_functions=all

  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@admin.com
      PGADMIN_DEFAULT_PASSWORD: admin
    ports:
      - "5050:80"

  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    environment:
      DATA_SOURCE_NAME: postgresql://postgres:postgres@postgres:5432/pg_deep_dive?sslmode=disable
    ports:
      - "9187:9187"

volumes:
  pgdata:
```

```sql
-- 핵심 진단 쿼리 세트

-- Dead Tuple 현황
SELECT schemaname, tablename,
       n_live_tup, n_dead_tup,
       round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_ratio,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- 인덱스 사용률
SELECT indexrelname, idx_scan, idx_tup_read, idx_tup_fetch,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan;

-- 잠금 대기 분석
SELECT pid, now() - pg_stat_activity.query_start AS duration,
       query, state, wait_event_type, wait_event
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 seconds';

-- Table Bloat 측정
SELECT tablename,
       pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS total,
       pg_size_pretty(pg_relation_size(tablename::regclass)) AS table,
       pg_size_pretty(pg_indexes_size(tablename::regclass)) AS indexes
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(tablename::regclass) DESC;

-- 튜플 가시성 확인 (pageinspect 확장)
CREATE EXTENSION pageinspect;
SELECT t_xmin, t_xmax, t_ctid, t_infomask
FROM heap_page_items(get_raw_page('orders', 0));
```

---

## 💡 핵심 분석 대상

```
PostgreSQL MVCC vs MySQL MVCC:

MySQL (InnoDB):
  UPDATE orders SET status = 'SHIPPED' WHERE id = 1;
  → 현재 페이지의 행을 직접 업데이트
  → 이전 버전을 Undo Log에 저장
  → 트랜잭션 종료 시 Undo Log 정리
  → 테이블 페이지는 항상 최신 버전만 존재

PostgreSQL:
  UPDATE orders SET status = 'SHIPPED' WHERE id = 1;
  → 기존 튜플: xmax = 현재 트랜잭션 ID (삭제됨을 표시)
  → 새 튜플: xmin = 현재 트랜잭션 ID (생성됨을 표시)
  → 두 버전이 Heap 페이지에 공존
  → 트랜잭션 종료 후에도 Dead Tuple이 페이지에 남음

Dead Tuple 문제:
  UPDATE를 1만 번 하면?
  → 페이지에 Dead Tuple이 1만 개 쌓임
  → 테이블 크기 10배 증가 (Table Bloat)
  → Full Scan 비용 10배 증가
  → VACUUM 필요!

VACUUM 동작:
  1. Dead Tuple 스캔 (xmax가 설정된 튜플)
  2. 더 이상 필요 없는 Dead Tuple 식별
     (어떤 트랜잭션도 해당 버전을 볼 수 없는 경우)
  3. Free Space Map 업데이트 (재사용 가능 표시)
  4. Visibility Map 업데이트 (모든 튜플이 visible한 페이지)
  5. Index Vacuum (Dead Tuple 가리키는 인덱스 엔트리 제거)

  ※ VACUUM은 OS에 공간을 반환하지 않음
  ※ VACUUM FULL만 파일 크기 축소 (AccessExclusiveLock)

GIN 인덱스 vs B-Tree for JSONB:

B-Tree:
  CREATE INDEX ON orders (data);  -- data는 JSONB
  → JSON 전체를 하나의 값으로 인덱싱
  → {"name": "John"}을 찾을 수는 있음
  → {"name": "John"} 내부 키 "name" 검색 불가

GIN:
  CREATE INDEX ON orders USING GIN (data);
  → JSONB 내부 키-값 쌍을 역색인
  → data @> '{"name": "John"}' → GIN으로 O(log N)
  → 어떤 JSONB 경로도 검색 가능

XID Wraparound 위험:
  PostgreSQL의 트랜잭션 ID: 32비트 (약 40억)
  초당 1000 트랜잭션 → 약 49일이면 20억 ID 소모

  문제: XID = 1과 XID = 40억+1이 같아 보임
        과거 트랜잭션이 미래로 인식 → 데이터 손상!

  예방: VACUUM이 오래된 튜플을 Freeze (xmin = 2)
        Freeze된 튜플은 Wraparound 영향 없음

  위험 신호:
  SELECT datname, age(datfrozenxid) FROM pg_database;
  → age가 2억 이상이면 Autovacuum 즉시 점검
  → age가 10억 이상이면 수동 VACUUM FREEZE 필요
```

---

## 📚 참고 자료

- PostgreSQL 공식 문서 — https://www.postgresql.org/docs/current/
- The Internals of PostgreSQL (Hironobu Suzuki) — https://www.interdb.jp/pg/
- PostgreSQL 소스 코드 — https://github.com/postgres/postgres
- Use The Index, Luke — https://use-the-index-luke.com/ (SQL 인덱스 완전 분해)
- Cybertec PostgreSQL Blog — https://www.cybertec-postgresql.com/en/blog/
- pganalyze Blog — https://pganalyze.com/blog
- Bruce Momjian (PostgreSQL 핵심 개발자) 발표자료 — https://momjian.us/main/presentations/

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~8개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (41개 목표)
- Docker Compose 실험 환경 (PostgreSQL + pgAdmin + Exporter)
- database-internals / mysql-deep-dive / linux-for-backend-deep-dive와 연결되는 지점 명시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
