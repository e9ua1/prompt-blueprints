# Unit Testing: Principles to Practice 레포지토리 제작 프롬프트

나는 "Unit Testing: Principles to Practice" 레포지토리를 만들려고 해.
JVM Deep Dive를 완성한 경험을 바탕으로, **테스트가 거짓말을 하지 않게 만드는 원리**를 안티패턴 Before와 올바른 패턴 After 비교로 끝까지 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "테스트를 작성하는 것과, 올바른 테스트를 작성하는 것은 다르다"

**핵심 차별화**:
1. "어떻게 작성하나"가 아닌 "왜 이렇게 작성해야 하나" (원리 중심)
2. 모든 패턴은 **Before(안티패턴) + After(개선 코드) + 무엇이 달라졌는가** 3단 비교
3. Vladimir Khorikov의 "Unit Testing: Principles, Practices, and Patterns" 핵심을 한국어 + 실무 문맥으로 재구성
4. 테스트 코드 자체를 "코드"로 다룬다 — 가독성, 결합도, 변경 비용까지 측정
5. 테스트 어려움이 설계 문제의 신호임을 코드로 증명

**타겟 독자**:
- 테스트는 쓰지만 신뢰가 가지 않는 개발자 — 매번 리팩터링 후 테스트가 깨지는 이유를 모르는 사람
- Mock과 Stub을 구분하지 못해서 잘못된 검증을 하는 개발자
- 커버리지 100%인데 버그가 새는 이유가 궁금한 개발자
- "Mockito Repository" vs "In-memory Fake Repository" 선택을 못 하는 개발자
- TDD를 시도했지만 흐름을 잡지 못한 개발자
- 레거시 코드에 테스트를 추가해야 하는 개발자

**기존 레포와의 관계**:
- **JVM Deep Dive**: 멀티스레드 테스트 / `Thread.sleep` 안티패턴 / 동시성 검증 연결
- **Effective Java**: 테스트 가능한 설계 원칙 (불변성, 의존성 주입) 연결
- **Object**: 책임 주도 설계 → 테스트 가능한 설계로 이어지는 흐름

---

## 🎯 1단계: 전체 구조 (이미 확정됨 — 변경 금지)

이 레포는 **7개 섹션, 총 44개 문서**로 구성된다. 챕터·문서 제목·핵심 내용은 아래에 고정되어 있으며 추가/삭제/순서 변경 없이 이대로 작성한다.

### 🔹 Section 1: Testing Fundamentals (6개 문서)
> **핵심 질문:** "좋은 테스트"와 "있는 테스트"는 무엇이 다른가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. What Is a Unit?](./testing-fundamentals/01-what-is-a-unit.md) | "단위"의 정의가 팀마다 다른 이유, 고전파(Detroit) vs 런던파(London) 관점 차이 |
| [02. The Test Pyramid](./testing-fundamentals/02-the-test-pyramid.md) | Unit / Integration / E2E 비율의 근거, 피라미드가 뒤집히면 생기는 일 |
| [03. AAA Pattern](./testing-fundamentals/03-aaa-pattern.md) | Arrange-Act-Assert 구조가 깨지는 신호, Given-When-Then과의 차이 |
| [04. Test Naming Conventions](./testing-fundamentals/04-test-naming-conventions.md) | `testSave()` vs `save_whenDuplicate_throwsException()` — 이름이 문서가 되는 방법 |
| [05. Coverage Myths](./testing-fundamentals/05-coverage-myths.md) | 커버리지 100%인데 버그가 생기는 이유, 커버리지가 측정하지 못하는 것 |
| [06. FIRST Principles](./testing-fundamentals/06-first-principles.md) | Fast / Isolated / Repeatable / Self-validating / Timely — 각 원칙이 깨졌을 때의 증상 |

### 🔹 Section 2: Anatomy of Good Tests (6개 문서)
> **핵심 질문:** 테스트 코드도 코드다. 어떻게 읽기 쉽고 유지보수하기 쉽게 만드는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. Single Assert Principle](./anatomy-of-good-tests/01-single-assert-principle.md) | 단일 단언 원칙의 의미, 여러 단언이 필요할 때 어떻게 분리하는가 |
| [02. Test Data Builders](./anatomy-of-good-tests/02-test-data-builders.md) | 복잡한 픽스처를 Builder 패턴으로 정리하는 방법, 가독성 변화 비교 |
| [03. Boundary Testing](./anatomy-of-good-tests/03-boundary-testing.md) | 경계값 분석, Off-by-One 에러를 테스트로 잡는 방법 |
| [04. Parameterized Tests](./anatomy-of-good-tests/04-parameterized-tests.md) | `@ParameterizedTest`, `@CsvSource`, `@MethodSource` — 중복 없이 케이스 확장 |
| [05. Fixtures & SetUp](./anatomy-of-good-tests/05-fixtures-and-setup.md) | `@BeforeEach`의 올바른 범위, 공유 픽스처가 만드는 숨겨진 결합 |
| [06. Meaningful Assertions](./anatomy-of-good-tests/06-meaningful-assertions.md) | 실패 메시지를 읽으면 원인을 알 수 있는 단언 작성법, AssertJ 활용 |

