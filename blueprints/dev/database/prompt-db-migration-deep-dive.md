# DB Migration Deep Dive 레포지토리 제작 프롬프트

나는 "DB Migration Deep Dive" 레포지토리를 만들려고 해.
`ALTER TABLE`을 직접 실행하는 것과, 스키마 변경이 버전 관리되고 팀 전체에 자동으로 적용되며, 프로덕션 배포 중 서비스 중단 없이 컬럼을 추가/삭제하고, 실패 시 안전하게 롤백하는 것은 다르다.

"로컬에서 SQL 직접 실행했고 팀원한테 Slack으로 공유했어"와 "Flyway 마이그레이션이 CI/CD에 통합되어 배포 시 자동 적용되고, 프로덕션 Lock 없이 Online DDL로 실행되며, 실패 시 명확한 복구 절차가 있는 것"의 차이를 만드는 레포야.

## 📋 프로젝트 목표

**컨셉**: "SQL을 직접 실행하는 것과, 스키마 변경을 코드처럼 관리하고 안전하게 프로덕션에 적용하는 것은 다르다"

**핵심 차별화**:
1. Flyway 내부 동작 — `flyway_schema_history` 테이블로 버전을 추적하고 체크섬으로 변경을 감지하는 원리
2. Zero-Downtime Migration — `ALTER TABLE`이 왜 프로덕션을 중단시키는가, Lock 없이 컬럼을 추가하는 방법
3. 롤백 전략 — DDL은 롤백이 없다는 전제 아래 Forward-Only 전략 설계
4. 팀 협업과 충돌 — 여러 브랜치에서 동시에 마이그레이션을 작성할 때 버전 충돌 해결

**타겟 독자**:
- DDL을 직접 실행하고 팀원에게 Slack으로 공유하는 개발자
- Flyway를 쓰지만 `spring.flyway.baseline-on-migrate=true`를 무엇인지 모르고 설정하는 개발자
- `ALTER TABLE ADD COLUMN`이 프로덕션에서 테이블 락을 걸 수 있다는 걸 모르는 개발자
- 마이그레이션 실패 시 어떻게 복구해야 할지 모르는 개발자
- MSA 환경에서 여러 서비스가 같은 DB를 공유하다가 스키마 변경 충돌이 난 개발자
- JPA `ddl-auto=update`를 프로덕션에서 쓰는 개발자

**선행 학습**:
- database-internals (MySQL InnoDB 내부 구조, DDL Lock 원리 이해 필수)
- spring-data-transaction (JPA Entity 매핑, Spring 통합 맥락)
- cicd-deep-dive (마이그레이션을 배포 파이프라인에 통합하는 맥락, 시너지 큼)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 스키마 관리의 필요성과 도구 비교 (5개 문서)
- 스키마를 코드로 관리해야 하는 이유 — 수동 DDL 실행의 문제(팀원 누락, 환경별 불일치, 이력 없음), IaC(Infrastructure as Code)와 동일한 철학, "DB도 Git으로 관리한다"
- Flyway vs Liquibase — 버전 기반(SQL 파일) vs 변경 기반(XML/YAML 변경셋) 접근의 차이, 학습 곡선, 롤백 지원 여부, 대규모 팀에서의 선택 기준
- `ddl-auto=update` 금지 이유 — Hibernate가 스키마를 자동 수정할 때의 위험(의도치 않은 컬럼 삭제, 인덱스 누락), 개발/프로덕션 환경 분리 원칙
- 마이그레이션 파일 명명 규칙 — `V{버전}__{설명}.sql` 형식, 버전 충돌 방지를 위한 타임스탬프 전략, 팀 내 협업 규칙 수립
- 마이그레이션 환경 전략 — 로컬/개발/스테이징/프로덕션 각 환경별 마이그레이션 적용 방식, 환경별 데이터 시딩 분리

### Chapter 2: Flyway 완전 분해 (6개 문서)
- Flyway 내부 동작 원리 — `flyway_schema_history` 테이블 구조(`version`, `checksum`, `success`), 시작 시 수행하는 검사 순서, 체크섬이 파일 변경을 감지하는 방법
- 마이그레이션 유형 — Versioned(`V`), Repeatable(`R`), Undo(`U`) 마이그레이션의 차이, 각각의 실행 시점과 적합한 사용 시나리오
- Flyway 설정 옵션 — `baseline-on-migrate`, `out-of-order`, `ignore-missing-migrations`, `clean-disabled` 각 옵션이 실제로 하는 일과 프로덕션에서의 주의사항
- Java 기반 마이그레이션 — `BaseJavaMigration` 구현으로 복잡한 데이터 변환, SQL로 표현 못하는 비즈니스 로직 마이그레이션(JSON 파싱, API 호출)
- Flyway Callbacks — `beforeMigrate`, `afterEachMigrate`, `afterMigrate` 콜백으로 마이그레이션 전후 작업(권한 재부여, 캐시 초기화) 자동화
- 체크섬 불일치 오류 해결 — `flyway repair`로 체크섬 재계산, 이미 적용된 마이그레이션 파일을 수정했을 때의 복구 절차, 팀에서 충돌 예방 방법

