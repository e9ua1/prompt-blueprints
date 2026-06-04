# Spring Boot Internals 레포지토리 제작 프롬프트

나는 "Spring Boot Internals" 레포지토리를 만들려고 해.
Spring Core / Spring MVC / Spring Data & Transaction Deep Dive를 완성한 경험을 바탕으로, **`@SpringBootApplication` 하나로 어떻게 모든 게 자동 설정되는가** 를 Auto-configuration 메커니즘부터 Fat JAR 실행 원리, Embedded Server 커스터마이징, GraalVM Native Image까지 끝까지 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "Spring Boot를 사용하는 것과, Spring Boot가 어떻게 마법을 부리는지 아는 것은 다르다"

**핵심 차별화**:
1. "어떻게 쓰나"가 아닌 **"어떻게 동작하나"** (`SpringApplication.run()` 한 줄을 분해)
2. `@EnableAutoConfiguration` → `AutoConfigurationImportSelector` → `spring.factories` / `imports` 파일 로딩 전 과정을 소스로 추적
3. `JarLauncher`가 커스텀 ClassLoader로 중첩 JAR를 실행하는 메커니즘을 바이트 레벨에서 분해
4. `PropertySource` 17단계 우선순위 + Relaxed Binding 정규화 알고리즘 명확히
5. Boot 2.x `spring.factories` → Boot 3.x `@AutoConfiguration` + `imports` 파일 마이그레이션 이유와 차이
6. `--debug` 옵션의 Auto-configuration 리포트 읽는 법 + 실측

**타겟 독자**:
- Spring Boot를 매일 쓰지만 `@SpringBootApplication` 어노테이션 하나가 어떻게 모든 걸 자동 설정하는지 모호한 개발자
- `@ConditionalOnClass` / `@ConditionalOnMissingBean` 평가 시점이 헷갈리는 개발자
- `application.yml` 우선순위 17단계를 정리하고 싶은 개발자
- Custom Auto-configuration 라이브러리를 만들고 싶은 개발자
- Fat JAR 안에서 무슨 일이 벌어지는지 궁금한 개발자
- GraalVM Native Image로 Spring Boot 앱을 빌드하고 싶은 개발자
- Actuator 커스텀 Endpoint를 작성하고 싶은 개발자

**선행 학습 (필수)**:
- **Spring Core Deep Dive** — IoC, DI, Bean LifeCycle, ApplicationContext 이해 필수
- **Spring MVC Deep Dive** — DispatcherServlet, HandlerMapping 이해 (5장에서 활용)

**기존 레포와의 관계**:
- **Spring Core Deep Dive**: `ApplicationContext` 생성·refresh 흐름 → Spring Boot가 이를 어떻게 확장하는가
- **Spring MVC Deep Dive**: `DispatcherServlet` 등록을 `WebMvcAutoConfiguration`이 어떻게 자동화하는가
- **Spring Data & Transaction Deep Dive**: `DataSourceAutoConfiguration` / `HibernateJpaAutoConfiguration` 추적
- **JVM Deep Dive**: `JarLauncher`의 커스텀 ClassLoader가 만드는 클래스 로딩 격리 / GraalVM Native Image AOT
- **Java Design Patterns**: Spring Boot의 Auto-configuration이 사용하는 Builder + Factory 패턴

---

## 🎯 1단계: 전체 구조 (이미 확정 — 변경 금지)

이 레포는 **7개 챕터, 총 45개 문서**로 구성된다. 챕터·문서 제목·핵심 내용은 아래에 고정되어 있으며 추가/삭제/순서 변경 없이 이대로 작성한다.

