# Observability Deep Dive 레포지토리 제작 프롬프트

나는 "Observability Deep Dive" 레포지토리를 만들려고 해.
Grafana 대시보드를 보는 것과, Java Agent가 바이트코드를 조작해 어떻게 메트릭을 수집하는지, 분산 추적이 서비스 경계를 넘어 어떻게 컨텍스트를 전파하는지, Prometheus가 시계열 데이터를 어떻게 압축 저장하는지를 완전히 파헤치는 레포야.

시스템이 복잡해질수록 "지금 무슨 일이 일어나고 있는가"를 아는 것이 가장 중요한 엔지니어링 스킬이야. 이 레포는 그 가시성(Observability)의 세 기둥인 메트릭/로그/트레이스가 내부에서 어떻게 동작하는지 파헤쳐.

## 📋 프로젝트 목표

**컨셉**: "메트릭을 수집하는 것과, 분산 추적이 서비스 경계를 넘어 어떻게 컨텍스트를 전파하고 연결하는지 아는 것은 다르다"

**핵심 차별화**:
1. Java Agent 완전 분해 — 바이트코드 조작(ASM/ByteBuddy)으로 어떻게 내 코드에 몰래 타이머를 심는가
2. 분산 추적의 전파 원리 — TraceContext가 HTTP 헤더를 타고 마이크로서비스를 건너가는 방식
3. Prometheus TSDB — 수십억 개의 시계열 데이터를 어떻게 압축하고 쿼리하는가
4. 비동기 컨텍스트 전파 — Virtual Thread/CompletableFuture에서 TraceContext를 잃지 않는 방법

**타겟 독자**:
- Micrometer를 쓰지만 Counter/Timer가 Prometheus에 어떻게 스크레이프되는지 모르는 개발자
- @Observed나 @NewSpan을 쓰지만 Span이 내부에서 어떻게 생성되고 전파되는지 모르는 개발자
- APM 도구가 코드 변경 없이 어떻게 메서드 실행 시간을 측정하는지 이해 못하는 개발자
- 로그와 트레이스를 따로 보면서 Trace ID로 연결하는 방법 모르는 개발자
- Grafana 대시보드를 만들지만 PromQL이 내부에서 어떤 연산을 하는지 모르는 개발자
- Spring Boot Actuator, Micrometer, Micrometer Tracing을 쓰면서 내부를 모르는 개발자

**선행 학습**:
- jvm-deep-dive (Java Agent, 바이트코드, 클래스로딩 이해 필수)
- java-concurrency-deep-dive (비동기 컨텍스트 전파 챕터 깊이 배가)
- network-deep-dive (HTTP 헤더를 통한 TraceContext 전파 이해 시 시너지)
- kubernetes-deep-dive (K8s에서의 Prometheus Operator, 사이드카 패턴과 연결 시 시너지)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Observability 기초 — 세 기둥과 설계 원칙 (4개 문서)
- Observability vs Monitoring — "알려진 미지수(Known Unknowns)" vs "알려지지 않은 미지수(Unknown Unknowns)", 왜 단순한 임계값 모니터링이 복잡한 시스템에서 부족한가
- 메트릭/로그/트레이스 — 각각이 답하는 질문의 차이, 세 기둥을 연결했을 때 어떤 진단이 가능해지는가
- OpenTelemetry 표준화 — OTel이 등장하기 전 벤더 종속 문제, Signal/Tracer/Meter/Logger 추상화 계층
- 계측(Instrumentation) 방법론 — 자동 계측(Java Agent) vs 수동 계측(@Observed) vs 라이브러리 계측, 각각의 적합 상황

### Chapter 2: Java Agent와 바이트코드 조작 — 코드 없이 계측하기 (6개 문서)
- Java Agent 메커니즘 — -javaagent 옵션, premain()과 agentmain() 차이, Instrumentation API로 클래스 변환하는 방식
- 바이트코드 조작 도구 비교 — ASM(저수준), Javassist(소스 코드 수준), ByteBuddy(선언적 DSL) 각각의 추상화 수준과 사용 사례
- ByteBuddy로 메서드 타이머 구현 — @Advice 어노테이션으로 메서드 진입/종료 시점에 코드 삽입, ElementMatcher로 계측 대상 선정
- OpenTelemetry Java Agent 내부 — 어떤 라이브러리에 어떤 계측이 자동 적용되는가, InstrumentationModule 구조, JDBC/HTTP/Spring 자동 계측
- Micrometer의 계측 모델 — MeterRegistry, Counter/Timer/Gauge/DistributionSummary 내부 구현, @Timed AOP 적용 방식
- 계측 오버헤드 측정 — Java Agent 추가로 인한 실제 CPU/메모리 오버헤드, 샘플링(Sampling)으로 오버헤드를 줄이는 방법

