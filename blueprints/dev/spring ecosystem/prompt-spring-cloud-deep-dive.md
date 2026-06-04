# Spring Cloud Deep Dive 레포지토리 제작 프롬프트

나는 "Spring Cloud Deep Dive" 레포지토리를 만들려고 해.
Microservices Architecture를 위한 Config Server, API Gateway, Service Discovery, Circuit Breaker의 원리를 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "분산 시스템을 위한 Spring Cloud 패턴 완전 정복"

**핵심 차별화**:
1. Config Server가 설정을 동적으로 리프레시하는 원리
2. Gateway의 라우팅과 필터 체인
3. Service Discovery (Eureka) 동작 메커니즘
4. Circuit Breaker (Resilience4j) 상태 전이

**타겟 독자**:
- MSA를 시작하는 개발자
- Config Server를 써봤지만 원리를 모르는 개발자
- Gateway 필터를 커스터마이징해야 하는 개발자
- Circuit Breaker의 OPEN/HALF_OPEN/CLOSED 상태를 설명 못하는 개발자

**선행 학습**:
- Spring Core Deep Dive (AOP, Filter 이해)
- Spring MVC Deep Dive (Request 처리 이해)
- Spring Boot Internals (Auto-configuration 이해)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Config Server (6개 문서)
- Externalized Configuration 필요성
- Config Server 동작 원리 (Git Backend)
- Config Client 자동 설정
- @RefreshScope와 동적 갱신
- Encryption/Decryption (대칭키, 비대칭키)
- Config Server High Availability

### Chapter 2: Service Discovery with Eureka (6개 문서)
- Service Registry 패턴
- Eureka Server 내부 구조
- Eureka Client 등록 과정 (Heartbeat)
- Service Instance Metadata
- Client-Side Load Balancing (LoadBalancerClient)
- Self-Preservation Mode

### Chapter 3: API Gateway (Spring Cloud Gateway) (7개 문서)
- Gateway vs Zuul (Reactive vs Blocking)
- Route Predicate Factory (Path, Method, Header)
- Gateway Filter Factory (AddRequestHeader, Retry)
- Global Filter 체인 실행 순서
- Route Locator 동적 라우팅
- Gateway Timeout & Circuit Breaker 통합
- Custom Filter 작성

### Chapter 4: Load Balancing (5개 문서)
- Ribbon vs Spring Cloud LoadBalancer
- LoadBalancerClient 동작 원리
- Load Balancing 알고리즘 (Round Robin, Weighted)
- Retry 전략
- Custom LoadBalancer 구현

### Chapter 5: Circuit Breaker (Resilience4j) (7개 문서)
- Circuit Breaker 패턴과 필요성
- CLOSED → OPEN → HALF_OPEN 상태 전이
- Failure Rate Threshold 계산
- Slow Call 탐지
- Fallback 메서드 작성
- Bulkhead 패턴 (격리)
- Rate Limiter 통합

### Chapter 6: Distributed Tracing (5개 문서)
- Sleuth와 Trace ID/Span ID
- Zipkin Server 연동
- MDC를 통한 로그 추적
- Baggage Propagation
- Custom Span Tags

### Chapter 7: Advanced Cloud Patterns (4개 문서)
- Event-Driven Architecture (Spring Cloud Stream)
- Saga Pattern (분산 트랜잭션)
- API Composition
- CQRS (Command Query Responsibility Segregation)

---

각 챕터는 **4~7개 문서**로 구성해줘. 총 **40개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 MSA에서 필요한가
## 😱 잘못된 구성 (Before - 단일 장애점)
## ✨ 올바른 패턴 (After - 고가용성)
## 🔬 내부 동작 원리 (Spring Cloud 소스 추적)
## 💻 실전 구성 (Docker Compose로 전체 환경)
## 🌐 분산 시스템 시나리오 (장애 상황 테스트)
## 🤔 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. Docker Compose로 MSA 환경 구성
2. 서비스 간 호출 흐름 추적
3. 장애 시뮬레이션 (서비스 다운, 네트워크 지연)
4. Actuator로 메트릭 확인
5. 분산 추적 (Zipkin UI)

**실험 환경**:
- Docker Compose (Config Server, Eureka, Gateway, Services)
- Zipkin Server
- Circuit Breaker 상태 확인 (Actuator)
- Service Discovery Dashboard (Eureka UI)

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**: 
   - Spring Boot Internals README 참고
   - Spring Cloud에 맞게 변형
   - MSA 아키텍처 강조
3. **챕터별 문서 작성**: 
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료

<Spring Core Deep Dive README를 여기에 붙여넣기>

<Spring Boot Internals README를 여기에 붙여넣기>

<Spring MVC Deep Dive README를 여기에 붙여넣기>

위 README들을 참고해서 비슷한 구조로 Spring Cloud Deep Dive 버전을 만들어줘.

**차이점**:
- 주제: MSA 패턴과 Spring Cloud
- 초점: 분산 시스템 문제 해결
- 실험: Docker Compose, 장애 시뮬레이션

---

## 💡 핵심 분석 대상

```
MSA 호출 흐름:

Client → Gateway (Port 8080)
       → Eureka (Service Discovery)
       → LoadBalancer (Client-Side)
       → Service A (Port 8081)
         → Circuit Breaker
         → Service B (Port 8082)
           → Config Server (설정 로드)

장애 시나리오:
1. Service B 다운 → Circuit Breaker OPEN → Fallback
2. Config Server 변경 → @RefreshScope → 동적 리프레시
3. Service A 인스턴스 3개 → Round Robin Load Balancing
4. Trace ID로 전체 요청 추적 (Zipkin)
```

이 흐름을 완전히 분해!

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (40개 목표)
- Docker Compose 구성 예시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
