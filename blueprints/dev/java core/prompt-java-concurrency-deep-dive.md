# Java Concurrency Deep Dive 레포지토리 제작 프롬프트

나는 "Java Concurrency Deep Dive" 레포지토리를 만들려고 해.
스레드를 만드는 것과, JVM이 Virtual Thread를 캐리어 스레드 위에서 어떻게 스케줄링하는지, synchronized와 ReentrantLock이 내부에서 어떻게 다르게 구현되는지, JMM이 가시성을 어떻게 보장하는지를 완전히 파헤치는 레포야.

jvm-deep-dive에서 JVM 구조를 배웠다면, 이제 "그래서 여러 스레드가 동시에 실행될 때 JVM 내부에서 무슨 일이 일어나는가"를 배우는 레포야.

## 📋 프로젝트 목표

**컨셉**: "스레드를 만드는 것과, JVM이 Virtual Thread를 캐리어 스레드 위에서 어떻게 스케줄링하고 컨텍스트 스위칭하는지 아는 것은 다르다"

**핵심 차별화**:
1. JMM 완전 분해 — happens-before가 실제로 무엇을 보장하는가, volatile이 충분하지 않은 경우
2. 락의 내부 구현 — synchronized가 Object Header의 Mark Word를 어떻게 사용하는가
3. Virtual Thread — Project Loom의 Continuation으로 어떻게 블로킹 없이 컨텍스트 스위칭하는가
4. Lock-Free 알고리즘 — CAS가 CPU 명령어 수준에서 어떻게 원자성을 보장하는가

**타겟 독자**:
- synchronized를 쓰지만 biased lock/thin lock/fat lock으로 확장되는 과정을 모르는 개발자
- volatile을 쓰지만 가시성과 원자성이 다른 보장임을 설명 못하는 개발자
- CompletableFuture를 쓰지만 Virtual Thread와 무엇이 다른지 모르는 개발자
- AtomicInteger가 왜 synchronized보다 빠른지 CAS 수준에서 설명 못하는 개발자
- ConcurrentHashMap을 쓰지만 내부 세그먼트 락 구조를 모르는 개발자
- ThreadLocal을 쓰지만 Virtual Thread에서 왜 주의해야 하는지 모르는 개발자
- Spring @Async, @Transactional의 스레드 모델을 JVM 관점에서 설명 못하는 개발자

**선행 학습**:
- jvm-deep-dive (JVM 구조, 클래스로딩, GC 기초 필수)
- linux-for-backend-deep-dive (OS 스레드, 컨텍스트 스위칭, epoll 이해 시 Virtual Thread 챕터 깊이 배가)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: OS 스레드와 Java 스레드 — 기반 원리 (5개 문서)
- OS 스레드 모델 — 커널 스레드와 사용자 스레드, Java 스레드가 OS 스레드에 1:1 매핑되는 이유와 비용
- 컨텍스트 스위칭 비용의 실체 — 레지스터 저장/복원, TLB 플러시, CPU 캐시 무효화가 처리량에 미치는 영향
- 스레드 생성과 Thread Pool — `new Thread()` 비용, ThreadPoolExecutor 내부 큐/스레드 관리, CallerRunsPolicy 동작
- 스레드 상태 머신 — NEW/RUNNABLE/WAITING/TIMED_WAITING/BLOCKED/TERMINATED 전환 조건과 원인
- 데몬 스레드와 JVM 종료 — JVM이 종료되는 조건, 데몬 스레드가 강제 종료되는 방식, ShutdownHook

### Chapter 2: Java Memory Model — 가시성과 순서의 보장 (6개 문서)
- 하드웨어 메모리 모델 — CPU 캐시 계층(L1/L2/L3), Write Buffer, 스토어 포워딩이 가시성 문제를 일으키는 원리
- JMM(Java Memory Model) — 주 메모리와 작업 메모리, JVM이 하드웨어 위에 추상화를 만드는 방식
- happens-before 관계 — 8가지 규칙(프로그램 순서, 모니터 락, volatile 쓰기-읽기 등)과 전이적 특성
- volatile 완전 분해 — 가시성 보장(캐시 무효화), 재정렬 방지(메모리 펜스), volatile만으로 충분하지 않은 케이스
- 명령어 재정렬 — 컴파일러 재정렬, CPU Out-of-Order Execution, JIT 최적화가 순서를 바꾸는 방식
- Double-Checked Locking의 함정 — Java 5 이전에 왜 깨졌는가, volatile 추가로 어떻게 수정됐는가