### Chapter 3: 분산 추적 — TraceContext가 서비스를 건너가는 방법 (6개 문서)
- Trace와 Span 모델 — 하나의 Trace를 구성하는 Span 계층 구조, ParentSpanId로 인과 관계를 표현하는 방식
- W3C TraceContext 표준 — traceparent 헤더 포맷(version-traceId-spanId-flags), tracestate로 벤더별 정보 전달
- HTTP 경계에서의 전파 — Propagator가 HTTP 요청에서 TraceContext를 추출하고 주입하는 방식, RestTemplate/WebClient/Feign 자동 계측
- 비동기 컨텍스트 전파 — CompletableFuture/ThreadPool에서 TraceContext가 유실되는 원인, Context 전파를 위한 Executor 래핑
- Virtual Thread에서의 추적 — Virtual Thread 스케줄링 시 ThreadLocal 기반 Context 전파의 한계, ScopedValue로의 전환
- Span 속성과 이벤트 — Span에 DB 쿼리, HTTP URL, 에러 정보를 추가하는 방법, Span Status로 오류를 표현하는 방식

### Chapter 4: 메트릭 — Prometheus TSDB 완전 분해 (6개 문서)
- Pull vs Push 모델 — Prometheus의 Pull 방식이 Push 방식 대비 갖는 장단점, Pushgateway의 용도
- Prometheus 데이터 모델 — TimeSeries = Metric Name + Label Set + Timestamp + Value, 카디널리티(Cardinality) 문제
- TSDB 저장 구조 — Head Block(메모리) → Compaction → Block(디스크), Chunk 단위 압축 저장
- 압축 알고리즘 — Timestamp: Delta-of-Delta 인코딩, Value: XOR 인코딩(Gorilla 알고리즘), 실제 압축률
- PromQL 내부 — Range Vector vs Instant Vector, rate()/increase() 계산 방법, Recording Rule로 비용 높은 쿼리 사전 계산
- 스크레이프와 서비스 디스커버리 — Kubernetes SD로 파드 자동 발견, relabeling으로 레이블 변환, 스크레이프 타임아웃 처리

### Chapter 5: 로그 — 구조화 로그와 중앙 수집 (5개 문서)
- 구조화 로그(Structured Logging) — JSON 로그가 텍스트 로그보다 검색에 유리한 이유, Logback/Log4j2에서 JSON 출력 설정
- MDC와 Trace ID 연결 — MDC(Mapped Diagnostic Context)에 traceId/spanId를 삽입해 로그와 트레이스를 연결하는 방법
- 로그 수집 아키텍처 — Filebeat/Fluentd가 파일을 tail하는 방식, 버퍼링과 백프레셔 처리, 중복 방지
- Loki — Prometheus와 같은 레이블 기반 로그 저장, LogQL 쿼리 언어, 인덱스 없는 저장 방식과 장단점
- 로그 레벨과 동적 변경 — Spring Boot Actuator로 런타임에 로그 레벨 변경, 운영 중 DEBUG 로그를 켜고 끄는 패턴

### Chapter 6: Grafana와 시각화 (4개 문서)
- Grafana 데이터 소스 아키텍처 — Prometheus/Loki/Tempo를 Grafana가 쿼리하는 방식, DataSource 플러그인 구조
- Exemplar — 메트릭(Prometheus)에서 특정 샘플이 발생한 Trace로 바로 점프하는 연결, Grafana에서 Tempo 연동
- 알림(Alerting) 설계 — Alertmanager 라우팅 규칙, 노이즈 줄이기(silence/inhibit), 증상 기반 vs 원인 기반 알림 설계
- RED 방법론과 USE 방법론 — Rate/Error/Duration으로 서비스를 진단하는 방법, Utilization/Saturation/Error로 리소스를 진단하는 방법