### 🔹 Chapter 1: Spring Boot Startup Process (7개 문서)
> **핵심 질문:** `SpringApplication.run()` 한 줄이 실행될 때, 내부에서 정확히 어떤 순서로 무슨 일이 벌어지는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. SpringApplication.run() 전체 과정](./startup-process/01-spring-application-run.md) | `prepareEnvironment()` → `createApplicationContext()` → `refreshContext()` 전 과정, 각 단계에서 호출되는 핵심 메서드 추적 |
| [02. @SpringBootApplication 3개 어노테이션 분해](./startup-process/02-springbootapplication-decompose.md) | `@SpringBootConfiguration` + `@EnableAutoConfiguration` + `@ComponentScan` 각각의 역할과 조합 효과, 없앴을 때 차이 |
| [03. @EnableAutoConfiguration 동작 원리](./startup-process/03-enable-auto-configuration.md) | `AutoConfigurationImportSelector`가 후보 설정 클래스를 수집하는 과정, `ImportCandidates` 로딩 메커니즘 |
| [04. spring.factories vs @AutoConfiguration (Boot 3.x)](./startup-process/04-spring-factories-vs-autoconfiguration.md) | Boot 2.x `spring.factories` 방식과 Boot 3.x `@AutoConfiguration`+`imports` 파일 방식의 차이, 마이그레이션 이유 |
| [05. @Conditional 어노테이션 평가 순서](./startup-process/05-conditional-evaluation-order.md) | `ConditionEvaluator`가 `@Conditional`을 평가하는 시점, Phase(`PARSE_CONFIGURATION` vs `REGISTER_BEAN`) 차이, 평가 순서 충돌 |
| [06. ApplicationContext 생성 과정](./startup-process/06-application-context-creation.md) | `AnnotationConfigServletWebServerApplicationContext` 생성 조건, 웹/비웹 타입에 따른 컨텍스트 선택 로직, `refresh()` 호출 흐름 |
| [07. Banner 출력과 Startup Logging](./startup-process/07-banner-and-startup-logging.md) | `SpringApplicationBannerPrinter` 동작, `StartupInfoLogger`가 시작 시간을 측정하는 방식, 시작 로그에 담긴 정보 읽는 법 |

### 🔹 Chapter 2: Auto-configuration Deep Dive (8개 문서)
> **핵심 질문:** Spring Boot는 클래스패스에 무엇이 있는지 어떻게 감지하고, 어떤 기준으로 Bean을 자동 생성하는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. @ConditionalOnClass 동작 메커니즘](./auto-configuration/01-conditional-on-class.md) | ASM 기반 클래스 존재 확인 원리(클래스 로딩 없이), `FilteringSpringBootCondition`이 후보를 필터링하는 방식 |
| [02. @ConditionalOnBean vs @ConditionalOnMissingBean](./auto-configuration/02-conditional-on-bean-missing.md) | Bean 탐색 시점의 미묘한 차이, 사용자 정의 Bean이 Auto-configuration을 오버라이드하는 원리, 순서 의존성 함정 |
| [03. @AutoConfigureAfter/@AutoConfigureBefore 순서 제어](./auto-configuration/03-autoconfigure-order.md) | Auto-configuration 클래스 간 의존 순서 선언 방식, `AutoConfigurationSorter`가 위상 정렬로 순서를 결정하는 과정 |
| [04. DataSource Auto-configuration 분석](./auto-configuration/04-datasource-autoconfiguration.md) | `DataSourceAutoConfiguration` 소스 전체 추적, `spring.datasource.*` 프로퍼티가 HikariCP로 변환되는 경로 |
| [05. JPA Auto-configuration 과정](./auto-configuration/05-jpa-autoconfiguration.md) | `HibernateJpaAutoConfiguration` → `LocalContainerEntityManagerFactoryBean` 생성 체인, `spring.jpa.*` 옵션이 적용되는 지점 |
| [06. Web MVC Auto-configuration](./auto-configuration/06-web-mvc-autoconfiguration.md) | `WebMvcAutoConfiguration`이 `DispatcherServlet`, `HandlerMapping`, `ViewResolver`를 등록하는 과정, `WebMvcConfigurer` 확장 포인트 |
| [07. Custom Auto-configuration 작성 가이드](./auto-configuration/07-custom-autoconfiguration.md) | 라이브러리에 Auto-configuration 추가하는 전 과정, `@AutoConfiguration`, 조건 어노테이션 조합 실전 패턴 |
| [08. spring-boot-autoconfigure-processor 역할](./auto-configuration/08-autoconfigure-processor.md) | 컴파일 타임에 `@ConditionalOnClass` 조건을 미리 처리해 시작 성능을 높이는 APT 프로세서 동작 원리 |