### 🔹 Section 3: Mocking Strategies (7개 문서)
> **핵심 질문:** Stub, Mock, Fake — 이 세 가지를 구분하지 못하면 테스트가 거짓말을 한다

| 문서 | 다루는 내용 |
|------|------------|
| [01. Test Doubles Taxonomy](./mocking-strategies/01-test-doubles-taxonomy.md) | Dummy / Stub / Spy / Mock / Fake 5가지 분류, 언제 무엇을 선택하는가 |
| [02. Stub vs Mock](./mocking-strategies/02-stub-vs-mock.md) | 상태 검증 vs 행동 검증의 트레이드오프, 잘못된 Mock 사용이 만드는 취약한 테스트 |
| [03. Mockito Best Practices](./mocking-strategies/03-mockito-best-practices.md) | `@Mock` / `@InjectMocks` / `@Captor` 올바른 사용, `when().thenReturn()` vs `doReturn()` |
| [04. Partial Mocking with Spy](./mocking-strategies/04-partial-mocking-with-spy.md) | `@Spy`가 필요한 상황과 함정, Spy가 필요하다면 설계를 의심하라 |
| [05. Argument Matchers](./mocking-strategies/05-argument-matchers.md) | `any()` 남용이 만드는 거짓 통과, `ArgumentCaptor`로 정확하게 검증하기 |
| [06. Verify Wisely](./mocking-strategies/06-verify-wisely.md) | `verify()`를 언제 써야 하고 언제 쓰면 안 되는가, 과도한 검증의 문제 |
| [07. Fakes over Mocks](./mocking-strategies/07-fakes-over-mocks.md) | In-memory Repository가 Mockito Repository보다 나은 경우, Fake의 장단점 |

### 🔹 Section 4: Testable Design (6개 문서)
> **핵심 질문:** 테스트가 어렵다는 느낌은 설계가 나쁘다는 신호인가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. DI for Testability](./testable-design/01-dependency-injection-for-testability.md) | 의존성 주입이 테스트에 미치는 영향, 생성자 주입 vs 필드 주입의 테스트 비용 차이 |
| [02. Avoiding Static Methods](./testable-design/02-avoiding-static-methods.md) | 정적 메서드가 테스트를 망치는 이유, `PowerMock` 없이 해결하는 방법 |
| [03. Interface Segregation](./testable-design/03-interface-segregation.md) | 작은 인터페이스가 Stub을 쉽게 만드는 이유, ISP와 테스트 용이성의 관계 |
| [04. Humble Object Pattern](./testable-design/04-humble-object-pattern.md) | UI / 외부 시스템과 로직을 분리해 테스트 가능한 영역을 최대화하는 방법 |
| [05. Pure Functions First](./testable-design/05-pure-functions-first.md) | 부수효과 없는 로직의 테스트 용이성, 도메인 모델을 순수하게 유지하는 전략 |
| [06. Ports and Adapters](./testable-design/06-ports-and-adapters.md) | Hexagonal Architecture에서 테스트 경계를 어떻게 그리는가 |

### 🔹 Section 5: Integration Testing (6개 문서)
> **핵심 질문:** 단위 테스트가 모두 통과했는데 왜 시스템이 망가지는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. Integration Test Scope](./integration-testing/01-integration-test-scope.md) | 무엇을 통합 테스트로 검증해야 하는가, 단위 테스트와의 역할 분리 |
| [02. Database Testing with Testcontainers](./integration-testing/02-database-testing-testcontainers.md) | H2 대신 실제 DB로 테스트하기, Testcontainers 설정과 비용 |
| [03. REST API Testing](./integration-testing/03-rest-api-testing.md) | `MockMvc` vs `WebTestClient` vs `RestAssured` 선택 기준과 비교 |
| [04. Spring Context Slicing](./integration-testing/04-spring-context-slicing.md) | `@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest`의 범위와 올바른 선택 |
| [05. Test Transaction Management](./integration-testing/05-test-transaction-management.md) | `@Transactional` 테스트의 롤백 함정, 실제 커밋을 검증해야 할 때 |
| [06. Contract Testing](./integration-testing/06-contract-testing.md) | Consumer-Driven Contracts, Pact로 서비스 간 계약을 테스트하는 방법 |

