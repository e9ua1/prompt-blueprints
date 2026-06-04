# MySQL Deep Dive 레포지토리 제작 프롬프트

나는 "MySQL Deep Dive" 레포지토리를 만들려고 해.
쿼리 최적화를 위한 실행계획 심화 분석, 파티셔닝·샤딩 전략, MySQL Replication 원리, 백업/복구 전략, 보안과 운영까지 완전히 파헤치는 레포를 만들 거야.

Real MySQL 2권에서 다루는 실무 운영 영역이 핵심이야.
database-internals(인덱스/MVCC/락/기본 원리)를 선행한 독자가 "그래서 실무에서 어떻게 쓰나?"를 배우는 레포야.

## 📋 프로젝트 목표

**컨셉**: "MySQL을 운영하는 것과, 장애 상황에서 원인을 찾고 튜닝하는 것은 다르다"

**핵심 차별화**:
1. 쿼리 최적화 심화 — 서브쿼리 변환, 세미조인, 파생 테이블이 내부에서 어떻게 처리되는가
2. 파티셔닝 — 언제 쓰고, 프루닝이 어떻게 동작하고, 왜 함정이 있는가
3. Replication — Binary Log 포맷(ROW/STATEMENT/MIXED) 차이와 지연 원인 분석
4. 백업/복구 — mysqldump vs XtraBackup 원리 차이, 특정 시점 복구(PITR)

**타겟 독자**:
- EXPLAIN을 읽을 줄 알지만 실행계획을 개선하는 법을 모르는 개발자
- 파티셔닝을 걸었는데 왜 빨라지지 않는지 모르는 개발자
- Replication Lag이 왜 발생하는지 설명 못하는 개발자
- 장애 시 DB를 어디까지 복구할 수 있는지 모르는 개발자
- MySQL 운영을 개발자 관점에서 이해하고 싶은 개발자

**선행 학습**:
- database-internals (인덱스, MVCC, 락, EXPLAIN 기초 필수)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Query Optimization 심화 (7개 문서)
- 서브쿼리 최적화 — IN/EXISTS/스칼라 서브쿼리가 내부에서 어떻게 변환되는가
- 세미조인(Semi-Join) 최적화 — 옵티마이저가 IN 서브쿼리를 Join으로 바꾸는 원리
- 파생 테이블(Derived Table)과 CTE — 임시 테이블이 생성되는 조건
- GROUP BY / ORDER BY 최적화 — 인덱스가 정렬을 대체하는 조건
- 윈도우 함수(Window Function) 실행 원리와 성능 특성
- 문자셋과 콜레이션 — 조인 시 묵시적 형변환이 인덱스를 망치는 이유
- 실행계획 개선 사례 (type: ALL → ref, Extra: Using filesort 제거)

### Chapter 2: 파티셔닝 (Partitioning) (6개 문서)
- 파티셔닝이 필요한 상황과 오해 — 인덱스와 파티셔닝은 다르다
- 파티션 종류 (RANGE, LIST, HASH, KEY) — 각각 어떤 상황에 쓰는가
- 파티션 프루닝(Partition Pruning) — 쿼리가 파티션을 건너뛰는 조건
- 파티션과 인덱스 — 로컬 인덱스 vs 글로벌 인덱스 차이
- 파티션의 함정 — PRIMARY KEY 제약, 외래키 불가, 파티션 키 선택 실수
- 파티션 운영 — 파티션 추가/삭제/재구성, 파티션 테이블 마이그레이션

### Chapter 3: Replication 원리 (7개 문서)
- MySQL Replication 아키텍처 — Source/Replica 구성과 Binary Log 흐름
- Binary Log 포맷 3가지 (STATEMENT vs ROW vs MIXED) — 각각의 장단점과 선택 기준
- GTID(Global Transaction ID) 기반 복제 — 장애 복구 시 왜 GTID가 중요한가
- Replication Lag 발생 원인 — IO Thread vs SQL Thread 병목 분석
- Semi-Synchronous Replication — 동기/비동기 복제의 트레이드오프
- 병렬 복제(Parallel Replication) — Replica가 지연을 따라잡는 메커니즘
- Spring에서 Read/Write 분리 — AbstractRoutingDataSource, Replica 지연 대응 전략

### Chapter 4: 백업과 복구 (5개 문서)
- mysqldump — 논리적 백업의 동작 원리와 한계 (--single-transaction 의미)
- XtraBackup — 물리적 백업의 원리, InnoDB Hot Backup이 가능한 이유
- Binary Log를 이용한 PITR (Point-In-Time Recovery) — 특정 시각으로 복구
- 복구 전략 설계 — RTO/RPO 개념과 백업 주기 결정
- 실전 장애 복구 시나리오 — 실수로 DELETE된 테이블 복구 과정

### Chapter 5: 스키마 설계와 데이터 타입 (5개 문서)
- 데이터 타입 선택 전략 — INT vs BIGINT, DATETIME vs TIMESTAMP, CHAR vs VARCHAR
- JSON 타입과 Generated Column — MySQL 8.0의 JSON 인덱싱 원리
- 정규화 vs 비정규화 — 조인 비용과 데이터 중복의 트레이드오프
- AUTO_INCREMENT의 함정 — 갭(Gap), 재사용 불가, 분산 환경에서의 한계
- 대용량 스키마 변경 — Online DDL vs pt-online-schema-change 비교