### 🔹 Chapter 3: Property & Configuration Management (7개 문서)
> **핵심 질문:** `application.yml`의 값이 어떻게 Java 객체로 변환되고, 여러 소스의 설정이 충돌할 때 무엇이 이기는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. application.properties vs application.yml 처리](./property-configuration/01-application-properties-yml.md) | 두 포맷의 로딩 시점과 처리 클래스 차이, 멀티 문서 YAML(`---`)이 동작하는 방식 |
| [02. @ConfigurationProperties 바인딩 메커니즘](./property-configuration/02-configuration-properties-binding.md) | `ConfigurationPropertiesBindingPostProcessor` → `Binder` → `BindHandler` 체인, 타입 변환과 검증 흐름 |
| [03. Relaxed Binding](./property-configuration/03-relaxed-binding.md) | `kebab-case`, `camelCase`, `SCREAMING_SNAKE_CASE`를 동일하게 처리하는 `ConfigurationPropertyName` 정규화 알고리즘 |
| [04. Type-safe Configuration Properties](./property-configuration/04-type-safe-configuration.md) | `record` + `@ConfigurationProperties` + `@Validated` 조합으로 불변·검증된 설정 객체 만드는 패턴, `@DefaultValue` 처리 |
| [05. @Value vs @ConfigurationProperties 비교](./property-configuration/05-value-vs-configuration-properties.md) | 두 방식의 바인딩 시점 차이, 타입 안전성과 리팩터링 용이성 트레이드오프, 각각을 선택해야 하는 기준 |
| [06. Profile 활성화 전략](./property-configuration/06-profile-activation.md) | `spring.profiles.active` vs `spring.profiles.include` vs `@Profile`, Profile 그룹과 Profile별 설정 파일 로딩 순서 |
| [07. Environment PropertySource 우선순위 17단계](./property-configuration/07-propertysource-priority.md) | 커맨드라인 인수부터 JAR 내부 기본값까지 17단계 전체 정리, 환경변수 vs `application.yml` 충돌 시 무엇이 이기는가 |

### 🔹 Chapter 4: Actuator Internals (6개 문서)
> **핵심 질문:** `/actuator/health`는 어떻게 동작하고, 커스텀 Endpoint와 Health Indicator는 어떻게 만드는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. Actuator Endpoint 구조](./actuator-internals/01-actuator-endpoint-structure.md) | `@Endpoint` / `@WebEndpoint` / `@JmxEndpoint` 처리 경로, `EndpointDiscoverer`가 Endpoint를 HTTP 경로로 변환하는 과정 |
| [02. Custom Health Indicator 작성](./actuator-internals/02-custom-health-indicator.md) | `HealthIndicator` 구현 방법, `CompositeHealthContributor`로 여러 하위 시스템을 묶는 패턴, 상태 집계 알고리즘 |
| [03. Metrics 수집 — Micrometer 통합](./actuator-internals/03-micrometer-metrics.md) | `MeterRegistry` 계층 구조, `@Timed` / `Counter` / `Gauge` 동작 원리, Prometheus 연동 시 데이터 흐름 |
| [04. Custom Actuator Endpoint 작성](./actuator-internals/04-custom-endpoint.md) | `@ReadOperation` / `@WriteOperation` / `@DeleteOperation` 처리 방식, 파라미터 바인딩과 반환 타입 변환 |
| [05. JMX vs HTTP Exposure](./actuator-internals/05-jmx-vs-http-exposure.md) | 두 노출 방식의 내부 처리 차이, `management.endpoints.web.exposure.include` 설정이 적용되는 필터링 지점 |
| [06. Actuator Security 설정](./actuator-internals/06-actuator-security.md) | `EndpointRequest.toAnyEndpoint()` 매처 동작 원리, Spring Security와의 통합 방식, 운영 환경 노출 최소화 전략 |

