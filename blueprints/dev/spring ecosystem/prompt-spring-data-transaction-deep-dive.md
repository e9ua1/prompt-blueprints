# Spring Data & Transaction Deep Dive 레포지토리 제작 프롬프트

나는 "Spring Data & Transaction Deep Dive" 레포지토리를 만들려고 해.
Spring Core / Spring MVC Deep Dive를 완성한 경험을 바탕으로, **Repository 인터페이스만으로 어떻게 구현체가 생기는가, @Transactional 한 줄이 내부에서 무슨 일을 하는가** 를 동적 프록시·Hibernate 내부·HikariCP 튜닝까지 끝까지 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "Repository를 만드는 것과, Repository가 어떻게 작동하는지 아는 것은 다르다"

**핵심 차별화**:
1. "어떻게 쓰나"가 아닌 **"어떻게 동작하나"** (`RepositoryFactorySupport` 동적 프록시 생성 등 Spring 내부 추적)
2. Hibernate `show_sql` 출력으로 **N+1 5가지 해결 전략별 실제 쿼리 수 실측 비교**
3. Propagation 7가지가 `AbstractPlatformTransactionManager` 에서 어떻게 분기되는지 소스 레벨로 분해
4. CGLIB 프록시 한계 → Self-Invocation 함정 → 바이트코드 레벨 설명
5. HikariCP `ConcurrentBag` 같은 내부 자료구조 + Pool Size 공식의 이론적 배경

**타겟 독자**:
- Spring Data JPA를 매일 쓰지만 `JpaRepository` 인터페이스 선언만으로 어떻게 구현체가 생기는지 모르는 개발자
- `@Transactional` Propagation 7가지 차이를 면접에서 설명하지 못하는 개발자
- N+1 문제를 `fetch join`으로만 해결하는 개발자
- `private` 메서드에 `@Transactional`이 안 되는 진짜 이유가 궁금한 개발자
- HikariCP Pool Size를 "그냥 10"으로 두는 개발자
- `@DataJpaTest`의 `@Transactional` 자동 롤백이 함정인지 모르는 개발자

**선행 학습 (필수)**:
- **Spring Core Deep Dive** — IoC, DI, AOP 프록시 원리 이해 필수 (이 레포는 Core를 안다고 가정)

**기존 레포와의 관계**:
- **Spring Core Deep Dive**: AOP / Proxy 메커니즘이 트랜잭션 처리에 어떻게 적용되는가
- **JVM Deep Dive**: CGLIB 바이트코드 / Hibernate ByteBuddy 프록시 / Connection Pool 내부 자료구조
- **Object**: 도메인 객체 vs JPA Entity 설계 트레이드오프
- **Unit Testing**: `@DataJpaTest` 슬라이스 / Testcontainers / `@Transactional` in Test 함정

---

## 🎯 1단계: 전체 구조 (이미 확정 — 변경 금지)

이 레포는 **7개 챕터, 총 45개 문서**로 구성된다. 챕터·문서 제목·핵심 내용은 아래에 고정되어 있으며 추가/삭제/순서 변경 없이 이대로 작성한다.

