# Spring WebFlux Deep Dive 레포지토리 제작 프롬프트

나는 "Spring WebFlux Deep Dive" 레포지토리를 만들려고 해.
WebFlux를 쓰는 것과, 왜 스레드 하나로 수천 개의 요청을 동시에 처리할 수 있는지, Backpressure가 어떻게 소비자가 생산자를 조절하는지, Reactor가 어떻게 비동기 파이프라인을 구성하는지를 완전히 파헤치는 레포야.

"RestTemplate 대신 WebClient를 쓰면 되겠지"와 "왜 스레드 풀 기반 모델이 I/O 집약적 서비스에서 한계를 가지는지, 이벤트 루프가 어떻게 수천 개의 연결을 소수의 스레드로 처리하는지, Mono와 Flux가 왜 즉시 실행되지 않는지 아는 것"의 차이를 만드는 레포야.

## 📋 프로젝트 목표

**컨셉**: "스레드를 늘리는 것과, 스레드가 I/O를 기다리는 시간을 없애는 것은 다르다"

**핵심 차별화**:
1. 블로킹 vs 논블로킹 — 스레드가 I/O를 기다리는 동안 무슨 일이 생기는가, C10K 문제가 왜 스레드 기반 모델의 한계인가
2. Project Reactor 내부 — Mono/Flux가 왜 지연 평가(Lazy Evaluation)인가, 연산자 체인이 어떻게 파이프라인을 구성하는가
3. Netty 이벤트 루프 — epoll/kqueue로 수천 개의 연결을 소수의 스레드로 처리하는 원리
4. Backpressure — 생산자가 소비자보다 빠를 때 시스템이 붕괴하지 않도록 조절하는 메커니즘

**타겟 독자**:
- WebClient를 쓰지만 subscribe()를 빠뜨려서 요청이 실행되지 않는 버그를 낸 개발자
- block()을 WebFlux 코드 안에서 호출하다 데드락을 경험한 개발자
- Reactive Stream이 왜 Iterable과 다른지, push vs pull 방식의 차이를 모르는 개발자
- flatMap과 concatMap의 차이를 모르는 개발자
- R2DBC 없이 WebFlux에서 JPA를 쓰다 스레드 고갈을 경험한 개발자
- WebFlux와 MVC 중 무엇을 선택해야 하는지 기준이 없는 개발자

**선행 학습**:
- spring-mvc-deep-dive (DispatcherServlet, 스레드 모델 — WebFlux와 대비 필수)
- spring-boot-internals (Auto-configuration — WebFlux 자동 설정)
- linux-for-backend-deep-dive (epoll, 논블로킹 I/O — Netty의 기반)
- network-deep-dive (TCP 연결, HTTP/2 — WebFlux 통신 레이어)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 왜 Reactive인가 — 블로킹 모델의 한계 (6개 문서)
- 전통적 스레드 기반 모델의 한계 — 요청당 스레드(Thread-per-Request) 모델, 스레드가 I/O를 기다리는 동안 CPU가 유휴 상태가 되는 원리, 스레드 컨텍스트 스위칭 비용, 1000개 동시 요청 시 스레드 1000개가 필요한 이유
- C10K 문제 — 동시 연결 10,000개를 처리할 때 스레드 기반 모델이 실패하는 이유, 메모리 계산(스레드 하나 ~1MB × 10000 = 10GB), Nginx가 Apache를 이긴 이유
- 논블로킹 I/O의 원리 — I/O 작업을 커널에 위임하고 콜백으로 완료를 통보받는 방식, epoll이 수천 개의 소켓을 하나의 시스템 콜로 감시하는 방법, 이벤트 루프가 I/O 완료 이벤트를 처리하는 흐름
- Reactive Programming의 탄생 — 비동기 콜백의 Callback Hell 문제, Promise/Future가 해결한 것과 못 한 것, Reactive Streams 스펙이 표준화한 것(Publisher/Subscriber/Subscription/Processor)
- WebFlux vs Spring MVC 선택 기준 — I/O 집약적 서비스(외부 API 다수 호출, 스트리밍)에서 WebFlux 우위, CPU 집약적 서비스에서 MVC와 차이 없는 이유, 팀 학습 비용, 블로킹 라이브러리(JPA) 사용 시 WebFlux의 함정
- Reactive Streams 스펙 완전 분해 — Publisher가 Subscriber에게 데이터를 push하는 방식, Subscription.request(n)으로 소비자가 생산자를 조절하는 Backpressure, onNext/onError/onComplete 신호

