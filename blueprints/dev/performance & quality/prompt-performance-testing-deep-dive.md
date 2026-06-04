# Performance Testing Deep Dive 레포지토리 제작 프롬프트

나는 "Performance Testing Deep Dive" 레포지토리를 만들려고 해.
API가 느리다고 느끼는 것과, 병목이 DB 쿼리인지 JVM GC인지 외부 API 호출인지를 수치로 특정하고, 부하 테스트로 한계치를 측정하며, 튜닝 전후를 정량적으로 비교하는 것은 다르다.

"느린 것 같다"와 "p99 응답시간이 3초이고 병목은 Connection Pool 고갈이며, Pool 크기를 20→50으로 늘리면 p99가 340ms로 줄어든다"를 증명하는 레포야.

## 📋 프로젝트 목표

**컨셉**: "성능이 느리다고 느끼는 것과, 병목 지점을 수치로 특정하고 개선을 증명하는 것은 다르다"

**핵심 차별화**:
1. 부하 테스트 설계 — 실제 트래픽 패턴을 재현하는 시나리오 설계 원칙
2. 병목 특정 방법론 — CPU/Memory/DB/Network 중 어디가 한계인지 데이터로 판단
3. JVM 성능 분석 — GC 로그, JFR, async-profiler로 코드 레벨 병목 찾기
4. 튜닝 사이클 — 측정 → 분석 → 가설 → 변경 → 재측정의 과학적 접근

**타겟 독자**:
- 성능 개선을 했지만 "체감상 빨라진 것 같다"고 말하는 개발자
- EXPLAIN으로 쿼리를 봤지만 실제 병목이 DB인지 확신 못하는 개발자
- 부하 테스트를 단순히 "서버가 안 죽으면 OK"로만 정의하는 개발자
- JVM GC 로그를 본 적 없는 개발자
- Connection Pool 크기를 기본값으로 두는 개발자
- k6/Gatling/nGrinder를 써봤지만 결과 해석 방법을 모르는 개발자

**선행 학습**:
- jvm-deep-dive (GC 알고리즘, JIT, 메모리 모델 — 병목 분석의 기반)
- linux-for-backend-deep-dive (CPU, 메모리, I/O, 네트워크 시스템 메트릭 이해)
- observability-deep-dive (Prometheus, Grafana로 메트릭 수집 — 시너지 최대)
- spring-data-transaction (HikariCP Connection Pool, 트랜잭션 범위)
- database-internals (슬로우 쿼리, 인덱스 — DB 병목 분석)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 성능 테스트 설계 원칙 (5개 문서)
- 성능 테스트 유형 분류 — Load(정상 부하)/Stress(한계 탐색)/Spike(갑작스러운 급증)/Soak(장시간 안정성) 테스트의 목적과 시나리오 설계 방법
- 성능 목표 정의 — SLI/SLO/SLA 차이, p50/p95/p99 백분위수가 평균보다 중요한 이유(Long Tail Latency), 목표치 없이 부하 테스트하는 것의 위험성
- 현실적 부하 시나리오 설계 — 프로덕션 트래픽 패턴 분석(Access Log 기반), Think Time 설정, Hot Spot 엔드포인트 특정, Warm-up 단계의 필요성
- 테스트 환경 구성 — Prod-like 환경 구성 원칙, 데이터 볼륨이 성능에 미치는 영향(빈 DB vs 1억 건 DB), 네트워크/하드웨어 차이가 결과를 어떻게 왜곡하는가
- 성능 기준선(Baseline) 수립 — 튜닝 전 기준 측정의 중요성, CI Pipeline에 성능 회귀 감지 통합, 배포마다 자동 Baseline 비교

### Chapter 2: k6 완전 분해 (6개 문서)
- k6 아키텍처 — Go 기반 엔진이 JavaScript 시나리오를 실행하는 방식, VU(Virtual User)가 실제 사용자를 시뮬레이션하는 원리, 고루틴 기반 동시성
- k6 시나리오 설계 — `options.scenarios`로 Ramping/Constant/Spike 패턴 구성, `stages`로 부하 단계 제어, `thresholds`로 자동 합격/실패 판정
- HTTP 요청과 검증 — `http.get/post`, Response 검증(`check`), 쿠키/세션 처리, 이전 응답 값을 다음 요청에 사용하는 패턴(로그인 토큰 재사용)
- 결과 분석 — `http_req_duration` 백분위수 해석, `http_req_failed` 에러율, `vus_max` 동시 접속자, 히스토그램으로 분포 확인
- k6 확장 — `k6 Cloud`와 분산 실행, InfluxDB + Grafana 대시보드 연동, Custom Metrics 정의(`new Counter/Gauge/Trend`)
- 실전 시나리오 — 전자상거래 주문 플로우(상품 조회 → 장바구니 → 결제) 시뮬레이션, 인증 포함 API 테스트, WebSocket 부하 테스트