### 🔹 Chapter 5: Embedded Server Configuration (6개 문서)
> **핵심 질문:** Spring Boot는 어떻게 외부 서버 없이 내장 서버로 요청을 받는가? 서버를 어떻게 교체하고 커스터마이징하는가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. Tomcat vs Jetty vs Undertow 비교](./embedded-server/01-tomcat-jetty-undertow.md) | 세 서버의 아키텍처 차이(스레드 모델, 커넥터 구조), 벤치마크 특성, 교체 시 의존성 변경 방법 |
| [02. ServletWebServerFactory 동작 원리](./embedded-server/02-servlet-web-server-factory.md) | `TomcatServletWebServerFactory`가 내장 Tomcat을 초기화하는 과정, `WebServer` 인터페이스와 생명주기 관리 |
| [03. Embedded Server 커스터마이징](./embedded-server/03-embedded-server-customization.md) | `WebServerFactoryCustomizer<T>` 구현 방식, 커넥터 설정, 스레드 풀 튜닝, 요청 큐 크기 조정 |
| [04. SSL/TLS 설정](./embedded-server/04-ssl-tls-configuration.md) | Keystore/Truststore 설정 방법, `server.ssl.*` 프로퍼티가 내장 서버에 적용되는 과정, mTLS 설정 |
| [05. HTTP/2 활성화](./embedded-server/05-http2-activation.md) | `server.http2.enabled=true`가 적용되는 내부 과정, ALPN 협상 메커니즘, Tomcat/Undertow/Jetty별 H2 지원 차이 |
| [06. Context Path & Port 설정](./embedded-server/06-context-path-port.md) | `server.port=0` 랜덤 포트 할당 원리, 다중 포트 리스닝, Context Path 설정이 `DispatcherServlet` 매핑에 미치는 영향 |

### 🔹 Chapter 6: Spring Boot DevTools (5개 문서)
> **핵심 질문:** DevTools는 어떻게 코드 변경을 감지하고 서버를 재시작하는가? Restart와 Reload는 무엇이 다른가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. LiveReload 동작 원리](./devtools/01-livereload-mechanism.md) | 내장 LiveReload 서버가 브라우저와 WebSocket으로 통신하는 방식, 파일 변경 감지 → 브라우저 리프레시 전 과정 |
| [02. Restart vs Reload 차이](./devtools/02-restart-vs-reload.md) | 두 개의 ClassLoader(Base / Restart) 분리 전략, Restart가 Full Restart보다 빠른 이유, JRebel과의 차이 |
| [03. Property Defaults — 캐싱 비활성화](./devtools/03-property-defaults.md) | DevTools가 자동으로 적용하는 개발용 기본값 목록(`spring.thymeleaf.cache=false` 등), 운영 환경에서 자동 비활성화 조건 |
| [04. Remote Debug & Remote Update](./devtools/04-remote-debug-update.md) | `spring.devtools.remote.secret` 설정으로 원격 서버에 클래스 변경을 푸시하는 메커니즘, HTTP 터널링 방식 |
| [05. Build Plugin 통합](./devtools/05-build-plugin-integration.md) | Maven / Gradle Spring Boot Plugin이 Fat JAR를 빌드하는 방식, `spring-boot:run`과 일반 `run`의 ClassLoader 차이 |