### Chapter 2: Project Reactor 핵심 (8개 문서)
- Mono와 Flux — Mono(0 또는 1개 값), Flux(0개 이상 값), 왜 즉시 실행되지 않는가(지연 평가 = Cold Publisher), subscribe() 호출 전까지 아무 일도 일어나지 않는 이유
- Reactor 연산자 완전 분해 — map(동기 변환) vs flatMap(비동기 변환), concatMap(순서 보장) vs flatMap(순서 무보장), switchMap(최신 것만), mergeMap 비교, 실수하기 쉬운 패턴
- 에러 처리 — onErrorReturn(기본값 반환), onErrorResume(대체 Publisher), onErrorMap(에러 변환), doOnError(로깅), retry/retryWhen(재시도 전략), Reactive에서 try-catch가 없는 이유
- 스케줄러(Scheduler) — subscribeOn(구독 시 실행 스레드), publishOn(이후 연산자 실행 스레드), Schedulers.boundedElastic()(블로킹 작업용), Schedulers.parallel()(CPU 집약적), 스레드 전환이 일어나는 시점
- Cold vs Hot Publisher — Cold: 구독할 때마다 새로운 데이터 스트림, Hot: 구독 시점과 관계없이 진행 중인 스트림, publish().refCount()로 Cold를 Hot으로, 실시간 이벤트 스트림에서 Hot Publisher 필요성
- Backpressure 전략 — BUFFER(소비 못한 데이터 버퍼), DROP(넘치는 데이터 버림), LATEST(최신 것만 유지), ERROR(소비자가 따라오지 못하면 에러), 각 전략의 메모리/정확성 트레이드오프
- Context — Reactive 파이프라인에서 ThreadLocal을 사용할 수 없는 이유, Reactor Context로 파이프라인 전체에 데이터 전달, Spring Security의 ReactiveSecurityContextHolder 동작 원리
- 테스트 — StepVerifier로 Mono/Flux 테스트, 시간 기반 연산자(delayElements) 테스트, TestPublisher로 커스텀 시나리오, WebTestClient로 통합 테스트

### Chapter 3: Netty와 이벤트 루프 (5개 문서)
- Netty 아키텍처 완전 분해 — Boss EventLoopGroup(연결 수락), Worker EventLoopGroup(I/O 처리), Channel Pipeline, ChannelHandler 체인, Spring WebFlux가 Netty를 서버로 사용하는 방식
- EventLoop 내부 동작 — 하나의 EventLoop가 여러 Channel을 담당, I/O 이벤트와 태스크 큐를 하나의 스레드에서 처리, CPU 코어 수에 맞춘 EventLoopGroup 크기 설정 이유
- ChannelPipeline과 Handler — Inbound(읽기 방향) vs Outbound(쓰기 방향) 핸들러, HTTP 디코딩/인코딩 핸들러 체인, WebFlux가 Netty 핸들러를 Spring Router에 연결하는 방법
- Netty에서 블로킹 코드의 위험 — EventLoop 스레드에서 블로킹 호출 시 전체 채널이 멈추는 이유, Schedulers.boundedElastic()로 블로킹 작업을 별도 스레드로 오프로딩하는 패턴
- 연결 관리 — Keep-Alive, HTTP/2 멀티플렉싱에서 하나의 연결로 여러 요청을 처리, 연결 풀 관리(ConnectionProvider), WebClient의 연결 재사용 전략

### Chapter 4: Spring WebFlux 핵심 (7개 문서)
- WebFlux 아키텍처 — DispatcherHandler가 DispatcherServlet을 대체, HandlerMapping/HandlerAdapter/HandlerResultHandler의 Reactive 버전, Spring MVC와 동일한 어노테이션(@GetMapping 등) 사용 가능한 이유
- 어노테이션 기반 vs 함수형 엔드포인트 — @RestController + @GetMapping(MVC 스타일), RouterFunction + HandlerFunction(함수형 스타일), 각각의 장단점과 선택 기준
- WebClient 완전 분해 — RestTemplate과 근본적으로 다른 이유(논블로킹 HTTP 클라이언트), retrieve() vs exchangeToMono(), 요청 헤더/바디 설정, 에러 처리, 재시도, 타임아웃, ConnectionProvider 설정
- WebClient 고급 패턴 — 여러 외부 API 병렬 호출(Mono.zip, Flux.merge), 순차 호출 vs 병렬 호출 성능 차이, 서킷 브레이커(Resilience4j Reactive) 통합, WebClient를 Bean으로 관리하는 방법
- 스트리밍 응답 — Server-Sent Events(SSE)로 서버에서 클라이언트로 실시간 데이터 push, Flux<ServerSentEvent>로 SSE 구현, WebSocket 양방향 통신 구현
- 요청/응답 처리 — ServerRequest/ServerResponse, Reactive 기반 요청 본문 읽기, multipart/form-data 처리, 파일 스트리밍 업로드/다운로드
- WebFilter — Reactive 기반 필터 체인, 인증/로깅/CORS 처리, WebFilter 순서 지정, ExchangeFilterFunction으로 WebClient 필터