### 🔹 Chapter 1: Spring Data JPA Internals (8개 문서)
> **핵심 질문:** `JpaRepository` 인터페이스만 선언했는데, 런타임에 어떻게 구현체가 생기는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. Repository 프록시 생성 과정](./spring-data-jpa-internals/01-repository-proxy-creation.md) | `RepositoryFactorySupport.getRepository()`가 JDK 동적 프록시를 생성하는 과정, `SimpleJpaRepository`로 위임되는 구조, `@EnableJpaRepositories` 트리거 |
| [02. Query Method 파싱 메커니즘](./spring-data-jpa-internals/02-query-method-parsing.md) | `findByNameAndAge()`가 JPQL로 변환되는 `PartTree` 파싱 과정, Subject / Predicate 분리, 지원 키워드 목록과 한계 |
| [03. @Query JPQL vs Native Query 처리](./spring-data-jpa-internals/03-query-annotation-jpql-native.md) | `@Query` 어노테이션이 `NamedQuery`와 다르게 처리되는 이유, Native Query 실행 경로, `nativeQuery=true`일 때 결과 매핑 방식 |
| [04. Projection — Interface vs DTO](./spring-data-jpa-internals/04-projection-interface-dto.md) | Closed / Open Interface Projection이 Hibernate 프록시로 구현되는 방식, DTO Projection이 인터페이스 방식보다 성능상 유리한 조건 |
| [05. QueryDSL 통합 원리](./spring-data-jpa-internals/05-querydsl-integration.md) | `QuerydslPredicateExecutor`가 Spring Data에 통합되는 방식, `JPAQueryFactory` 설정과 `Q타입` 생성 메커니즘, 타입 안전 동적 쿼리 |
| [06. Specifications를 통한 동적 쿼리](./spring-data-jpa-internals/06-specifications-dynamic-query.md) | `JpaSpecificationExecutor`와 `Specification<T>` 인터페이스, Criteria API를 래핑하는 구조, Specification 조합(`and`, `or`) 원리 |
| [07. Auditing 동작 원리](./spring-data-jpa-internals/07-auditing-internals.md) | `@CreatedDate` / `@LastModifiedDate`가 `AuditingEntityListener`와 `AuditorAware`를 통해 주입되는 과정, `@EnableJpaAuditing` 설정 체인 |
| [08. Custom Repository 구현 패턴](./spring-data-jpa-internals/08-custom-repository-pattern.md) | `Impl` 네이밍 컨벤션의 동작 원리, `JpaRepositoryImplementation` 위임 구조, `EntityManager` 직접 사용이 필요한 경우와 설계 트레이드오프 |

### 🔹 Chapter 2: Transaction Management (8개 문서)
> **핵심 질문:** `@Transactional(propagation=REQUIRES_NEW)`는 내부적으로 무슨 일을 하는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. PlatformTransactionManager 구조와 구현체](./transaction-management/01-platform-transaction-manager.md) | `PlatformTransactionManager` 인터페이스 설계, `JpaTransactionManager` vs `DataSourceTransactionManager` 구현 차이, `TransactionStatus` 객체 역할 |
| [02. @Transactional 프록시 생성 메커니즘](./transaction-management/02-transactional-proxy-mechanism.md) | `TransactionInterceptor`가 AOP Advice로 등록되는 과정, `TransactionAttributeSource`의 어노테이션 파싱, 트랜잭션 시작/커밋/롤백 흐름 |
| [03. Propagation 7가지 완전 분석](./transaction-management/03-propagation-seven-types.md) | `REQUIRED` / `REQUIRES_NEW` / `NESTED` / `SUPPORTS` / `NOT_SUPPORTED` / `MANDATORY` / `NEVER` 각각의 분기 조건과 내부 동작, 물리 트랜잭션 vs 논리 트랜잭션 구분 |
| [04. Isolation Level과 Database Lock](./transaction-management/04-isolation-level-lock.md) | READ_UNCOMMITTED ~ SERIALIZABLE 4단계의 JDBC 설정 경로, Phantom Read / Non-Repeatable Read 재현 코드, DB별 기본 Isolation Level과 Lock 전략 |
| [05. private 메서드에 @Transactional이 안 되는 이유](./transaction-management/05-why-private-transactional-fails.md) | CGLIB 오버라이딩 불가 원리, Self-Invocation 함정 (`this.method()` 호출 시 프록시 우회), `AopContext.currentProxy()` 우회 방법과 트레이드오프 |
| [06. readOnly=true의 실제 효과](./transaction-management/06-readonly-true-effects.md) | Hibernate `FlushMode.MANUAL` 설정, 스냅샷 비교 생략 비용 절감, JDBC `Connection.setReadOnly()` 힌트, MySQL/PostgreSQL 리플리카 라우팅 연계 |
| [07. Rollback 규칙 — checked vs unchecked](./transaction-management/07-rollback-rules.md) | Spring 기본 롤백 정책 (`RuntimeException` 기준), `rollbackFor` / `noRollbackFor` 설정 방법, checked exception으로 롤백하지 않아 데이터가 오염되는 실제 사례 |
| [08. TransactionSynchronization 활용](./transaction-management/08-transaction-synchronization.md) | `TransactionSynchronizationManager`의 ThreadLocal 기반 자원 관리, `afterCommit()` / `afterCompletion()` 훅 활용 패턴, `@TransactionalEventListener`와의 연결 |