### Chapter 3: 병목 특정 방법론 (7개 문서)
- USE 방법론 — Utilization(사용률)/Saturation(포화도)/Errors(에러)로 CPU·Memory·Disk·Network 체계적 점검, 시스템 메트릭으로 병목 후보 좁히기
- CPU 병목 분석 — `top`, `mpstat`으로 CPU 사용률 확인, User vs System vs IOWait 구분, CPU-Bound vs I/O-Bound 판단 기준
- 메모리 병목 분석 — JVM Heap vs Off-Heap, GC 빈도와 Stop-The-World 시간이 응답시간에 미치는 영향, GC 로그로 메모리 누수 패턴 감지
- DB 병목 분석 — Connection Pool 고갈 감지(`HikariCP` 메트릭), 슬로우 쿼리 식별, Lock Contention 확인(`SHOW PROCESSLIST`), Connection 대기 시간 측정
- 외부 API 호출 병목 — Timeout 설정 없는 외부 API 호출이 스레드 풀을 고갈시키는 원리, CircuitBreaker 없는 연쇄 장애, 비동기 처리로 전환 판단 기준
- 스레드 풀 분석 — Spring MVC 스레드 풀 고갈 감지, Thread Dump 분석(`jstack`, `kill -3`), BLOCKED/WAITING 스레드 원인 특정
- 네트워크 병목 분석 — 응답 크기 vs 응답시간 상관관계, TCP Connection 재사용(Keep-Alive), DNS 조회 지연, 직렬화/역직렬화 비용

### Chapter 4: JVM 성능 프로파일링 (6개 문서)
- GC 로그 분석 — `-Xlog:gc*` 옵션으로 GC 로그 수집, GC 유형별 Stop-The-World 시간, GC 빈도가 과도한 원인(객체 생성 속도 vs 수집 속도)
- Java Flight Recorder(JFR) — JVM 내부 이벤트 녹화(CPU 샘플링, 메모리 할당, Lock 경합, I/O 대기), JDK Mission Control로 시각화, 프로덕션에서 안전하게 사용하는 방법
- async-profiler — CPU/메모리 할당/Lock 프로파일링, Flame Graph 읽는 법(넓을수록 CPU 비중 높음), 특정 메서드가 전체 CPU의 몇 %를 차지하는지 측정
- Heap Dump 분석 — `jmap -dump` 또는 Actuator로 Heap Dump 수집, MAT(Memory Analyzer Tool)로 메모리 누수 원인 특정, 대용량 객체 보유 체인 추적
- JVM 튜닝 파라미터 — G1GC vs ZGC 선택 기준, `-Xms/-Xmx` 동일 설정 이유(힙 크기 재조정 비용), GC 튜닝의 트레이드오프(처리량 vs 지연시간)
- Spring Boot Actuator 성능 메트릭 — `/actuator/metrics/jvm.*`, `/actuator/metrics/hikaricp.*`, `/actuator/metrics/http.server.requests` 핵심 지표 해석

### Chapter 5: DB 성능 튜닝 (5개 문서)
- HikariCP 튜닝 — Pool 크기 공식(`Tn * (Cm - 1) + 1`), `maximumPoolSize`가 크다고 좋지 않은 이유(DB 연결 비용), `connectionTimeout`과 `idleTimeout` 설정 원칙
- 쿼리 성능 최적화 — EXPLAIN ANALYZE로 실행 계획 해석, 인덱스 추가 전후 비교, N+1 쿼리 감지(`p6spy`, `spring.jpa.show-sql`), Fetch Join으로 해결
- 커넥션 모니터링 — `SHOW STATUS LIKE 'Threads_connected'`, HikariCP `pending` 메트릭, 커넥션 풀 고갈 시 에러 패턴, 읽기 전용 트랜잭션 분리
- 캐시 전략과 효과 측정 — Redis 캐시 도입 전후 DB 쿼리 수 비교, Cache Hit Rate 측정, 캐시 워밍업 전략, 캐시 무효화 버그 탐지
- 페이징과 대용량 조회 최적화 — OFFSET 방식의 성능 저하 수치화, 커서 기반 페이징으로 전환 전후 비교, 스트리밍 처리(`JdbcTemplate.queryForStream`)