### Chapter 3: 락의 내부 구현 — synchronized에서 StampedLock까지 (7개 문서)
- Object Header와 Mark Word — 모든 Java 객체가 락 상태를 저장하는 방식, biased/thin/fat lock 계층
- synchronized 내부 — Biased Lock(바이어스 락) → Lightweight Lock(경량 락) → Heavyweight Lock(중량 락) 확장 과정
- ReentrantLock 내부 — AQS(AbstractQueuedSynchronizer) 대기 큐 구조, fair vs non-fair 락의 성능 차이
- ReadWriteLock — 읽기/쓰기 락 분리로 동시성을 높이는 원리, write starvation 발생 조건
- StampedLock — Optimistic Read로 읽기 락 없이 읽기 시도, validateStamp로 검증하는 낙관적 동시성 제어
- 조건 변수 — `Object.wait()/notify()/notifyAll()` vs `Condition.await()/signal()`, Spurious Wakeup 처리
- 락 성능 비교 — synchronized vs ReentrantLock vs StampedLock 벤치마크, JIT가 synchronized를 최적화하는 방식

### Chapter 4: Lock-Free 알고리즘과 원자적 연산 (6개 문서)
- CAS(Compare-And-Swap) — CPU 명령어(LOCK CMPXCHG) 수준에서 원자성이 보장되는 원리, ABA 문제
- AtomicInteger/AtomicLong 내부 — sun.misc.Unsafe의 compareAndSwapInt 사용, VarHandle로의 전환(Java 9+)
- AtomicReference와 AtomicStampedReference — 참조 타입의 원자적 교체, ABA 문제를 스탬프로 해결하는 방법
- LongAdder vs AtomicLong — 고경쟁 환경에서 Cell 분산으로 CAS 실패를 줄이는 스트라이프 기법
- Lock-Free 자료구조 — ConcurrentLinkedQueue 내부(Michael-Scott Queue), Non-Blocking 알고리즘 설계 원칙
- 메모리 오더링 — VarHandle의 getAcquire/setRelease/getOpaque 각각이 보장하는 순서 수준과 성능 차이

### Chapter 5: 동시성 자료구조 내부 구현 (5개 문서)
- ConcurrentHashMap 진화 — Java 7 세그먼트 락 → Java 8 버킷 레벨 synchronized + CAS, computeIfAbsent 내부
- CopyOnWriteArrayList — 쓰기마다 배열 복사, 읽기가 락 없이 안전한 이유, 적합/부적합 사용 사례
- BlockingQueue 내부 구현 — ArrayBlockingQueue(단일 락) vs LinkedBlockingQueue(헤드/테일 분리 락) 비교
- ConcurrentSkipListMap — 스킵 리스트로 Lock-Free 정렬 맵을 구현하는 방법, TreeMap과 성능 비교
- Exchanger와 Phaser — 두 스레드 간 데이터 교환 원리, Phaser로 여러 단계 동기화를 구현하는 방식

### Chapter 6: Virtual Thread (Project Loom) — 패러다임의 전환 (6개 문서)
- 왜 Virtual Thread인가 — OS 스레드의 한계(1MB 스택, 컨텍스트 스위칭 비용), I/O 바운드 작업에서 스레드 풀이 낭비되는 구조
- Continuation과 스케줄링 — JVM Continuation 객체가 실행 상태를 저장하는 방식, ForkJoinPool 캐리어 스레드 위에서 스케줄링
- 블로킹 시 동작 — Virtual Thread가 블로킹 I/O(소켓 read)를 만났을 때 캐리어 스레드에서 언마운트되는 과정
- Pinning 문제 — synchronized 블록 안에서 블로킹 시 캐리어 스레드가 묶이는 원인, ReentrantLock으로 해결
- ThreadLocal과 ScopedValue — Virtual Thread에서 ThreadLocal이 메모리 누수를 일으키는 이유, Java 21 ScopedValue 설계 원리
- Spring Boot + Virtual Thread — server.threads.virtual.enabled=true 설정, @Async와 Virtual Thread 조합, JPA/DataSource와의 호환성