### Chapter 6: 성능 모니터링과 진단 (5개 문서)
- Performance Schema 완전 가이드 — 어떤 쿼리가, 어디서, 얼마나 걸리는가
- sys 스키마 — 운영자를 위한 DBA 뷰 모음 활용법
- InnoDB 상태 분석 (`SHOW ENGINE INNODB STATUS`) — 출력 항목 완전 해석
- MySQL 8.0 새 기능 — 히스토그램 통계, Invisible Index, 쿼리 힌트 개선
- 운영 중 발생하는 문제 패턴 — Metadata Lock 대기, Buffer Pool 부족, 디스크 풀

### Chapter 7: 보안과 사용자 관리 (3개 문서)
- 사용자와 권한 설계 — 최소 권한 원칙, Role 기반 권한 관리
- 연결 보안 — SSL/TLS 설정, 비밀번호 정책, 감사 로그
- 개발/운영 환경 분리 전략 — 접근 제어, 마스킹, 민감 데이터 관리

---

각 챕터는 **3~7개 문서**로 구성해줘. 총 **38개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 실무에서 중요한가
## 😱 흔한 실수 (Before — 잘못된 운영/설계 패턴)
## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)
## 🔬 내부 동작 원리 (MySQL 내부 메커니즘 분석)
## 💻 실전 실험 (재현 가능한 시나리오와 쿼리)
## 📊 성능/비용 비교 (수치로 보는 차이)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. "이론 → 재현 → 분석 → 해결" 순서로 전개
2. 실행계획(EXPLAIN) 개선 전후 비교 (type, Extra 컬럼 변화)
3. Replication 실습은 Docker Compose Source + Replica 구성으로
4. 백업/복구 실험은 실제 데이터 삭제 후 복구 과정까지
5. database-internals 내용을 "전제"로 두고 심화 확장

**실험 환경**:
```yaml
# docker-compose.yml
services:
  mysql-source:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: deep_dive
    volumes:
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
      - ./source.cnf:/etc/mysql/conf.d/source.cnf
    ports:
      - "3306:3306"

  mysql-replica:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
    volumes:
      - ./replica.cnf:/etc/mysql/conf.d/replica.cnf
    ports:
      - "3307:3306"
    depends_on:
      - mysql-source

# source.cnf
# [mysqld]
# server-id=1
# log-bin=mysql-bin
# binlog-format=ROW
# gtid-mode=ON
# enforce-gtid-consistency=ON
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**:
   - database-internals README 스타일 참고
   - "선행: database-internals" 명시
   - 실무 운영 관점 강조
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료

<database-internals README를 여기에 붙여넣기>

<Spring Data & Transaction README를 여기에 붙여넣기>

위 README들을 참고해서 비슷한 구조로 MySQL Deep Dive 버전을 만들어줘.

**차이점**:
- 주제: 쿼리 최적화 심화 + 파티셔닝 + Replication + 백업/복구 + 운영
- 초점: 내부 원리보다 "실무에서 왜 이게 문제가 되는가"
- 실험: Source-Replica 구성, PITR 복구, 실행계획 개선 전후 비교

---

## 💡 핵심 분석 대상

```
쿼리 최적화 흐름:

느린 쿼리 감지 (Slow Query Log / Performance Schema)
  │
  ▼
EXPLAIN ANALYZE로 실행계획 분석
  ├── type: ALL → Full Table Scan 확인
  ├── Extra: Using filesort → 정렬에 인덱스 미사용
  ├── Extra: Using temporary → 임시 테이블 생성
  └── rows: 대량 → 인덱스 선택성 문제
        │
        ▼
원인 파악
  ├── 인덱스 없음 → 추가
  ├── 인덱스 있지만 안 탐 → 형변환/함수 제거
  ├── 서브쿼리 → 세미조인 / CTE로 변환
  └── 정렬 → 인덱스 순서와 ORDER BY 일치
        │
        ▼
EXPLAIN ANALYZE 재실행 → 개선 확인

Replication 지연 분석:
  SHOW SLAVE STATUS
  ├── Seconds_Behind_Master: 지연 초
  ├── IO Thread: Binary Log 수신 상태
  └── SQL Thread: Relay Log 재생 상태
        │
  지연 원인 분석:
  ├── 대용량 트랜잭션 → 병렬 복제 설정
  ├── DDL → Metadata Lock 대기
  └── 네트워크 지연 → Semi-Sync 비활성화 검토

장애 시나리오:
1. 실수로 테이블 DROP → Binary Log로 PITR 복구
2. Replication Lag 100초 → 병렬 복제로 해소
3. 서브쿼리 IN → 세미조인으로 변환 → 10배 개선
4. 파티션 프루닝 안 됨 → 파티션 키 조건 추가 → 해소
```

이 흐름을 완전히 분해!

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (3~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (38개 목표)
- Docker Compose 실험 환경 (Source + Replica 포함)
- database-internals와 연결되는 지점 명시 (어느 문서가 어느 내용을 전제로 하는지)

**준비됐으면 1단계 구조 설계부터 시작해줘!**