### Chapter 6: 튜닝 사이클과 결과 보고 (5개 문서)
- 과학적 튜닝 접근법 — 하나씩 변경하는 이유(다중 변경 시 원인 불명), 가설 → 측정 → 결론 순서, 튜닝 일지 작성법
- 튜닝 효과 정량화 — 개선 전후 p95/p99 비교표, 처리량(TPS) 변화, 에러율 변화, 인프라 비용 대비 개선 효과
- 성능 테스트 결과 보고서 — 비기술직도 이해할 수 있는 보고서 작성법, 개선 전후 그래프, 리스크 및 추가 권장 사항
- 성능 회귀 방지 — CI Pipeline에 k6 Smoke Test 통합, 임계값 초과 시 배포 차단, Baseline 자동 업데이트 전략
- 실전 케이스 스터디 — 전자상거래 주문 API 병목 분석부터 튜닝까지 전 과정(DB 커넥션 고갈 → 캐시 도입 → 인덱스 최적화 → 결과 측정)

### Chapter 7: 분산 환경 성능 테스트 (5개 문서)
- 분산 부하 테스트 — k6 Operator로 Kubernetes에서 분산 실행, 단일 머신 한계 극복, 테스트 결과 집계 방법
- 마이크로서비스 성능 — 서비스 간 호출 체인의 병목 특정(분산 추적 활용), 각 서비스의 독립적인 SLO 설정
- Kafka 성능 테스트 — Producer/Consumer 처리량 측정, Consumer Lag 시나리오 재현, 파티션 수 변경 전후 비교
- 컨테이너 환경 성능 — JVM의 컨테이너 CPU/메모리 인식 문제(`UseContainerSupport`), K8s Resource Limit이 성능에 미치는 영향, HPA 트리거 시점
- Chaos Engineering 기초 — 의도적 장애 주입으로 성능 저하 시나리오 검증, Chaos Monkey for Spring Boot, 장애 시 Circuit Breaker 동작 확인

---

각 챕터는 **5~7개 문서**로 구성해줘. 총 **39개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 실무에서 중요한가
## 😱 흔한 실수 (Before — 감으로 튜닝하는 접근)
## ✨ 올바른 접근 (After — 데이터로 증명하는 접근)
## 🔬 내부 동작 원리 (JVM/k6/DB 내부 분석)
## 💻 실전 실험 (k6 스크립트, async-profiler, GC 로그 분석)
## 📊 성능 비교 (튜닝 전후 p95/p99, TPS 비교표)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 모든 성능 개선은 Before/After 수치 비교 필수 (느낌이 아닌 데이터)
2. k6 스크립트는 그대로 실행 가능한 완전한 코드
3. 병목 분석은 항상 "어떤 메트릭을 보고 판단하는가"를 명시
4. JVM 분석 도구(JFR, async-profiler)는 실제 출력 예시 포함
5. observability-deep-dive 연결 — Prometheus/Grafana 대시보드로 메트릭 시각화

**실험 환경**:
```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      JAVA_OPTS: >
        -Xms512m -Xmx512m
        -XX:+UseG1GC
        -Xlog:gc*:file=/logs/gc.log
        -XX:StartFlightRecording=filename=/logs/app.jfr,duration=60s

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: perf_test
    volumes:
      - ./init-data.sql:/docker-entrypoint-initdb.d/init.sql

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  influxdb:
    image: influxdb:2.7
    ports:
      - "8086:8086"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
```

```javascript
// k6/load-test.js — 실전 부하 테스트 시나리오
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

const errorRate = new Rate('errors');
const orderDuration = new Trend('order_duration');

export const options = {
  scenarios: {
    normal_load: {
      executor: 'ramping-vus',
      stages: [
        { duration: '1m', target: 50 },   // Warm-up
        { duration: '5m', target: 200 },  // Load
        { duration: '2m', target: 0 },    // Cool-down
      ],
    },
  },
  thresholds: {
    'http_req_duration{name:order}': ['p(95)<500', 'p(99)<1000'],
    'errors': ['rate<0.01'],              // 에러율 1% 미만
  },
};

export default function () {
  // 1. 로그인
  const loginRes = http.post('http://localhost:8080/api/login', {
    username: 'user@test.com', password: 'password'
  });
  const token = loginRes.json('token');

  // 2. 상품 조회
  http.get('http://localhost:8080/api/products/1', {
    headers: { Authorization: `Bearer ${token}` },
    tags: { name: 'product_detail' },
  });

  // 3. 주문 생성
  const start = new Date();
  const orderRes = http.post('http://localhost:8080/api/orders', JSON.stringify({
    productId: 1, quantity: 2,
  }), {
    headers: { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json' },
    tags: { name: 'order' },
  });
  orderDuration.add(new Date() - start);

  check(orderRes, {
    'order created': (r) => r.status === 201,
    'order has id': (r) => r.json('id') !== undefined,
  }) || errorRate.add(1);

  sleep(1);
}
```