### Chapter 7: 실전 패턴과 장애 분석 (5개 문서)
- 데드락 발생과 분석 — 데드락 4가지 조건, Thread Dump로 데드락 찾기, jstack/jcmd 활용
- 라이브락과 기아(Starvation) — 스핀 락에서 라이브락 발생 조건, 기아 문제와 공정한 락의 트레이드오프
- 성능 병목 패턴 — Lock Contention 측정(JFR, async-profiler), 과도한 synchronized 범위의 처리량 저하
- 비동기 프로그래밍 비교 — CompletableFuture(스레드 풀) vs WebFlux(이벤트 루프) vs Virtual Thread 실제 차이
- Spring 동시성 이슈 — @Transactional과 멀티스레드 주의사항, @Async 스레드 풀 설정, 싱글톤 Bean의 상태 공유 문제

---

각 챕터는 **5~7개 문서**로 구성해줘. 총 **40개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 실무에서 중요한가
## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)
## ✨ 올바른 접근 (After — 원리를 알고 난 설계/구현)
## 🔬 내부 동작 원리 (JVM/JIT/CPU 수준 분석)
## 💻 실전 실험 (재현 가능한 코드, JVM 플래그, 벤치마크)
## 📊 성능/비용 비교 (벤치마크 수치, OS 스레드 vs Virtual Thread 등)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 모든 동기화 메커니즘은 "JVM이 내부에서 어떤 CPU 명령어를 사용하는가"까지 연결
2. JVM 플래그(`-XX:+PrintBiasedLockingStatistics`, `-XX:+UnlockDiagnosticVMOptions`)로 내부 상태 확인
3. JMH(Java Microbenchmark Harness)로 성능 비교 실험 코드 포함
4. Thread Dump, JFR(Java Flight Recorder), async-profiler를 활용한 실전 진단
5. Spring 연결 — @Transactional, @Async, Virtual Thread와 JPA의 실제 동작

**실험 환경**:
```yaml
# docker-compose.yml
services:
  java-lab:
    image: eclipse-temurin:21-jdk
    volumes:
      - ./src:/workspace/src
      - ./benchmarks:/workspace/benchmarks
    working_dir: /workspace
    command: /bin/bash
    stdin_open: true
    tty: true
    environment:
      - JAVA_OPTS=-XX:+UseZGC -Xmx512m

  jmh-runner:
    image: maven:3.9-eclipse-temurin-21
    volumes:
      - ./benchmarks:/workspace
    working_dir: /workspace
    command: mvn clean package && java -jar target/benchmarks.jar
```

```bash
# 실험용 공통 명령어 세트

# JVM 플래그로 내부 동작 확인
java -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintBiasedLockingStatistics \    # 바이어스 락 통계
     -XX:+LogCompilation \                  # JIT 컴파일 로그
     -XX:+PrintAssembly \                   # 어셈블리 코드 출력
     MyApp

# Virtual Thread 실험
java -Djdk.tracePinnedThreads=full \        # Pinning 추적
     MyVirtualThreadApp

# Thread Dump (데드락 분석)
jstack <pid>
jcmd <pid> Thread.print

# Java Flight Recorder (성능 프로파일링)
jcmd <pid> JFR.start duration=60s filename=recording.jfr
jfr print --events jdk.VirtualThreadPinned recording.jfr

# JMH 벤치마크 실행
java -jar benchmarks.jar \
  -f 3 \                    # 3 fork
  -wi 5 \                   # 5 warmup iterations
  -i 10 \                   # 10 measurement iterations
  -t 4 \                    # 4 threads
  ".*LockBenchmark.*"
```