### Chapter 3: Zero-Downtime Migration 전략 (7개 문서)
- DDL이 Lock을 거는 원리 — InnoDB Online DDL의 Lock 수준(EXCLUSIVE/SHARED/NONE), `ALTER TABLE`이 테이블 전체를 잠그는 조건, 수백만 행 테이블에서 DDL의 위험
- Online DDL과 pt-online-schema-change — `ALGORITHM=INPLACE`/`COPY`/`INSTANT` 차이, Ghost(gh-ost)와 pt-osc가 프로덕션 Lock 없이 스키마를 변경하는 원리
- Expand-Contract 패턴 — Breaking Change 없이 컬럼을 변경하는 3단계(Expand: 새 컬럼 추가 → Contract: 구 컬럼 제거), 배포와 마이그레이션의 분리
- 컬럼 추가 — NOT NULL 컬럼을 기본값 없이 추가하면 왜 안 되는가, 기존 행이 있는 테이블에 컬럼을 안전하게 추가하는 순서
- 컬럼 이름 변경 — 직접 RENAME은 Breaking Change, 새 컬럼 추가 → 데이터 복사 → 애플리케이션 전환 → 구 컬럼 삭제의 단계적 전략
- 인덱스 추가 — `CREATE INDEX`가 프로덕션에서 느린 이유, `CREATE INDEX CONCURRENTLY`(PostgreSQL)와 MySQL Online DDL 비교, 대용량 테이블 인덱스 추가 전략
- 외래 키 제약 관리 — 외래 키 추가가 기존 데이터를 검증하며 Lock을 거는 원리, 데이터 정합성 검증을 애플리케이션 레벨로 이동하는 전략, 외래 키 없이 무결성 보장

### Chapter 4: 롤백 전략과 장애 복구 (5개 문서)
- DDL 롤백이 없는 이유 — MySQL DDL의 Auto-Commit 특성, 데이터 변경과 스키마 변경의 롤백 비대칭성, PostgreSQL의 Transactional DDL과의 차이
- Forward-Only 마이그레이션 전략 — "문제가 생기면 되돌리지 않고 앞으로 간다", 다음 버전에서 수정하는 패턴, 애플리케이션과 스키마의 하위 호환성 유지
- Flyway Undo 마이그레이션 — `U{버전}` 파일로 롤백 SQL 제공, 실제 사용 시나리오와 한계, 데이터가 이미 변경된 경우 Undo의 위험성
- 마이그레이션 실패 시 복구 절차 — `flyway_schema_history`의 `success=false` 레코드 처리, `flyway repair`로 실패 레코드 제거, 수동 복구 시 체크리스트
- 백업과 마이그레이션 — 대형 마이그레이션 전 DB 스냅샷, 백업 없는 마이그레이션의 위험, Point-in-Time Recovery로 되돌리는 시나리오

### Chapter 5: 팀 협업과 충돌 해결 (5개 문서)
- 마이그레이션 버전 충돌 — 두 개발자가 동시에 `V3__*.sql`을 만들었을 때, `out-of-order` 설정의 허용 범위, 타임스탬프 기반 버전(yyyyMMddHHmmss)으로 충돌 최소화
- 브랜치 전략과 마이그레이션 — Feature 브랜치에서 마이그레이션 작성 → main 병합 시 충돌 해결 절차, 장기 브랜치의 마이그레이션 누적 위험
- 코드 리뷰 체크리스트 — 마이그레이션 PR 리뷰 시 확인 사항(Lock 위험, 롤백 가능성, 대용량 데이터 처리, 인덱스 적용 여부), 마이그레이션 리뷰 자동화
- 멀티 모듈 마이그레이션 — 여러 Spring 모듈이 같은 DB를 공유하는 경우 마이그레이션 파일 위치 전략, 모듈별 스키마 분리
- MSA에서의 스키마 관리 — Database per Service 원칙에서 각 서비스가 독립적으로 마이그레이션, 서비스 간 공유 테이블 제거 전략

