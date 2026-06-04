# Kotlin Deep Dive 레포지토리 제작 프롬프트

나는 "Kotlin Deep Dive" 레포지토리를 만들려고 해.
코루틴이 *어떻게 스레드 없이 일시정지·재개*하는지(CPS·상태머신), Flow의 백프레셔, null 안전·inline·고차 함수가 바이트코드로 어떻게 번역되는지를 완전히 파헤치는 레포야.
"코루틴을 쓰는 것"과 "suspend가 컴파일러에 의해 상태머신이 되고 Dispatcher가 어떻게 스레드를 배정하는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "Kotlin으로 코루틴을 쓰는 것과, suspend가 어떻게 상태머신으로 컴파일되고 구조적 동시성이 무엇을 보장하는지 아는 것은 다르다"

**핵심 차별화**:
1. 코루틴 = 상태머신 — suspend 함수가 CPS 변환으로 일시정지 가능한 상태머신이 되는 원리
2. 구조적 동시성 — 스코프·Job 계층이 누수·취소를 자동 관리하는 메커니즘
3. Flow와 백프레셔 — 콜드 스트림·suspend 기반 흐름 제어, Reactive와의 차이
4. 바이트코드의 진실 — inline·data class·null 안전이 JVM 바이트코드로 번역되는 방식

**타겟 독자**:
- 코루틴을 쓰지만 suspend가 어떻게 멈추고 재개되는지 모르는 개발자
- `Dispatchers.IO`를 쓰지만 스레드 배정 원리를 모르는 개발자
- Flow를 쓰지만 콜드/백프레셔를 정확히 모르는 개발자
- inline·reified를 외워 쓰는 개발자
- `jvm-deep-dive`·`java-concurrency`를 Kotlin으로 확장하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `jvm-deep-dive`(바이트코드·스레드), `java-concurrency-deep-dive`(스레드·메모리 모델).
**🤝 시너지**: `android-runtime-deep-dive`(Kotlin→DEX), `jetpack-compose-internals-deep-dive`(Compose는 코루틴/Snapshot 기반), `go-deep-dive`(고루틴 대조).
**🧬 수렴**: `concurrency-models-compared`(코루틴 ↔ 고루틴·async/await·Virtual Thread).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: Kotlin 컴파일과 기본 (5개 문서)
- Kotlin→바이트코드 — 컴파일 파이프라인, JVM 타깃, IntelliJ 바이트코드 뷰어
- null 안전 — nullable 타입이 바이트코드(어노테이션·체크)로, 플랫폼 타입
- data class — equals/hashCode/copy/componentN 생성 결과
- 프로퍼티 — getter/setter·backing field, lateinit·lazy의 구현
- 확장 함수 — 정적 메서드로 컴파일, 디스패치(정적), 리시버

### Chapter 2: 코루틴 내부 — 상태머신 (7개 문서)
- 코루틴이란 — 스레드가 아닌 일시정지 가능한 계산, 경량성의 의미
- suspend 함수 — Continuation, 호출 규약(숨은 Continuation 인자)
- CPS 변환 — 컴파일러가 suspend를 continuation-passing으로 바꾸는 원리
- 상태머신 생성 — await 지점마다 상태 라벨, 재개 시 점프(decompile로 확인)
- Continuation·resumeWith — 일시정지 후 어떻게 다시 시작하나
- 코루틴 빌더 — launch/async/runBlocking, 무엇을 반환·언제 시작
- 코루틴 vs 스레드 — 수만 코루틴 가능한 이유, 스택 없는 일시정지

### Chapter 3: 구조적 동시성과 Dispatcher (6개 문서)
- 구조적 동시성 — 스코프가 자식 코루틴 생명을 묶기, 누수 방지
- CoroutineScope·Job — Job 계층·부모-자식, 완료·취소 전파
- 취소 — 협력적 취소, isActive·suspend 지점에서 취소 확인, NonCancellable
- 예외 처리 — 예외 전파, SupervisorJob, CoroutineExceptionHandler
- Dispatcher — Default/IO/Main, 스레드 풀 배정, withContext 전환
- 디스패처 내부 — IO의 탄력적 풀, Main의 Looper 연동(android)

### Chapter 4: Flow와 백프레셔 (6개 문서)
- Flow 개념 — 콜드 스트림, suspend 기반, 수집(collect) 시 시작
- Flow 빌더·연산자 — emit·map·filter, 중간 연산의 지연
- 백프레셔 — suspend로 생산-소비 속도 조율, buffer·conflate
- StateFlow·SharedFlow — 핫 스트림, 상태 보유·다중 구독(android 상태 관리)
- 컨텍스트·flowOn — 업스트림 디스패처 전환, 컨텍스트 보존 규칙
- Flow vs Reactive(RxJava) — 모델 비교, 코루틴 통합 이점

### Chapter 5: 타입 시스템과 null 안전 (6개 문서)
- 타입 시스템 — 통합 타입(Any/Unit/Nothing), 스마트 캐스트
- null 안전 심화 — ?.·?:·!!·let, 플랫폼 타입의 위험
- 제네릭·변성 — in/out(선언처 변성), 스타 프로젝션, reified
- sealed·enum — sealed 계층, when 망라성, 대수적 데이터 타입
- 위임(Delegation) — by 위임, observable·vetoable, 위임 프로퍼티 구현
- 인터페이스·기본 구현 — 바이트코드(DefaultImpls), Java 상호운용