### Chapter 5: R2DBC와 Reactive 데이터 액세스 (5개 문서)
- JPA와 WebFlux의 충돌 — JPA가 내부적으로 블로킹 JDBC를 사용하는 이유, WebFlux에서 JPA를 쓰면 EventLoop 스레드가 블로킹되는 문제, Schedulers.boundedElastic()로 임시 해결하는 방법의 한계
- R2DBC 완전 분해 — JDBC를 완전히 대체하는 논블로킹 DB 드라이버, DatabaseClient API, Spring Data R2DBC Repository 사용법, JPA와 비교한 기능 제약(지연 로딩 없음, 복잡한 조인 없음)
- Reactive Transaction — R2DBC에서 트랜잭션 관리, @Transactional이 Reactive 컨텍스트에서 동작하는 방식, TransactionalOperator로 프로그래밍 방식 트랜잭션
- R2DBC 실전 패턴 — N+1 문제를 Reactive에서 해결하는 방법, DatabaseClient로 복잡한 쿼리 작성, Flux.flatMap으로 배치 처리, R2DBC Connection Pool 설정
- Redis Reactive — Lettuce 기반 Reactive Redis 클라이언트, ReactiveRedisTemplate 사용법, Reactive Pub/Sub, WebFlux + Redis 조합의 실시간 기능 구현

### Chapter 6: Spring Security Reactive (4개 문서)
- Reactive Security 아키텍처 — SecurityWebFilterChain이 FilterChainProxy를 대체, ReactiveAuthenticationManager, ServerSecurityContextRepository, 기존 Spring Security와의 차이
- JWT 인증 Reactive 구현 — WebFilter에서 JWT 추출/검증, ReactiveSecurityContextHolder에 인증 정보 저장, 각 요청마다 SecurityContext를 Reactive Context에 전달하는 방법
- Method Security — @PreAuthorize가 Reactive 메서드에서 동작하는 방식, Mono<Authentication>으로 현재 사용자 접근, 역할 기반 접근 제어
- OAuth2 Reactive — Reactive 기반 OAuth2 Login, ReactiveOAuth2AuthorizedClientManager, WebClient에 OAuth2 토큰 자동 첨부

### Chapter 7: 성능 튜닝과 실전 패턴 (5개 문서)
- WebFlux 성능 측정 — 스레드 기반 MVC vs 이벤트 루프 기반 WebFlux 처리량 비교, 외부 API 호출 수에 따른 차이, wrk/k6로 부하 테스트
- 운영 중 발생하는 문제 패턴 — 블로킹 코드 혼입 감지(BlockHound 라이브러리), 메모리 누수(구독 해제 안 된 Flux), 스케줄러 선택 실수로 인한 성능 저하
- Reactive Caching — Caffeine/Redis와 Reactor 통합, Mono.cache()로 중복 구독 방지, 캐시된 Publisher의 갱신 전략
- 마이크로서비스에서 WebFlux — 서비스 간 WebClient 호출 체인, Reactive 기반 Circuit Breaker, 타임아웃과 폴백, 분산 추적(Micrometer + Zipkin) Reactive 통합
- 언제 WebFlux를 쓰지 말아야 하는가 — 블로킹 라이브러리 의존성이 많은 경우, 팀이 Reactive 패러다임에 익숙하지 않은 경우, 단순 CRUD 서비스, 디버깅 어려움 허용 범위

---

총 **40개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 WebFlux에서 중요한가
## 😱 흔한 실수 (Before — 블로킹 방식 또는 잘못된 Reactive 코드)
## ✨ 올바른 접근 (After — 올바른 Reactive 코드)
## 🔬 내부 동작 원리 (Reactor 연산자 내부, Netty 이벤트 루프, 신호 전파)
## 💻 실전 코드 (Spring WebFlux + WebClient + R2DBC 기반)
## 📊 성능 비교 (MVC vs WebFlux 처리량, 블로킹 vs 논블로킹 응답 시간)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 모든 연산자 설명은 마블 다이어그램(텍스트 버전)으로 시각화
2. 블로킹 코드 vs 논블로킹 코드를 항상 Before/After로 비교
3. "이 코드가 언제 실행되는가"를 항상 명시 — subscribe() 전과 후
4. BlockHound로 블로킹 코드 감지 실험 포함
5. spring-mvc-deep-dive 레포와 대비 — DispatcherServlet vs DispatcherHandler