### 🔹 Section 6: Test Anti-patterns (7개 문서)
> **핵심 질문:** 테스트가 있는데도 코드 변경이 두렵다면 — 무엇이 잘못된 것인가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. Test Logic in Production](./test-anti-patterns/01-test-logic-in-production.md) | `if (isTest)` 분기, 테스트 전용 생성자 — 테스트가 프로덕션을 오염시키는 패턴 |
| [02. Sleepy Tests](./test-anti-patterns/02-sleepy-tests.md) | `Thread.sleep()`으로 비동기를 검증하는 문제, `Awaitility`로 안전하게 대체하기 |
| [03. Flickering Tests](./test-anti-patterns/03-flickering-tests.md) | 불규칙하게 실패하는 테스트의 4가지 원인과 각각의 해결 전략 |
| [04. Overspecified Tests](./test-anti-patterns/04-overspecified-tests.md) | 구현 세부사항을 검증하는 테스트, 리팩터링할 때마다 테스트가 깨지는 이유 |
| [05. Test Code Duplication](./test-anti-patterns/05-test-code-duplication.md) | 반복되는 `setUp`, 반복되는 단언 — DRY가 테스트에서 의미하는 것 |
| [06. Hidden Test Dependencies](./test-anti-patterns/06-hidden-test-dependencies.md) | 테스트 실행 순서에 의존하는 테스트, 공유 상태가 만드는 시한폭탄 |
| [07. Assertion Roulette](./test-anti-patterns/07-assertion-roulette.md) | 실패 시 어떤 단언이 틀렸는지 알 수 없는 테스트, SoftAssertions 활용 |

### 🔹 Section 7: Advanced Topics (6개 문서)
> **핵심 질문:** 테스트를 "잘 쓴다"는 것을 넘어, 테스트가 설계를 이끌 수 있는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. Mutation Testing](./advanced-topics/01-mutation-testing.md) | PIT(PITest)로 테스트 품질 측정하기, 커버리지가 잡지 못하는 결함 찾기 |
| [02. Property-Based Testing](./advanced-topics/02-property-based-testing.md) | `jqwik`으로 경계를 자동 탐색, 예제 기반 테스트와 속성 기반 테스트의 차이 |
| [03. Architecture Testing](./advanced-topics/03-architecture-testing.md) | ArchUnit으로 레이어 의존성 규칙을 테스트로 강제하는 방법 |
| [04. TDD Workflow](./advanced-topics/04-tdd-workflow.md) | Red → Green → Refactor 사이클의 실전 리듬, 언제 테스트를 먼저 쓰고 언제 나중에 쓰는가 |
| [05. Test-Driven Design](./advanced-topics/05-test-driven-design.md) | 테스트를 먼저 쓰면 설계가 어떻게 달라지는가, "테스트 가능성 = 설계 품질"의 의미 |
| [06. Testing Legacy Code](./advanced-topics/06-testing-legacy-code.md) | 테스트 없는 코드에 테스트를 추가하는 순서, Seam 개념과 Characterization Test |

**총 문서 수: 6 + 6 + 7 + 6 + 6 + 7 + 6 = 44개 (확정)**

---

## 📐 문서 구조 (모든 문서 동일 적용)

```markdown
## 🎯 핵심 질문
## 🔍 왜 이 패턴이 중요한가 (이 패턴이 없을 때 생기는 실제 문제)
## 😱 안티패턴 (Before — 흔히 쓰는 잘못된 코드)
## ✨ 올바른 패턴 (After — 개선된 코드와 무엇이 더 나은지)
## 💻 실전 적용 (복사해서 바로 쓸 수 있는 예제, JUnit5 + AssertJ + Mockito)
## 🤔 트레이드오프 (이 패턴의 한계와 적용하지 말아야 할 상황)
## 📌 핵심 정리 (한 화면 요약)
```

---

## 🎨 스타일 가이드

1. **Before → After 코드 비교를 모든 문서에 강제** — Before가 없는 패턴 설명은 빈 약속과 같다
2. **JUnit 5 + AssertJ + Mockito 4.x 조합** 으로 통일 (JUnit 4 / Hamcrest 코드는 작성 금지)
3. **Spring 통합 테스트 예제는 Spring Boot 3.x + Java 17 기준**
4. **테스트 실패 시나리오를 직접 만들어 보여주기** — "이 코드가 실패할 때 IDE에 어떤 메시지가 뜨는가" 까지
5. **냄새 진단 → 리팩터링 → 검증** 흐름으로 안티패턴 섹션 작성 (단순 "이렇게 하지 마세요" 금지)
6. **실측 가능한 부분은 측정** — 예: Sleepy Test가 1000번 실행 시 평균 시간, Awaitility로 바꿨을 때 평균 시간

---

## 🛠️ 실험 도구