### Chapter 6: inline과 고차 함수 (6개 문서)
- 고차 함수 비용 — 람다가 객체(Function)로, 호출 오버헤드
- inline — 람다를 호출부에 인라인, 객체 할당 제거, 바이트코드 확인
- noinline·crossinline — 인라인 제어, 비국소 반환
- reified — inline 덕에 런타임 타입 유지, 제네릭 타입 검사
- 람다·SAM — SAM 변환, Java 함수형 인터페이스 상호운용
- 측정 — inline 전후 할당·바이트코드 비교

### Chapter 7: DSL·메타·실전 (6개 문서)
- 수신 객체 람다 — `T.() -> Unit`, DSL의 토대(빌더)
- 연산자 오버로딩·중위 — 가독성 있는 API 설계
- 컴파일러 플러그인 — kapt vs KSP, 어노테이션 처리(Compose·Hilt 기반)
- 멀티플랫폼 미리보기 — expect/actual 개념(KMP 레포 연결)
- Java 상호운용 — 양방향 호출, 함정(null·기본 인자·companion)
- 종합 — 코루틴 + Flow + DSL로 작은 비동기 라이브러리 구현

→ **총 42개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 생성 바이트코드·상태머신(decompile), `💻 실전 실험`은 바이트코드 뷰어·코루틴 디버그. `📊`는 inline/코루틴 비용·할당 비교.

## 🎨 스타일 가이드

1. **바이트코드로 까본다** — suspend 상태머신·inline 결과를 디컴파일로 *직접 본다*
2. **코루틴=상태머신 일관** — 모든 코루틴 동작을 상태머신·Continuation으로 환원
3. **Java/Go와 대조** — 코루틴 vs 스레드 vs 고루틴
4. **jvm/concurrency 레포로 착지** — 바이트코드·스레드는 그 레포로
5. CPS 변환·Job 계층·Flow는 다이어그램으로

## 🔬 검증 환경

> docker 가능(kotlinc). 핵심은 **바이트코드 뷰어 + 코루틴 디버그**.

```dockerfile
FROM eclipse-temurin:21-jdk
RUN curl -s https://get.sdkman.io | bash && \
    bash -c "source ~/.sdkman/bin/sdkman-init.sh && sdk install kotlin"
```

```bash
# 바이트코드/디컴파일로 상태머신 보기
kotlinc Coroutine.kt -include-runtime -d out.jar
# IntelliJ: Tools > Kotlin > Show Kotlin Bytecode > Decompile
#   → suspend 함수가 switch(label) 상태머신으로 디컴파일됨 (핵심!)

# inline 확인
# inline 함수 호출부 디컴파일 → 람다 객체 없이 본문이 펼쳐짐 확인

# 코루틴 디버그
# -Dkotlinx.coroutines.debug 로 코루틴 이름·스택
# IntelliJ Coroutine Debugger로 일시정지 지점·디스패처 관찰

# 수만 코루틴 vs 수만 스레드 메모리 비교 실험
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🤖 Android(언어) 톤, "코루틴을 쓴다 vs 상태머신이 된다" 포지셔닝, `🔗 레포 연결`(jvm·java-concurrency·concurrency-models-compared)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- Kotlin 공식 문서·코루틴 가이드 — https://kotlinlang.org/docs/coroutines-guide.html
- "Deep Dive into Coroutines" (Roman Elizarov 발표들)
- *Kotlin Coroutines: Deep Dive* (Marcin Moskała)
- KEEP(코루틴 설계 문서) — https://github.com/Kotlin/KEEP
- kotlinx.coroutines 소스

## 💡 핵심 분석 대상

```
suspend → 상태머신 (CPS 변환):
  suspend fun load() {
    val a = fetchA()   // suspend
    val b = fetchB(a)  // suspend
    show(a, b)
  }
  ↓ 컴파일러 생성(개념)
  fun load(cont: Continuation) {
    when(cont.label) {
      0 -> { cont.label=1; fetchA(cont); return }  // 일시정지
      1 -> { val a=cont.result; cont.label=2; fetchB(a,cont); return }
      2 -> { show(a, cont.result) }
    }
  }
  → 스레드를 안 막고 라벨로 재개 (decompile로 확인 가능)

구조적 동시성:
  scope.launch {        // 부모 Job
    launch { ... }       // 자식 1
    launch { ... }       // 자식 2
  }
  → scope 취소 = 모든 자식 취소, 자식 실패 = 형제 취소(누수 0)

Dispatcher:
  withContext(Dispatchers.IO) { 블로킹 IO }   // IO 풀
  withContext(Dispatchers.Main) { UI 갱신 }   // 메인 Looper
  → 같은 코루틴이 스레드를 옮겨 다님

inline (할당 제거):
  list.forEach { ... }   // inline → 람다 객체 없음, 루프로 펼쳐짐
  비inline 고차함수       → Function 객체 생성(할당)
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 42개 확인 + 바이트코드뷰어/코루틴디버그 검증 환경 + jvm/java-concurrency/concurrency-models-compared 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