### 🔹 Chapter 7: Packaging & Deployment (6개 문서)
> **핵심 질문:** `java -jar app.jar`는 어떻게 동작하는가? 컨테이너 환경과 Native Image에 맞는 패키징은 무엇이 다른가?

| 문서 | 다루는 내용 |
|------|------------|
| [01. Fat JAR 구조와 JarLauncher](./packaging-deployment/01-fat-jar-structure.md) | `BOOT-INF/classes`, `BOOT-INF/lib` 디렉토리 구조, `JarLauncher`가 중첩 JAR를 로딩하는 커스텀 ClassLoader 구현 |
| [02. JarLauncher vs WarLauncher](./packaging-deployment/02-jar-vs-war-launcher.md) | 두 Launcher의 ClassLoader 전략 차이, WAR 배포 시 `ServletInitializer` 역할, 임베디드 vs 외부 컨테이너 배포 선택 기준 |
| [03. Layered JAR for Docker 최적화](./packaging-deployment/03-layered-jar-docker.md) | `jarmode=layertools`로 레이어를 분리하는 원리, 의존성 레이어 캐싱으로 Docker 빌드 시간을 줄이는 전략 |
| [04. Native Image with GraalVM](./packaging-deployment/04-native-image-graalvm.md) | AOT(Ahead-Of-Time) 컴파일 과정, 리플렉션·프록시 힌트 등록 방법, Spring Boot AOT 엔진이 힌트를 자동 생성하는 원리 |
| [05. Cloud Native Buildpacks](./packaging-deployment/05-cloud-native-buildpacks.md) | `spring-boot:build-image` 명령이 Buildpack으로 컨테이너 이미지를 만드는 과정, Dockerfile 없이 OCI 이미지 생성 |
| [06. Production-ready Checklist](./packaging-deployment/06-production-checklist.md) | Actuator 보안 설정, 로깅 레벨 전략, JVM 메모리 튜닝, Graceful Shutdown 설정, Health Check 엔드포인트 설계 |

**총 문서 수: 7 + 8 + 7 + 6 + 6 + 5 + 6 = 45개 (확정)**

---

## 📐 문서 구조 (모든 문서 동일 적용)