### Chapter 7: Spring과 Observability 통합 (6개 문서)
- Spring Boot Actuator — /actuator/metrics, /actuator/health 엔드포인트 내부, MeterRegistry 자동 구성
- Micrometer Tracing — Spring Cloud Sleuth에서 Micrometer Tracing으로의 전환, Tracer/Span/Observation API
- @Observed와 ObservationRegistry — AOP 기반 메서드 계측, ObservationHandler로 커스텀 처리 추가
- Spring WebFlux와 트레이싱 — 리액티브 파이프라인에서 Context 전파, 구독 경계를 넘는 TraceContext 유지
- Kubernetes 환경 배포 — Prometheus Operator(ServiceMonitor), 사이드카 패턴(OpenTelemetry Collector), auto-instrumentation
- 실전 진단 시나리오 — 응답 시간 급증 원인 찾기, 메트릭→트레이스→로그 연계 분석, SLO 위반 알림 설계

---

각 챕터는 4~6개 문서로 구성해줘. 총 37개 문서 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 실무에서 중요한가
## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)
## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)
## 🔬 내부 동작 원리 (바이트코드 수준, 프로토콜 수준, 저장 구조 분석)
## 💻 실전 실험 (재현 가능한 코드, ByteBuddy 예제, PromQL 실험)
## 📊 성능/비용 비교 (샘플링 비율별 오버헤드, TSDB 압축률 등)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. "어떤 도구를 쓰는가"보다 "그 도구가 내부에서 어떻게 동작하는가"에 초점
2. ByteBuddy 코드로 직접 Java Agent를 만들어보는 실험 포함 (원리를 손으로 구현)
3. traceparent 헤더를 직접 curl로 전달해보는 실험 포함
4. Spring Boot 예제 코드로 Micrometer/Tracing을 연결하는 실전 패턴
5. 운영 장애 시나리오 → 메트릭에서 트레이스, 트레이스에서 로그로 이어지는 연계 진단 흐름

**실험 환경**:
```yaml
# docker-compose.yml
services:
  prometheus:
    image: prom/prometheus:v2.47.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - promdata:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-remote-write-receiver'
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:10.0.0
    depends_on:
      - prometheus
      - loki
      - tempo
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    volumes:
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    ports:
      - "3000:3000"

  loki:
    image: grafana/loki:2.9.0
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
    ports:
      - "3100:3100"

  tempo:
    image: grafana/tempo:2.2.0
    command: ["-config.file=/etc/tempo.yaml"]
    volumes:
      - ./tempo.yaml:/etc/tempo.yaml
      - tempodata:/tmp/tempo
    ports:
      - "3200:3200"   # Tempo HTTP
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP

  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.88.0
    command: ["--config=/etc/otelcol-contrib/otel-collector.yaml"]
    volumes:
      - ./otel-collector.yaml:/etc/otelcol-contrib/otel-collector.yaml
    ports:
      - "4319:4317"   # OTLP gRPC (외부 앱 → Collector)
      - "4320:4318"   # OTLP HTTP
      - "8888:8888"   # Collector 내부 메트릭

  app:
    image: eclipse-temurin:21-jdk
    environment:
      - JAVA_TOOL_OPTIONS=-javaagent:/otel-agent/opentelemetry-javaagent.jar
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
      - OTEL_SERVICE_NAME=demo-app
      - OTEL_LOGS_EXPORTER=otlp
      - OTEL_METRICS_EXPORTER=otlp
      - OTEL_TRACES_EXPORTER=otlp
    volumes:
      - ./otel-agent:/otel-agent
    depends_on:
      - otel-collector

volumes:
  promdata:
  tempodata:
```

```bash
# 실험용 공통 명령어 세트

# TraceContext 헤더 직접 전달
curl -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01" \
     http://localhost:8080/api/orders

# Prometheus 메트릭 원본 확인
curl http://localhost:8080/actuator/prometheus | grep http_server_requests

# Trace 직접 조회 (Tempo API)
curl "http://localhost:3200/api/traces/<trace-id>"

# 바이트코드 조작 디버깅
java -javaagent:myagent.jar \
     -Dotel.javaagent.debug=true \
     MyApp

# JVM 플래그로 클래스 변환 확인
java -XX:+TraceClassLoading \
     -javaagent:myagent.jar MyApp
```