### 🔹 Chapter 3: JPA & Hibernate Integration (7개 문서)
> **핵심 질문:** Hibernate는 어떻게 변경을 감지하고, 언제 쿼리를 날리는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. EntityManager vs Hibernate Session](./jpa-hibernate-integration/01-entitymanager-vs-session.md) | JPA 표준 `EntityManager`와 Hibernate `Session`의 관계, `EntityManager.unwrap(Session.class)` 필요한 상황, Spring의 `SharedEntityManagerCreator` 프록시 구조 |
| [02. Persistence Context — 1차 캐시 동작 원리](./jpa-hibernate-integration/02-persistence-context-first-cache.md) | `PersistenceContext` 내부 `EntityKey → EntityEntry` 맵 구조, 동일 트랜잭션 내 동일 ID 조회 시 쿼리가 발생하지 않는 이유, Detached / Managed / Removed 상태 전환 |
| [03. Dirty Checking 메커니즘](./jpa-hibernate-integration/03-dirty-checking-mechanism.md) | `ActionQueue`와 `EntityEntry.loadedState` 스냅샷 비교 방식, `flush()` 시점에 변경 감지가 일어나는 내부 코드, FlushMode 별 동작 차이 (`AUTO` vs `COMMIT`) |
| [04. N+1 문제 완전 해결](./jpa-hibernate-integration/04-n-plus-one-complete-solution.md) | N+1 발생 원리 (Lazy Loading 프록시 초기화 시점), `@EntityGraph` / `Fetch Join` / `@BatchSize` / `FetchMode.SUBSELECT` / DTO Projection 5가지 해결 전략별 실제 SQL 쿼리 수 비교 |
| [05. Lazy Loading 프록시 생성 과정](./jpa-hibernate-integration/05-lazy-loading-proxy.md) | Hibernate `ByteBuddyProxyFactory`가 엔티티 서브클래스를 생성하는 방식, `LazyInitializationException` 발생 조건, OSIV(Open Session In View) 패턴의 트레이드오프 |
| [06. Cascade 타입과 Orphan Removal](./jpa-hibernate-integration/06-cascade-orphan-removal.md) | `CascadeType` 6가지의 내부 동작 원리, `orphanRemoval=true`와 `CascadeType.REMOVE`의 차이, 무분별한 `ALL` 사용의 위험성과 연관관계 설계 기준 |
| [07. Batch Insert/Update 최적화](./jpa-hibernate-integration/07-batch-insert-update.md) | `hibernate.jdbc.batch_size` 설정만으로 Batch가 안 되는 이유 (`IDENTITY` 전략 한계), `SEQUENCE` 전략과 `allocationSize` 조합, `saveAll()` 호출 시 실제 발생하는 SQL 수 실측 |