**실험 환경**:
```yaml
# docker-compose.yml
services:
  webflux-app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_R2DBC_URL: r2dbc:postgresql://postgres:5432/webflux_db
      SPRING_REDIS_HOST: redis

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: webflux_db
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"

  redis:
    image: redis:7.0
    ports:
      - "6379:6379"

  # 외부 API 목킹 (느린 응답 시뮬레이션)
  wiremock:
    image: wiremock/wiremock:latest
    ports:
      - "8090:8080"
    volumes:
      - ./wiremock:/home/wiremock

  # 부하 테스트
  k6:
    image: grafana/k6:latest
    volumes:
      - ./k6:/scripts
```

---

## 💡 핵심 분석 대상

```
=== 블로킹 모델 vs 이벤트 루프 ===

블로킹 (Spring MVC + Tomcat):
요청 1 → 스레드 A 할당
  └── DB 쿼리 대기 (200ms) → 스레드 A 블로킹 중 (CPU 유휴)
요청 2 → 스레드 B 할당
요청 3 → 스레드 C 할당
...
요청 200 → 스레드 고갈 → 큐 대기 → 응답 지연

논블로킹 (Spring WebFlux + Netty):
요청 1 → EventLoop 스레드 A
  └── DB 쿼리 시작 → 커널에 위임 → 스레드 A는 다음 요청으로
요청 2 → EventLoop 스레드 A (같은 스레드!)
요청 3 → EventLoop 스레드 A
...
DB 응답 도착 → 이벤트 큐 → EventLoop가 콜백 실행

결과: 스레드 8개 (CPU 코어 수)로 10,000 동시 연결 처리 가능

=== Reactor 파이프라인 실행 원리 ===

Mono<String> result = webClient.get()
    .uri("/users/1")
    .retrieve()
    .bodyToMono(User.class)         // 아직 실행 안 됨
    .map(user -> user.getName())    // 아직 실행 안 됨
    .flatMap(name ->                // 아직 실행 안 됨
        emailService.findByName(name))
    .onErrorReturn("unknown");      // 아직 실행 안 됨

// 여기까지는 파이프라인 "설계도"만 만든 것
// subscribe() 호출 시 실행 시작:
result.subscribe(
    value -> log.info("Result: {}", value),
    error -> log.error("Error", error)
);

신호 전파 방향:
subscribe() 호출
  → Subscription 생성 (아래 → 위로)
  → request(n) 신호 (소비자 → 생산자)
  → 데이터 흐름 시작 (위 → 아래)
  → onNext("value") 신호
  → map/flatMap 연산자 실행
  → 최종 Subscriber에게 전달

=== Backpressure 동작 ===

Flux.range(1, 1_000_000)           // 100만 개 생성
    .map(i -> heavyProcess(i))      // 처리 느림
    .subscribe(
        value -> slowConsumer(value) // 소비 더 느림
    );

Backpressure 없으면:
  생산자: 100만 개를 메모리에 버퍼 → OutOfMemoryError

Backpressure 있으면:
  소비자: request(256) → 256개만 요청
  생산자: 256개만 생성 후 대기
  소비자: 처리 완료 → request(256) 다시 요청
  → 메모리 O(256) 수준으로 제한

=== flatMap vs concatMap ===

// flatMap: 순서 무보장, 최대 동시성
Flux.range(1, 5)
    .flatMap(i -> callExternalApi(i))  // 5개 동시 호출
// 결과: 3, 1, 4, 2, 5 (응답 속도 순)

// concatMap: 순서 보장, 순차 처리
Flux.range(1, 5)
    .concatMap(i -> callExternalApi(i))  // 1개씩 순차 호출
// 결과: 1, 2, 3, 4, 5 (입력 순서)
// 성능: flatMap보다 느림 (동시성 없음)
```

---

## 📚 참고 자료

- Project Reactor 공식 문서 — https://projectreactor.io/docs/core/release/reference/
- Spring WebFlux 공식 문서 — https://docs.spring.io/spring-framework/docs/current/reference/html/web-reactive.html
- Reactive Programming with RxJava (Tomasz Nurkiewicz, Ben Christensen)
- Hands-On Reactive Programming in Spring 5 (Oleh Dokuka, Igor Lozynskyi)
- Netty in Action (Norman Maurer, Marvin Wolfthal)
- BlockHound — https://github.com/reactor/BlockHound

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~8개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (40개 목표)
- Docker Compose 실험 환경 (WebFlux App + PostgreSQL R2DBC + Redis + WireMock)
- spring-mvc-deep-dive / linux-for-backend-deep-dive / network-deep-dive와 연결되는 지점 명시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