```markdown
## 🎯 핵심 질문
## 🔍 왜 이 메커니즘이 존재하는가
## 😱 흔한 오해 또는 잘못된 사용 (Before)
## ✨ 올바른 이해와 사용 (After)
## 🔬 내부 동작 원리 (Spring Boot 소스 직접 추적)
## 💻 실험으로 확인하기 (실행 가능한 코드 + `--debug` 리포트 분석)
## 📊 측정/검증 (해당되는 경우 — 시작 시간, 메모리, Native Image 크기 등)
## 🤔 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

---

## 🎨 스타일 가이드

1. **Spring Boot 소스코드 직접 추적 강제** — `SpringApplication`, `AutoConfigurationImportSelector`, `ConditionEvaluator`, `Binder`, `JarLauncher` 등 실제 클래스 호출 경로 첨부
2. **`--debug` 옵션 Auto-configuration 리포트 활용** — "이 Bean이 왜 등록되었는가 / 등록 안 되었는가"를 Positive matches / Negative matches 출력으로 증명
3. **Spring Boot 3.x + Java 17+** 기준 통일 (Boot 2.x와 차이 나는 부분만 별도 명시)
4. **`spring-boot-starter-actuator` 의 `/actuator/configprops`, `/actuator/beans`, `/actuator/conditions` 엔드포인트 적극 활용** — 실제로 어떤 설정과 Bean이 등록되었는지 검증
5. **Fat JAR 분석은 `unzip -l` 출력 + `JarLauncher` 바이트코드 분석** — 추상적 설명 금지
6. **Native Image는 시작 시간·메모리·이미지 크기 실측** — JVM 모드와 비교 표
7. **Auto-configuration 흐름은 mermaid sequence diagram** — `SpringApplication.run()` → ApplicationContext refresh → AutoConfigurationImportSelector → @Conditional 평가 등

---

## 🛠️ 실험 환경

- Spring Boot 3.x
- Java 17+ (GraalVM 21 for Native Image)
- Maven Spring Boot Plugin / Gradle Spring Boot Plugin
- `--debug` 옵션 (Auto-configuration 리포트)
- Spring Boot Actuator (`/actuator/conditions`, `/actuator/configprops`, `/actuator/beans`)
- Tomcat / Jetty / Undertow (서버 비교)
- GraalVM Native Image
- Docker / Buildpacks (컨테이너 패키징)

---

## 🎯 2단계: 작업 순서

전체 구조는 위에 이미 확정되어 있으므로:

1. **디렉토리 생성**: bash 명령어로 7개 챕터 폴더 생성
   ```
   startup-process/
   auto-configuration/
   property-configuration/
   actuator-internals/
   embedded-server/
   devtools/
   packaging-deployment/
   ```
2. **README.md 작성**:
   - 빠른 시작 배지 7개 (각 챕터의 첫 문서)
   - "일반 자료 vs 이 레포" 비교 표 (`@SpringBootApplication`을 붙이면 자동 설정됩니다 → `@EnableAutoConfiguration` → `AutoConfigurationImportSelector` → `spring.factories` 로딩 전 과정)
   - 7개 챕터 details 토글 + 표 형식 문서 목록
   - **Spring Core / Spring MVC Deep Dive 선행 학습 권장 명시**
   - 학습 경로 (Auto-configuration 입문 1주 / 설정·운영 2주 / 종합 마스터 6주 / Native Image 추적 별도)
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 한 챕터 완성 후 다음으로
   - 각 문서는 3000~4000 단어 분량 (소스 추적·실측 분량 포함)
   - **모든 문서는 위 10섹션 구조 준수**
   - **모든 Auto-configuration 관련 문서는 `--debug` 리포트 또는 `/actuator/conditions` 출력 첨부 강제**

---

## 📚 참고 자료

<Spring Core Deep Dive README를 여기에 붙여넣기>
<Spring MVC Deep Dive README를 여기에 붙여넣기>
<Spring Data & Transaction Deep Dive README를 여기에 붙여넣기>
<JVM Deep Dive README를 여기에 붙여넣기>

위 README들의 비주얼 톤 (배지, details 토글, 학습 경로) 유지.

**선행 학습**:
- **Spring Core Deep Dive** — `BeanDefinition`, `ApplicationContext` refresh 흐름은 거기서 다뤘다고 가정
- **Spring MVC Deep Dive** — `DispatcherServlet`, `HandlerMapping`은 거기서 다뤘다고 가정 (5장 Embedded Server에서 활용)

**공식 문서**:
- Spring Boot Reference — docs.spring.io/spring-boot/docs/current/reference/htmlsingle
- GraalVM Native Image — graalvm.org/native-image
- Cloud Native Buildpacks — buildpacks.io

---

## 💡 핵심 분석 대상

### `@SpringBootApplication` 분해 (Chapter 1)

```java
@Target(TYPE)
@SpringBootConfiguration  // = @Configuration
@EnableAutoConfiguration   // ★ 마법의 시작
@ComponentScan(...)        // 현재 패키지 하위 스캔
public @interface SpringBootApplication { ... }
```

```
@EnableAutoConfiguration
   ↓
@Import(AutoConfigurationImportSelector.class)
   ↓
selectImports() → ImportCandidates.load(AutoConfiguration.class, classLoader)
   ↓
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports 파일 로딩
   ↓
@AutoConfiguration이 붙은 후보 클래스 목록 반환
   ↓
각 클래스의 @Conditional 평가 → 통과한 것만 Bean 등록
```

→ 어노테이션 하나가 어떻게 수십 개의 자동 설정으로 이어지는지 전체 체인 추적

### Auto-configuration 리포트 읽기 (Chapter 2)

```
$ ./gradlew bootRun --args='--debug'

