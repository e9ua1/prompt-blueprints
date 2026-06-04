# Database Deep Dive 레포지토리 제작 프롬프트

나는 "Database Deep Dive" 레포지토리를 만들려고 해.
인덱스가 왜 B-Tree인지, 쿼리 플래너가 어떻게 실행계획을 세우는지, MVCC가 트랜잭션 격리를 어떻게 구현하는지를 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "SQL을 쓰는 것과, DB가 내부에서 무슨 일을 하는지 아는 것은 다르다"

**핵심 차별화**:
1. 인덱스가 왜 B-Tree 구조인지, Clustered vs Non-Clustered 차이
2. 쿼리 옵티마이저가 실행계획을 세우는 원리
3. MVCC가 트랜잭션 격리 수준을 어떻게 구현하는지
4. 락(Lock)의 종류와 데드락 발생 원리

**타겟 독자**:
- N+1 문제를 해결했지만 왜 발생하는지 설명 못하는 개발자
- 인덱스를 걸었는데 쿼리가 느린 이유를 모르는 개발자
- `REPEATABLE_READ`와 `READ_COMMITTED` 차이를 설명 못하는 개발자
- 데드락 로그를 보고 원인을 분석 못하는 개발자
- Spring Data JPA를 쓰면서 DB 내부를 블랙박스로 두는 개발자

**선행 학습**:
- Spring Data & Transaction Deep Dive (JPA/Transaction 개념 이해)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Storage & File Structure (5개 문서)
- DB가 데이터를 디스크에 저장하는 방식 (Page, Block, Extent)
- InnoDB Buffer Pool — 메모리와 디스크 사이 캐시 레이어
- Row Format과 데이터 저장 구조
- Tablespace와 파일 구성
- WAL (Write-Ahead Logging) — 장애 복구의 핵심

### Chapter 2: Index 완전 분해 (7개 문서)
- B-Tree 인덱스 — 왜 Binary Tree가 아닌 B-Tree인가
- Clustered Index vs Non-Clustered Index
- Covering Index와 Index-Only Scan
- Composite Index — 컬럼 순서가 왜 중요한가
- Index Selectivity와 Cardinality
- 인덱스를 타지 않는 쿼리 패턴 (함수, 형변환, LIKE '%접두사')
- 인덱스 설계 전략 — 언제 걸고 언제 걸지 않는가

### Chapter 3: Query Execution & Optimizer (6개 문서)
- 쿼리 실행 단계 (Parse → Optimize → Execute)
- 실행계획 읽는 법 (EXPLAIN, EXPLAIN ANALYZE)
- Cost-Based Optimizer — 옵티마이저가 인덱스를 선택하는 기준
- Join 알고리즘 (Nested Loop, Hash Join, Sort Merge Join)
- 통계 정보 (Statistics) — 옵티마이저의 판단 근거
- 힌트(Hint)와 강제 인덱스 사용

### Chapter 4: Transaction & MVCC (7개 문서)
- ACID의 실제 구현 방법
- MVCC (Multi-Version Concurrency Control) 동작 원리
- Undo Log — MVCC가 이전 버전을 보관하는 방법
- 트랜잭션 격리 수준 4가지 — 구현 원리까지
- Phantom Read가 발생하는 조건과 방지 방법
- Consistent Read vs Current Read
- Redo Log와 트랜잭션 내구성 (Durability)

### Chapter 5: Lock & Concurrency (6개 문서)
- 락의 종류 (Shared Lock, Exclusive Lock, Intention Lock)
- Row Lock vs Table Lock — InnoDB가 락을 거는 단위
- Gap Lock과 Next-Key Lock — Phantom Read 방지 메커니즘
- 데드락 발생 원리와 감지 알고리즘
- 데드락 로그 분석과 해결 전략
- Optimistic Lock vs Pessimistic Lock 선택 기준

### Chapter 6: Performance Tuning (5개 문서)
- Slow Query 분석 방법론 (Slow Query Log, Performance Schema)
- N+1 문제 — DB 관점에서의 근본 원인
- 페이징 최적화 — OFFSET의 함정과 커서 기반 페이징
- 대용량 테이블 조회 최적화 (파티셔닝, 샤딩)
- Connection Pool 튜닝 — HikariCP와 DB 연결 관리