- JUnit 5 (`@ParameterizedTest`, `@MethodSource`, `@TestInstance`)
- AssertJ (`assertThat`, `SoftAssertions`, custom assertion)
- Mockito 4.x (`@Mock`, `@Spy`, `ArgumentCaptor`, `verify`)
- Awaitility (비동기 검증)
- Testcontainers (실제 DB 통합 테스트)
- ArchUnit (아키텍처 규칙 검증)
- PIT(PITest) (Mutation Testing)
- jqwik (Property-Based Testing)
- Spring Boot Test (`@WebMvcTest`, `@DataJpaTest`, `@SpringBootTest`)

---

## 🎯 2단계: 작업 순서

전체 구조는 위에 이미 확정되어 있으므로:

1. **디렉토리 생성**: bash 명령어로 7개 섹션 폴더 생성
   ```
   testing-fundamentals/
   anatomy-of-good-tests/
   mocking-strategies/
   testable-design/
   integration-testing/
   test-anti-patterns/
   advanced-topics/
   ```
2. **README.md 작성**:
   - 빠른 시작 배지 (각 섹션의 첫 문서로 바로 가는 진입점)
   - 7개 섹션 details 토글 + 표 형식 문서 목록
   - 학습 경로 (입문 2~3주 / 신뢰 회복 4~6주 / 마스터 2~3개월)
   - "일반 자료 vs 이 레포" 비교 표
   - 7섹션 문서 구조 명시
3. **섹션별 문서 작성**:
   - Section 1부터 순서대로
   - 한 섹션 완성 후 다음으로
   - 각 문서는 2500~3500 단어 분량
   - **모든 문서는 위 7섹션 구조를 준수**

---

## 📚 참고 자료

<JVM Deep Dive README를 여기에 붙여넣기>

<Effective Java README를 여기에 붙여넣기>

<Object README를 여기에 붙여넣기>

위 README들의 비주얼 톤 (배지, details 토글, 표)을 유지하되, 깊이는 안티패턴 → 올바른 패턴 비교 중심으로.

---

## 💡 핵심 분석 대상

### 안티패턴 vs 올바른 패턴 비교 — 모든 문서에 강제

```java
// ❌ Before — Sleepy Test (Section 6)
@Test
void asyncTask_completes_successfully() {
    service.startAsyncTask();
    Thread.sleep(1000);  // 환경에 따라 실패하는 시한폭탄
    assertThat(service.isCompleted()).isTrue();
}

// ✅ After — Awaitility
@Test
void asyncTask_completes_successfully() {
    service.startAsyncTask();
    await().atMost(2, SECONDS)
           .until(() -> service.isCompleted());
}
```

### Stub vs Mock 진짜 차이 (Section 3)

```java
// Stub — 상태 검증 (Detroit School)
when(userRepository.findById(1L)).thenReturn(Optional.of(user));
User result = service.getUser(1L);
assertThat(result.getName()).isEqualTo("IQ");

// Mock — 행동 검증 (London School)
service.deleteUser(1L);
verify(userRepository).delete(any(User.class));
verify(eventPublisher).publish(any(UserDeletedEvent.class));
```

→ 같은 테스트 대역인데 검증 방식이 다른 이유, 잘못 섞이면 테스트가 거짓말을 하게 되는 경로 추적

### Fake가 Mock보다 나은 경우 (Section 3)

```java
// Mockito Repository — 50줄의 stubbing 지옥
when(repo.findById(1L)).thenReturn(Optional.of(u1));
when(repo.findById(2L)).thenReturn(Optional.of(u2));
when(repo.findAll()).thenReturn(List.of(u1, u2));
when(repo.save(any())).thenAnswer(...);

// In-memory Fake — 한 번 작성해서 재사용
class FakeUserRepository implements UserRepository {
    private final Map<Long, User> store = new HashMap<>();
    public Optional<User> findById(Long id) { return Optional.ofNullable(store.get(id)); }
    public User save(User u) { store.put(u.getId(), u); return u; }
}
```

→ 어떤 레포지토리/서비스에 Fake가 더 나은 선택인지, 트레이드오프 명시

---

## 🎯 진행 방식

**구조는 이미 확정되어 있으므로 1단계는 생략하고 바로 2단계(디렉토리 생성)부터 시작.**

진행 순서:
1. 디렉토리 7개 생성
2. README.md 작성 (학습 경로 포함)
3. Section 1부터 순서대로 6 → 6 → 7 → 6 → 6 → 7 → 6 = 44개 문서 작성
4. 각 문서마다 7섹션 구조 강제 준수
5. Before/After 코드 비교를 모든 안티패턴/패턴 문서에 포함

**시작해줘! Section 1의 README와 문서 1번부터 작성.**