```java
// ByteBuddy로 간단한 메서드 타이머 구현 (실험 예제)
public class TimingAgent {
    public static void premain(String args, Instrumentation instrumentation) {
        new AgentBuilder.Default()
            .type(ElementMatchers.nameStartsWith("com.example"))
            .transform((builder, type, classLoader, module, protectionDomain) ->
                builder.method(ElementMatchers.isAnnotatedWith(Timed.class))
                    .intercept(Advice.to(TimingAdvice.class))
            )
            .installOn(instrumentation);
    }
}

public class TimingAdvice {
    @Advice.OnMethodEnter
    static long enter() {
        return System.nanoTime();
    }

    @Advice.OnMethodExit(onThrowable = Throwable.class)
    static void exit(@Advice.Origin String method,
                     @Advice.Enter long startTime) {
        long elapsed = System.nanoTime() - startTime;
        System.out.printf("Method %s took %d ms%n", method, elapsed / 1_000_000);
    }
}
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**:
   - mysql-deep-dive README 스타일 참고
   - "모니터링 도구를 쓰는 것을 넘어 Observability의 내부 원리를 이해한다"는 차별화 강조
   - jvm-deep-dive, java-concurrency, Spring 레포와의 연결 지점 명시
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료 섹션

README 작성 시 다음을 Reference로 포함해줘:
- Distributed Systems Observability (Cindy Sridharan)
- OpenTelemetry 공식 문서 — https://opentelemetry.io/docs/
- Prometheus 공식 문서 — https://prometheus.io/docs/
- Grafana 공식 문서 — https://grafana.com/docs/
- Gorilla: Fast, Scalable, Reliable Time Series Database (Facebook 논문)
- Micrometer 공식 문서 — https://micrometer.io/docs/

---

## 💡 핵심 분석 대상

```
Java Agent 바이트코드 조작 흐름:

-javaagent:opentelemetry-javaagent.jar 실행
  │
  ▼ premain() 호출 (main() 이전)
  │   Instrumentation API 획득
  │
  ▼ ClassFileTransformer 등록
  │   모든 클래스 로딩 시 변환 훅 등록
  │
  ▼ RestTemplate 클래스 로딩 시:
  │   원본 바이트코드 → ByteBuddy 변환 → 새 바이트코드
  │
  │   변환 전 execute():
  │     return doExecute(url, method, ...);
  │
  │   변환 후 execute():
  │     Span span = tracer.spanBuilder(url).startSpan();
  │     try (Scope scope = span.makeCurrent()) {
  │       injectTraceContext(request);  // traceparent 헤더 삽입
  │       return doExecute(url, method, ...);
  │     } finally { span.end(); }

분산 추적 전파:

Service A                              Service B
  Span(traceId=abc, spanId=111) 생성
  │
  ▼ RestTemplate 호출
    Headers:
      traceparent: 00-abc-111-01       ← TraceContext 주입
  │
  →─────────────────────────────────→
                                       Span(traceId=abc, spanId=222, parent=111) 생성
                                       → 같은 traceId로 이어진 Span

Tempo에서 조회:
  traceId=abc → [Span(111, /api/order), Span(222, /api/products)]
  → 두 서비스를 하나의 요청 흐름으로 시각화

Prometheus TSDB 압축 (Gorilla 알고리즘):

Timestamp 압축 (Delta-of-Delta):
  원본: [0, 15, 30, 45, 60, ...]
  Delta: [15, 15, 15, 15, ...]
  Delta-of-Delta: [0, 0, 0, ...]
  → VarInt 인코딩으로 1~2비트 사용 (64비트 대비 극적 압축)

Value 압축 (XOR):
  연속된 값의 XOR → 변화 부분만 저장
  카운터의 작은 증분 → XOR 결과 거의 0
  → 평균 1.37비트/샘플 (8바이트 대비 약 46배 압축)

RED 방법론:
  Rate:     rate(http_requests_total[5m])
  Errors:   rate(http_requests_total{status=~"5.."}[5m])
              / rate(http_requests_total[5m])
  Duration: histogram_quantile(0.99,
              rate(http_request_duration_seconds_bucket[5m]))
  → 이 세 메트릭으로 서비스 상태를 1분 안에 진단

메트릭-트레이스-로그 연결 (Exemplar):
  Prometheus 메트릭에서 p99 스파이크 감지
    → Exemplar: 해당 시점의 traceId 포함
    → Grafana에서 클릭 → Tempo에서 Trace 조회
    → Trace에서 느린 Span 발견
    → Span의 TraceId/SpanId → Loki에서 로그 조회
    → "Connection timeout to DB" 에러 발견
  = 메트릭 → 트레이스 → 로그 3단계 드릴다운
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (37개 목표)
- Docker Compose 실험 환경 (Prometheus + Grafana + Loki + Tempo + OTel Collector)
- jvm-deep-dive, java-concurrency, Spring 레포, kubernetes-deep-dive와 연결되는 지점 명시 (어느 문서에서 어느 레포와 연결하는지)

**준비됐으면 1단계 구조 설계부터 시작해줘!**