### 🔹 Chapter 4: Query Performance Tuning (6개 문서)
> **핵심 질문:** 같은 결과를 내는 쿼리인데, 왜 어떤 방법은 SQL이 1개고 어떤 방법은 N+1개인가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. JPQL vs Criteria API vs QueryDSL 비교](./query-performance-tuning/01-jpql-criteria-querydsl.md) | 세 방식의 컴파일 타임 안전성 / 동적 쿼리 지원 / 성능 비교, 각각이 내부에서 `CriteriaQuery`와 `NativeQuery`로 변환되는 경로 |
| [02. Fetch Join vs @EntityGraph 선택 기준](./query-performance-tuning/02-fetch-join-vs-entity-graph.md) | Fetch Join과 `@EntityGraph`가 생성하는 SQL 비교, `MultipleBagFetchException` 발생 조건과 회피 전략, 컬렉션 2개 이상 조인 시 선택 기준 |
| [03. Pagination 최적화 전략](./query-performance-tuning/03-pagination-optimization.md) | `Pageable`을 사용한 `LIMIT/OFFSET` 방식의 성능 함정 (대용량 오프셋), Cursor 기반 Pagination, `CountQuery` 분리 최적화, `@QueryHints`로 페이지 카운트 캐싱 |
| [04. @BatchSize vs default_batch_fetch_size](./query-performance-tuning/04-batch-size-configuration.md) | `@BatchSize`(엔티티 레벨)와 `hibernate.default_batch_fetch_size`(전역 설정)의 차이, IN 절 파라미터 수가 성능에 미치는 영향, 적정 값 설정 기준 |
| [05. Query Plan Cache 동작](./query-performance-tuning/05-query-plan-cache.md) | Hibernate가 JPQL을 파싱해 `QueryPlan`으로 캐싱하는 구조, `hibernate.query.plan_cache_max_size` 튜닝, 파라미터 바인딩 방식(`?` vs `:name`)이 캐시 히트율에 미치는 영향 |
| [06. Second-Level Cache — Ehcache / Redis](./query-performance-tuning/06-second-level-cache.md) | `@Cache` 어노테이션이 Hibernate Region Factory를 통해 Ehcache/Redis에 저장되는 구조, `READ_ONLY` / `READ_WRITE` / `NONSTRICT_READ_WRITE` 전략 차이, 캐시 무효화 시점 |

### 🔹 Chapter 5: Spring JDBC (5개 문서)
> **핵심 질문:** `JdbcTemplate`은 어떻게 반복되는 JDBC 보일러플레이트를 제거하는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. JdbcTemplate 내부 구조](./spring-jdbc/01-jdbctemplate-internals.md) | Template Method 패턴으로 `Connection` 획득 / `PreparedStatement` 생성 / 예외 변환 / `Connection` 반환이 추상화되는 방식, `DataSourceUtils`를 통한 트랜잭션 컨텍스트 연계 |
| [02. RowMapper vs ResultSetExtractor](./spring-jdbc/02-rowmapper-vs-resultsetextractor.md) | `RowMapper`(행 단위 변환)와 `ResultSetExtractor`(전체 `ResultSet` 제어)의 사용 시점, `BeanPropertyRowMapper`의 리플렉션 비용과 커스텀 `RowMapper`와의 성능 비교 |
| [03. NamedParameterJdbcTemplate 활용](./spring-jdbc/03-named-parameter-jdbctemplate.md) | `:name` 바인딩이 내부적으로 `?` 치환되는 파싱 과정, `MapSqlParameterSource` vs `BeanPropertySqlParameterSource` 차이, SQL Injection 방지 원리 |
| [04. SimpleJdbcInsert로 단순 삽입](./spring-jdbc/04-simple-jdbc-insert.md) | `SimpleJdbcInsert`가 DB 메타데이터를 조회해 컬럼 목록을 자동 감지하는 방식, `usingGeneratedKeyColumns()`로 자동 생성 키를 받는 과정 |
| [05. Batch 처리 최적화](./spring-jdbc/05-batch-processing.md) | `batchUpdate()`가 `PreparedStatement.addBatch()` / `executeBatch()`를 감싸는 구조, Chunk 단위 처리 패턴, JPA `saveAll()`과의 실측 처리 속도 비교 |