### Chapter 6: CI/CD 파이프라인 통합 (5개 문서)
- 배포 파이프라인에서의 마이그레이션 시점 — 애플리케이션 시작 시 자동 실행 vs 별도 Job으로 실행의 트레이드오프, 두 방식의 실패 시나리오 비교
- GitHub Actions 통합 — 마이그레이션 전용 Job 구성, 테스트 DB에 마이그레이션 적용 후 통합 테스트, 프로덕션 마이그레이션 승인 게이트
- 마이그레이션 검증 자동화 — `flyway validate`로 적용 전 체크섬 검증, 드라이 런(Dry Run)으로 SQL 미리 확인, 스테이징 환경 선 적용 후 프로덕션 적용
- Kubernetes 배포와 마이그레이션 — Init Container로 마이그레이션 먼저 실행 후 앱 시작, Job 리소스로 마이그레이션 분리 실행, 실패 시 Pod 재시작 방지
- 마이그레이션 감사(Audit) — 누가 언제 어떤 마이그레이션을 실행했는지 추적, `flyway_schema_history` 보존 정책, 컴플라이언스 리포트

### Chapter 7: 실전 시나리오와 Spring 통합 (5개 문서)
- Spring Boot + Flyway 자동 설정 — Auto-configuration 동작 원리, `FlywayMigrationStrategy` 커스터마이징, 테스트 환경에서 `@FlywayTest` 사용
- 테스트 데이터 관리 — `R__test_data.sql` Repeatable 마이그레이션으로 환경별 시드 데이터, Testcontainers + Flyway 조합, 테스트 격리 전략
- JPA Entity와 마이그레이션 동기화 — Entity 변경과 마이그레이션 파일을 함께 커밋하는 규칙, `validate` 모드로 불일치 감지, 스키마 우선 vs Entity 우선 워크플로우
- 대용량 데이터 마이그레이션 — 수천만 건 데이터 변환을 단일 UPDATE로 하면 안 되는 이유, 배치 처리(Chunk 단위), 백그라운드 마이그레이션 패턴
- 실전 케이스 스터디 — 컬럼 이름 변경 완전 가이드(Expand-Contract 전 과정), 복합 인덱스 추가 무중단 적용, 테이블 분리 마이그레이션 실전

---

각 챕터는 **5~7개 문서**로 구성해줘. 총 **38개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 실무에서 중요한가
## 😱 흔한 실수 (Before — 수동 DDL 실행하는 접근)
## ✨ 올바른 접근 (After — 마이그레이션을 코드로 관리하는 접근)
## 🔬 내부 동작 원리 (Flyway 체크섬, InnoDB DDL Lock 분석)
## 💻 실전 실험 (Flyway CLI, gh-ost, MySQL Lock 재현)
## 📊 성능/비용 비교 (Lock 시간, 마이그레이션 소요 시간)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 모든 마이그레이션 예시는 실제 실행 가능한 SQL 파일
2. DDL Lock 위험은 반드시 실험으로 재현 (information_schema.innodb_trx)
3. database-internals와 연결 — InnoDB Online DDL이 어떻게 동작하는지 심화
4. CI/CD와 연결 — 마이그레이션이 배포 파이프라인 어디에 위치해야 하는지
5. Before(수동)/After(Flyway 자동화) 코드와 워크플로우 비교 필수

**실험 환경**:
```yaml
# docker-compose.yml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: migration_test
    ports:
      - "3306:3306"
    command: >
      --innodb_lock_wait_timeout=5
      --general_log=ON
      --general_log_file=/var/log/mysql/general.log

  flyway:
    image: flyway/flyway:9
    volumes:
      - ./migrations:/flyway/sql
    environment:
      FLYWAY_URL: jdbc:mysql://mysql:3306/migration_test
      FLYWAY_USER: root
      FLYWAY_PASSWORD: root
    command: migrate
    depends_on:
      - mysql

  spring-app:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/migration_test
      SPRING_FLYWAY_ENABLED: "true"
      SPRING_JPA_HIBERNATE_DDL_AUTO: validate
    depends_on:
      - mysql
    ports:
      - "8080:8080"
```

```sql
-- migrations/V1__create_orders.sql
CREATE TABLE orders (
    id         BIGINT       NOT NULL AUTO_INCREMENT,
    user_id    BIGINT       NOT NULL,
    status     VARCHAR(20)  NOT NULL DEFAULT 'PENDING',
    created_at DATETIME(6)  NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
    PRIMARY KEY (id),
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- migrations/V2__add_total_amount.sql (Zero-Downtime 컬럼 추가)
-- Step 1: Nullable로 추가 (Lock 최소화)
ALTER TABLE orders ADD COLUMN total_amount BIGINT NULL;

-- migrations/V3__backfill_total_amount.sql
-- Step 2: 배치로 데이터 채우기 (Lock 없이)
UPDATE orders SET total_amount = 0 WHERE total_amount IS NULL LIMIT 1000;

-- migrations/V4__not_null_total_amount.sql
-- Step 3: NOT NULL 제약 추가 (이미 데이터가 있으므로 빠름)
ALTER TABLE orders MODIFY COLUMN total_amount BIGINT NOT NULL DEFAULT 0;
```