```java
// 기본 JMH 벤치마크 구조 (모든 챕터에서 활용)
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@State(Scope.Benchmark)
@Fork(3)
@Warmup(iterations = 5)
@Measurement(iterations = 10)
public class ConcurrencyBenchmark {

    // 실험 대상 코드
    @Benchmark
    public void synchronizedMethod() { /* ... */ }

    @Benchmark
    public void reentrantLock() { /* ... */ }

    @Benchmark
    public void stampedLockOptimistic() { /* ... */ }
}
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**:
   - mysql-deep-dive/jvm-deep-dive README 스타일 참고
   - "synchronized를 쓰는 것을 넘어 락의 내부 구현과 Virtual Thread를 이해한다"는 차별화 강조
   - jvm-deep-dive, linux-deep-dive, Spring 레포와의 연결 지점 명시
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료 섹션

README 작성 시 다음을 Reference로 포함해줘:
- Java Concurrency in Practice (Brian Goetz et al.) — 바이블
- The Art of Multiprocessor Programming (Herlihy & Shavit) — Lock-Free 알고리즘
- Inside the Java Virtual Machine (Bill Venners) — JVM 내부 구조
- JEP 425 (Virtual Threads) — https://openjdk.org/jeps/425
- JEP 444 (Virtual Threads in Java 21) — https://openjdk.org/jeps/444
- Aleksey Shipilëv's Blog — https://shipilev.net/ (JMM, JIT, 성능)
- Nitsan Wakart's Blog — https://psy-lob-saw.blogspot.com/ (Lock-Free, 성능)

---

## 💡 핵심 분석 대상

```
synchronized 내부 락 확장 과정:

객체 생성 직후: Biased Lock 상태
  Object Header Mark Word:
  [thread_id | epoch | 01] — 특정 스레드가 "소유권 주장"
  → 같은 스레드가 다시 진입: Mark Word 비교만, CAS 없음 (가장 빠름)

다른 스레드 진입 시도: Thin Lock(Lightweight Lock)으로 확장
  CAS로 Mark Word를 Lock Record 주소로 교체
  → CAS 성공: 락 획득
  → CAS 실패: 경쟁 발생 → Fat Lock으로 확장

경쟁 심화: Fat Lock(Heavyweight Lock, Monitor)
  OS Mutex 사용 → 대기 스레드 Sleep
  → 컨텍스트 스위칭 발생 (비쌈)

JIT 최적화 (Lock Elision):
  escape analysis로 객체가 스택을 벗어나지 않으면
  synchronized 제거 가능
  → 컴파일 후 락 없는 코드가 됨

Virtual Thread 동작 원리:

Platform Thread(OS Thread)가 Carrier 역할:
  ForkJoinPool에 8개 캐리어 스레드 (CPU 코어 수)

Virtual Thread 100만 개 생성 가능:
  각 Virtual Thread = JVM Heap에 Continuation 객체
  (스택 정보를 힙에 저장 → 수 KB)
  vs Platform Thread: 1MB 스택 + OS 자원

블로킹 I/O 발생 시:
  VirtualThread가 Socket.read() 호출
    → NIO non-blocking으로 재작성됨 (JDK 내부)
    → 데이터 없으면: Continuation 저장 → Unmount
    → Carrier Thread는 다른 VirtualThread 실행
    → 데이터 도착 → Mount 다시 Carrier에
    → Continuation 복원 → 실행 재개

synchronized 안에서 블로킹:
  VirtualThread가 synchronized 블록 내 블로킹 호출
    → JVM은 이 경우 Unmount 불가 (JVM 제한)
    → Carrier Thread도 같이 블로킹 = "Pinning"
    → JFR jdk.VirtualThreadPinned 이벤트로 감지
  해결: synchronized → ReentrantLock으로 교체

JMM happens-before 실제 예시:
  // 스레드 A
  data = 42;           // write
  ready = true;        // volatile write

  // 스레드 B
  if (ready) {         // volatile read
    // data를 안전하게 읽을 수 있는가?
    print(data);       // 42가 보장되는가?
  }

  volatile write → volatile read happens-before 관계 성립
  → volatile write 이전의 모든 쓰기 (data = 42)가
    volatile read 이후 모든 읽기 (print(data))에 보임
  → 보장됨!

  volatile 없이:
  data = 42가 CPU 캐시에만 있고 주 메모리에 미반영
  → 스레드 B는 data의 이전 값(0)을 볼 수 있음
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (40개 목표)
- JMH 벤치마크 포함 실험 환경 (Java 21)
- jvm-deep-dive, linux-deep-dive, Spring 레포와 연결되는 지점 명시 (어느 문서에서 어느 레포와 연결하는지)

**준비됐으면 1단계 구조 설계부터 시작해줘!**