### 🔹 Chapter 6: Connection Pool Management (6개 문서)
> **핵심 질문:** 적정 Pool Size는 어떻게 결정하며, Connection Leak은 어디서 발생하는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. HikariCP 설정과 최적화](./connection-pool-management/01-hikaricp-configuration.md) | `HikariPool` 내부 `ConcurrentBag` 자료구조, 주요 설정값(`maximumPoolSize`, `minimumIdle`, `connectionTimeout`) 의미와 상호작용, Spring Boot 자동 설정 경로 |
| [02. Connection Leak 탐지와 디버깅](./connection-pool-management/02-connection-leak-detection.md) | `leakDetectionThreshold` 설정 시 `ProxyLeakTask`가 스택 트레이스를 캡처하는 방식, 트랜잭션 밖에서 `EntityManager`를 직접 사용할 때 발생하는 Leak 패턴 |
| [03. Pool Size 튜닝 공식](./connection-pool-management/03-pool-size-tuning.md) | `Connections = cores × 2 + effective_spindle_count` 공식의 이론적 배경, I/O 바운드 vs CPU 바운드 서비스에서의 적정값 차이, 과도한 Pool Size가 오히려 성능을 낮추는 이유 |
| [04. Connection Health Check 전략](./connection-pool-management/04-connection-health-check.md) | `connectionTestQuery` vs `connectionInitSql` vs JDBC4 `isValid()` 방식 비교, `keepaliveTime`으로 유휴 Connection 유효성 유지, DB 재시작 후 Pool 복구 동작 |
| [05. Statement Caching 효과](./connection-pool-management/05-statement-caching.md) | HikariCP `cachePrepStmts` 설정이 DB 드라이버 레벨 캐싱과 연동되는 방식, MySQL `prepStmtCacheSize`와 `prepStmtCacheSqlLimit` 튜닝 포인트, 캐싱 전후 파싱 비용 실측 |
| [06. Connection Timeout vs Idle Timeout](./connection-pool-management/06-connection-timeout-vs-idle-timeout.md) | `connectionTimeout`(연결 대기) / `idleTimeout`(유휴 제거) / `maxLifetime`(최대 수명) 세 타임아웃의 역할 차이, DB 방화벽 강제 종료 시나리오에서 `maxLifetime` 설정의 중요성 |

### 🔹 Chapter 7: Testing Data Layer (5개 문서)
> **핵심 질문:** `@DataJpaTest`는 무엇을 로딩하고, 테스트의 `@Transactional`은 왜 함정인가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. @DataJpaTest 범위와 제약](./testing-data-layer/01-datajpatest-scope.md) | `@DataJpaTest`가 로딩하는 슬라이스 컨텍스트 범위, 기본 H2 인메모리 DB 설정 경로, `@Service` 빈이 로딩되지 않는 이유와 `@Import`로 추가하는 방법 |
| [02. @Transactional in Test의 함정](./testing-data-layer/02-transactional-test-pitfall.md) | 테스트 메서드의 `@Transactional` 자동 롤백이 실제 트랜잭션 동작을 은폐하는 방식, `REQUIRES_NEW` 동작이 테스트에서 다르게 보이는 이유, 통합 테스트에서 `@Rollback(false)` 사용 기준 |
| [03. Testcontainers vs H2 선택 기준](./testing-data-layer/03-testcontainers-vs-h2.md) | H2 호환 모드의 한계(JSON 타입, DB 고유 함수), Testcontainers `@DynamicPropertySource`로 실제 DB를 띄우는 방식, CI 환경에서의 컨테이너 재사용 전략 |
| [04. Repository 테스트 전략](./testing-data-layer/04-repository-test-strategy.md) | Query Method 단위 테스트 / 복잡한 JPQL 통합 테스트 / Custom Repository 격리 테스트 전략 구분, `TestEntityManager`를 활용한 fixture 데이터 준비 패턴 |
| [05. @Sql 스크립트 관리](./testing-data-layer/05-sql-script-management.md) | `@Sql`의 `executionPhase`(`BEFORE_TEST_METHOD` / `AFTER_TEST_METHOD`) 동작 순서, SQL 스크립트 격리 전략, `@SqlConfig`로 트랜잭션 처리 방식 제어 |