============================
CONDITIONS EVALUATION REPORT
============================

Positive matches:  // ✅ 조건 통과 → Bean 등록됨
-----------------
   DataSourceAutoConfiguration matched:
      - @ConditionalOnClass found required class 'javax.sql.DataSource'
      - @ConditionalOnMissingBean (types: javax.sql.DataSource)

Negative matches:  // ❌ 조건 미통과 → Bean 미등록
-----------------
   GsonAutoConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class 'com.google.gson.Gson'

Exclusions:
-----------
   None
```

→ 이 리포트를 모든 Auto-configuration 문서에 활용해서 "이 Bean이 왜 등록되었는가/안 되었는가"를 증명

### PropertySource 17단계 우선순위 (Chapter 3)

```
1. Devtools 글로벌 설정 (~/.spring-boot-devtools.properties)
2. @TestPropertySource (테스트만)
3. properties / args attribute @SpringBootTest
4. 커맨드라인 인수
5. SPRING_APPLICATION_JSON
6. ServletConfig init parameters
7. ServletContext init parameters
8. JNDI 속성
9. Java System Properties
10. OS 환경 변수
11. RandomValuePropertySource (random.*)
12. Profile-specific 외부 application-{profile}.yml
13. Profile-specific 내부 application-{profile}.yml
14. 외부 application.yml
15. 내부 application.yml
16. @PropertySource (@Configuration 클래스)
17. SpringApplication.setDefaultProperties
```

→ 같은 키가 여러 곳에 있을 때 위쪽이 이긴다 — 시각적으로 명확히 + 실측 코드 첨부

### JarLauncher 동작 (Chapter 7)

```
$ unzip -l app.jar
Archive:  app.jar
   META-INF/
   META-INF/MANIFEST.MF        ← Main-Class: org.springframework.boot.loader.JarLauncher
   org/springframework/boot/loader/  ← Spring Boot Loader
   BOOT-INF/
   BOOT-INF/classes/           ← 우리 코드
   BOOT-INF/lib/spring-core-6.x.jar  ← 의존성 JAR (중첩 JAR)
   BOOT-INF/lib/spring-context-6.x.jar
   ...
```

```java
// JarLauncher.main() 단순화
public static void main(String[] args) throws Exception {
    new JarLauncher().launch(args);
}

protected void launch(String[] args) {
    // 1. 중첩 JAR를 ClassLoader가 읽을 수 있게 LaunchedURLClassLoader 생성
    ClassLoader cl = createClassLoader(getClassPathArchives());
    // 2. Main-Class 속성 (실제 우리 메인 클래스) 호출
    String mainClass = getMainClass();  // BOOT-INF/classes/.../Application
    Method mainMethod = cl.loadClass(mainClass).getMethod("main", String[].class);
    mainMethod.invoke(null, (Object) args);
}
```

→ JVM은 중첩 JAR를 모르는데 어떻게 읽는가 — 커스텀 ClassLoader가 답

---

## 🎯 진행 방식

**구조는 이미 확정되어 있으므로 1단계는 생략하고 바로 2단계(디렉토리 생성)부터 시작.**

진행 순서:
1. 디렉토리 7개 생성
2. README.md 작성 (Spring Core / MVC 선행 학습 명시)
3. Chapter 1부터 순서대로 7 → 8 → 7 → 6 → 6 → 5 → 6 = 45개 문서 작성
4. 각 문서마다 10섹션 구조 강제 준수
5. Auto-configuration 관련 문서는 `--debug` 리포트 또는 `/actuator/conditions` 출력 강제 첨부
6. JarLauncher / Native Image는 실측 (시작 시간 / 메모리 / 이미지 크기)

**시작해줘! 디렉토리 생성 후 README, 그다음 Chapter 1의 1번 문서부터.**