### Chapter 7: Replication & High Availability (4개 문서)
- MySQL Replication 원리 (Binary Log 기반)
- Read Replica — 읽기 분산과 정합성 트레이드오프
- GTID 기반 복제와 자동 페일오버
- Spring에서 Read/Write 분리 구현 (AbstractRoutingDataSource)

---

각 챕터는 **4~7개 문서**로 구성해줘. 총 **40개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 중요한가
## 😱 잘못된 이해 (Before — 블랙박스로 두는 접근)
## ✨ 올바른 이해 (After — 원리를 알고 난 접근)
## 🔬 내부 동작 원리 (MySQL/InnoDB 소스 레벨 분석)
## 💻 실전 실험 (EXPLAIN, 락 확인, 슬로우 쿼리 재현)
## 📊 성능 비교 (인덱스 유무, 격리 수준별 차이)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. EXPLAIN / EXPLAIN ANALYZE 결과 직접 분석
2. 락 확인 쿼리 (`information_schema.innodb_locks` 등)
3. 슬로우 쿼리 재현 → 원인 분석 → 해결 순서
4. Spring Data JPA와 연결 (이 DB 동작이 JPA에서 어떻게 나타나는지)
5. 실제 데이터로 실험 (테스트 데이터 INSERT 스크립트 포함)

**실험 환경**:
```yaml
# docker-compose.yml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: deep_dive
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # 테스트 데이터
    ports:
      - "3306:3306"
    command: >
      --slow_query_log=ON
      --slow_query_log_file=/var/log/mysql/slow.log
      --long_query_time=0.1
      --innodb_monitor_enable=all
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**:
   - Spring Cloud Deep Dive README 스타일 참고
   - DB 내부 구조 강조
   - JPA/Spring과의 연결 명시
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료

<Spring Core Deep Dive README를 여기에 붙여넣기>

<Spring Data & Transaction README를 여기에 붙여넣기>

위 README들을 참고해서 비슷한 구조로 Database Deep Dive 버전을 만들어줘.

**차이점**:
- 주제: MySQL InnoDB 내부 동작 원리
- 초점: 인덱스 + 트랜잭션 + 락 + 성능
- 실험: EXPLAIN 분석, 슬로우 쿼리 재현, 락 확인

---

## 💡 핵심 분석 대상

```
쿼리 실행 흐름:

애플리케이션 (JPA/JDBC)
  │
  ▼
MySQL Server Layer
  ├── Parser (SQL → Parse Tree)
  ├── Optimizer (실행계획 수립)
  │     ├── 통계 정보 조회
  │     ├── 인덱스 선택
  │     └── Join 순서 결정
  └── Executor (실행계획 실행)
        │
        ▼
InnoDB Storage Engine
  ├── Buffer Pool (메모리 캐시)
  ├── B-Tree Index 탐색
  ├── MVCC (Undo Log 기반 버전 관리)
  ├── Lock Manager (Row/Gap Lock)
  └── Disk I/O (Page 단위 읽기/쓰기)

트랜잭션 흐름:
  BEGIN
    → MVCC 스냅샷 생성
    → 쿼리 실행 (Lock 획득)
    → Undo Log 기록
  COMMIT
    → Redo Log flush
    → Lock 해제
    → Buffer Pool → Disk (비동기)

장애 시나리오:
1. 인덱스 없는 쿼리 → Full Table Scan → Slow Query
2. Dirty Read / Non-Repeatable Read / Phantom Read 격리 수준별 재현
3. 데드락 발생 → InnoDB 감지 → 롤백 → 로그 분석
4. N+1 → Slow Query Log → Fetch Join으로 해결
```

이 흐름을 완전히 분해!

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (40개 목표)
- Docker Compose 실험 환경 구성
- Spring Data JPA와 연결되는 지점 명시 (어느 문서에서 JPA와 연결하는지)

**준비됐으면 1단계 구조 설계부터 시작해줘!**