**총 문서 수: 8 + 8 + 7 + 6 + 5 + 6 + 5 = 45개 (확정)**

---

## 📐 문서 구조 (모든 문서 동일 적용)

```markdown
## 🎯 핵심 질문
## 🔍 왜 이 메커니즘이 존재하는가
## 😱 흔한 오해 또는 잘못된 사용 (Before)
## ✨ 올바른 이해와 사용 (After)
## 🔬 내부 동작 원리 (Spring / Hibernate / HikariCP 소스 추적)
## 💻 실험으로 확인하기 (실행 가능한 코드 + Hibernate `show_sql` 출력)
## 📊 SQL 쿼리 수 / 처리 시간 실측 (해당되는 경우)
## 🤔 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

---

## 🎨 스타일 가이드

1. **Spring / Hibernate / HikariCP 소스코드 직접 추적** — `RepositoryFactorySupport`, `TransactionInterceptor`, `AbstractPlatformTransactionManager`, `ActionQueue`, `HikariPool.ConcurrentBag` 등 실제 클래스 호출
2. **모든 N+1 / Batch / Pagination 비교는 Hibernate `show_sql` 출력 첨부 강제** — "쿼리가 N개 나간다" ❌ → "이 코드를 돌리면 다음 N+1개의 SQL이 출력된다" ✅
3. **Spring Boot 3.x + Java 17 + Hibernate 6.x + HikariCP 5.x** 기준 통일
4. **트랜잭션 동작은 Sequence Diagram(mermaid)로 시각화** — Propagation 7가지가 어떻게 분기되는지 그림으로
5. **CGLIB / ByteBuddy 프록시는 바이트코드 분석** — Self-Invocation이 왜 안 되는지 `javap`로 보여주기
6. **Connection Pool은 `ConcurrentBag` 같은 자료구조 시각화 강제** — 추상적 설명 금지

---

## 🛠️ 실험 환경

- Spring Boot 3.x + Spring Data JPA + Hibernate 6.x
- HikariCP 5.x (Spring Boot 기본)
- Java 17+
- MySQL / PostgreSQL (Testcontainers로 실행)
- H2 (단순 슬라이스 테스트)
- Hibernate `show_sql=true`, `format_sql=true`, `use_sql_comments=true`
- p6spy 또는 datasource-proxy (실제 발생 SQL 로깅)
- VisualVM / JFR (Connection Pool 모니터링)

---

## 🎯 2단계: 작업 순서

전체 구조는 위에 이미 확정되어 있으므로:

1. **디렉토리 생성**: bash 명령어로 7개 챕터 폴더 생성
   ```
   spring-data-jpa-internals/
   transaction-management/
   jpa-hibernate-integration/
   query-performance-tuning/
   spring-jdbc/
   connection-pool-management/
   testing-data-layer/
   ```
2. **README.md 작성**:
   - 빠른 시작 배지 7개 (각 챕터의 첫 문서)
   - "일반 자료 vs 이 레포" 비교 표 (현재 README 톤 그대로)
   - 7개 챕터 details 토글 + 표 형식 문서 목록
   - **Spring Core Deep Dive 선행 학습 권장 명시**
   - 학습 경로 (`@Transactional` 면접 2주 / JPA 원리 6주 / 종합 마스터 3개월)
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 한 챕터 완성 후 다음으로
   - 각 문서는 3000~4000 단어 분량 (소스 추적 분량 포함)
   - **모든 문서는 위 10섹션 구조 준수**
   - **모든 SQL 관련 문서는 `show_sql` 출력 첨부 강제**

---

## 📚 참고 자료

<Spring Core Deep Dive README를 여기에 붙여넣기>
<Spring MVC Deep Dive README를 여기에 붙여넣기>
<JVM Deep Dive README를 여기에 붙여넣기>

위 README들의 비주얼 톤 (배지, details, 표) 유지.

**Spring Core Deep Dive** 가 선행 학습이므로, AOP / Proxy 메커니즘은 깊이 설명하지 않고 "Core 레포에서 다룬 그대로" 가정.

---

## 💡 핵심 분석 대상

### Repository 동적 프록시 생성 (Chapter 1)

```java
public interface UserRepository extends JpaRepository<User, Long> {
    User findByEmail(String email);  // 구현체 없음. 어떻게 동작하는가?
}
```

```
Spring 시작 시:
1. @EnableJpaRepositories → JpaRepositoriesRegistrar
2. RepositoryFactorySupport.getRepository(UserRepository.class)
3. JDK Proxy.newProxyInstance(loader, [UserRepository.class], handler)
4. handler = RepositoryFactorySupport$DefaultMethodInvokingMethodInterceptor
5. 내부적으로 SimpleJpaRepository로 위임
6. findByEmail → PartTree 파싱 → JPQL "SELECT u FROM User u WHERE u.email = ?1"
```

→ 인터페이스 선언만으로 구현체가 생기는 전체 체인 추적

### Propagation 7가지 분기 (Chapter 2)

```java
// AbstractPlatformTransactionManager.handleExistingTransaction() 단순화
if (propagation == NEVER) throw new IllegalTransactionStateException();
if (propagation == NOT_SUPPORTED) return suspend(transaction);
if (propagation == REQUIRES_NEW) {
    SuspendedResources s = suspend(transaction);
    try { return startTransaction(...); } catch (...) { resume(s); }
}
if (propagation == NESTED) {
    // Savepoint 생성
    if (useSavepointForNestedTransaction()) status.createAndHoldSavepoint();
}
// REQUIRED, SUPPORTS, MANDATORY는 기존 트랜잭션 그대로
```

→ 7가지가 각각 어떤 분기를 타는지 소스 레벨로 그리고 mermaid sequence diagram

### N+1 5가지 해결 전략 SQL 비교 (Chapter 3)

| 전략 | 발생 SQL 수 (사용자 10명, 각 주문 5개) |
|:-----|:------:|
| 기본 (Lazy) | 1 + 10 = 11개 |
| Fetch Join | 1개 (Cartesian product) |
| `@EntityGraph` | 1개 |
| `@BatchSize(size=5)` | 1 + 2 = 3개 |
| `FetchMode.SUBSELECT` | 1 + 1 = 2개 |
| DTO Projection | 1개 (필요 컬럼만) |

→ 각 전략의 `show_sql` 출력을 첨부해서 실제 어떻게 다른지 보여주기

### `private` 메서드 @Transactional 안 되는 이유 (Chapter 2)

```java
// CGLIB가 생성한 프록시 (단순화)
public class UserService$$EnhancerBySpringCGLIB extends UserService {
    @Override
    public void publicMethod() {  // ✅ 오버라이드 가능
        // 트랜잭션 시작 → super.publicMethod() → 커밋
    }
    // privateMethod()는 오버라이드 불가능 → 프록시 우회됨
}
```

→ `javap`로 실제 생성된 바이트코드 보여주기 + Self-Invocation 함정 분해

---

## 🎯 진행 방식

**구조는 이미 확정되어 있으므로 1단계는 생략하고 바로 2단계(디렉토리 생성)부터 시작.**

진행 순서:
1. 디렉토리 7개 생성
2. README.md 작성 (Spring Core 선행 학습 명시)
3. Chapter 1부터 순서대로 8 → 8 → 7 → 6 → 5 → 6 → 5 = 45개 문서 작성
4. 각 문서마다 10섹션 구조 강제 준수
5. SQL 관련 문서는 `show_sql` 출력 강제 첨부
6. CGLIB / ByteBuddy 프록시는 바이트코드 분석 강제

**시작해줘! 디렉토리 생성 후 README, 그다음 Chapter 1의 1번 문서부터.**