```bash
# Flyway 명령어 세트

# 마이그레이션 상태 확인
flyway -url=jdbc:mysql://localhost:3306/migration_test \
  -user=root -password=root info

# 마이그레이션 실행
flyway migrate

# 체크섬 불일치 수정
flyway repair

# 마이그레이션 검증 (적용 전 확인)
flyway validate

# DDL Lock 모니터링 (MySQL)
SELECT * FROM information_schema.innodb_trx;
SELECT * FROM performance_schema.metadata_locks WHERE OBJECT_TYPE='TABLE';

# gh-ost로 Online Schema Change
gh-ost \
  --host=localhost --user=root --password=root \
  --database=migration_test --table=orders \
  --alter="ADD COLUMN notes VARCHAR(255)" \
  --execute
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**:
   - database-internals README 스타일 참고
   - "SQL 직접 실행과 마이그레이션 코드 관리의 차이"라는 포지셔닝 강조
   - database-internals, spring-data-transaction, cicd-deep-dive와의 연결 지점 명시
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료

- Flyway 공식 문서 — https://documentation.red-gate.com/fd/
- Liquibase 공식 문서 — https://docs.liquibase.com/
- gh-ost (GitHub Online Schema Migrations) — https://github.com/github/gh-ost
- Evolutionary Database Design (Martin Fowler) — https://martinfowler.com/articles/evodb.html
- MySQL Online DDL 공식 문서 — https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl.html
- Designing Data-Intensive Applications (Martin Kleppmann) — 스키마 진화 챕터

---

## 💡 핵심 분석 대상

```
Flyway 시작 시 내부 동작:

Spring Boot 시작
  │
  ▼ FlywayAutoConfiguration 활성화
  │
  ▼ flyway_schema_history 테이블 존재 확인
  │   없으면 생성
  │
  ▼ classpath:db/migration/*.sql 파일 스캔
  │
  ▼ 각 파일의 체크섬 계산 (CRC32)
  │
  ▼ flyway_schema_history와 비교
  │   ├── version 없음 → 새 마이그레이션 → 실행
  │   ├── version 있고 체크섬 일치 → 이미 적용됨 → 건너뜀
  │   └── version 있고 체크섬 불일치 → 오류! 파일 변경 감지
  │
  ▼ 미적용 마이그레이션 순서대로 실행
  │
  ▼ 실행 후 flyway_schema_history에 기록
  │   (version, script, checksum, installed_on, success)

DDL Lock 위험 시나리오:

MySQL 8.0, orders 테이블 1억 건
  ↓
개발자: ALTER TABLE orders ADD COLUMN address VARCHAR(255) NOT NULL DEFAULT '';
  ↓
MySQL의 선택:
  ALGORITHM=COPY: 임시 테이블에 전체 복사 → 수십 분간 Write Lock
  ALGORITHM=INPLACE: 테이블 재구성 없이 메타데이터만 변경 → Lock 짧음
  ALGORITHM=INSTANT: 메타데이터만 변경, 즉시 완료 (MySQL 8.0.29+)

안전한 Zero-Downtime 순서:
  V1: ADD COLUMN address VARCHAR(255) NULL          ← INSTANT (순간)
  V2: UPDATE ... SET address='' WHERE id BETWEEN... ← 배치 (Lock 없이)
  V3: MODIFY COLUMN address VARCHAR(255) NOT NULL   ← 빠름 (이미 데이터 있음)

Expand-Contract 패턴:

  1단계 (Expand) — 새 컬럼 추가, 두 컬럼 동시 쓰기:
    V1: ALTER TABLE users ADD COLUMN full_name VARCHAR(200);
    코드: user.setName(v); user.setFullName(v);  // 둘 다 씀

  2단계 (Migrate) — 기존 데이터 채우기:
    V2: UPDATE users SET full_name = name WHERE full_name IS NULL;

  3단계 (Contract) — 구 컬럼 제거, 새 컬럼만 사용:
    V3: ALTER TABLE users DROP COLUMN name;
    코드: user.setFullName(v);  // 새 컬럼만 사용
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (38개 목표)
- Docker Compose 실험 환경 (MySQL + Flyway CLI + Spring App)
- database-internals, spring-data-transaction, cicd-deep-dive와 연결되는 지점 명시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