```bash
# 성능 분석 명령어 세트

# k6 실행 + InfluxDB 결과 저장
k6 run --out influxdb=http://localhost:8086/k6 k6/load-test.js

# JVM Thread Dump (스레드 상태 분석)
jstack $(jps | grep App | awk '{print $1}')

# async-profiler CPU 프로파일링
./profiler.sh -d 60 -f flamegraph.html $(jps | grep App | awk '{print $1}')

# GC 로그 실시간 분석
tail -f /logs/gc.log | grep -E "GC|pause"

# HikariCP 메트릭 (Actuator)
curl http://localhost:8080/actuator/metrics/hikaricp.connections.pending

# MySQL 현재 쿼리 모니터링
mysql -e "SHOW FULL PROCESSLIST" | grep -v Sleep
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**:
   - observability-deep-dive README 스타일 참고
   - "느낌이 아닌 데이터로 성능을 증명한다"는 차별화 강조
   - jvm-deep-dive, observability-deep-dive, database-internals와의 연결 지점 명시
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료

- k6 공식 문서 — https://grafana.com/docs/k6/latest/
- async-profiler — https://github.com/async-profiler/async-profiler
- Gatling 공식 문서 — https://docs.gatling.io/
- Java Flight Recorder — https://docs.oracle.com/javacomponents/jmc.htm
- Brendan Gregg — Systems Performance (USE 방법론, Flame Graph 저자)
- HikariCP 설정 가이드 — https://github.com/brettwooldridge/HikariCP/wiki/About-Pool-Sizing
- Google SRE Book — https://sre.google/sre-book/ (SLI/SLO/SLA 챕터)

---

## 💡 핵심 분석 대상

```
성능 병목 특정 흐름:

k6 부하 테스트 실행
  → p99 응답시간 3,200ms (목표: 500ms)
  → 에러율 8% (목표: 1% 미만)
  │
  ▼ [1단계: 시스템 메트릭 확인]
  CPU: 45% → 병목 아님
  Memory: 75% → 의심
  DB 연결 대기: pending=18 / max=20 → 🎯 병목 후보

  ▼ [2단계: JVM 분석]
  GC 로그: Full GC 5초마다 → Stop-The-World 800ms
  Thread Dump: 18개 스레드 WAITING on HikariPool
  → Connection Pool 고갈 확인

  ▼ [3단계: DB 분석]
  SHOW PROCESSLIST: 20개 연결 모두 사용 중
  슬로우 쿼리: SELECT * FROM orders WHERE user_id=? → 0.8초
  EXPLAIN: Full Table Scan (인덱스 없음)

  ▼ [병목 원인 특정]
  1. user_id 인덱스 누락 → 쿼리 0.8초
  2. 느린 쿼리로 Connection 오래 점유
  3. Pool 고갈(max=20) → 대기 스레드 급증
  4. GC 압박으로 추가 지연

  ▼ [튜닝 적용 및 재측정]
  1. ALTER TABLE orders ADD INDEX idx_user_id (user_id)
     쿼리 0.8s → 3ms
  2. maximumPoolSize 20 → 30
  3. 재측정: p99 = 180ms ✓, 에러율 = 0.02% ✓

p99 vs 평균 차이 중요성:
  평균 응답시간: 120ms (정상처럼 보임)
  p99 응답시간:  3,200ms (100명 중 1명은 3.2초 기다림)
  → 평균은 이상 징후를 숨김
  → SLO는 반드시 백분위수(p95/p99)로 정의
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (39개 목표)
- Docker Compose 실험 환경 (App + MySQL + InfluxDB + Grafana)
- jvm-deep-dive, observability-deep-dive, database-internals와 연결되는 지점 명시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
