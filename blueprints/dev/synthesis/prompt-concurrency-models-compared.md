# Concurrency Models Compared 레포지토리 제작 프롬프트

나는 "Concurrency Models Compared" 레포지토리를 만들려고 해.
이건 개별 기술 레포가 아니라 **횡단 비교(Synthesis)** 레포야. 여러 플랫폼이 "여러 일을 동시에"라는 같은 문제를 *서로 다르게* 푼 방식 — JVM Virtual Thread, JS Event Loop, Kotlin Coroutine, Swift async/Actor, Go 고루틴, Rust async를 한자리에 놓고 비교한다.
"한 언어의 동시성을 아는 것"과 "모든 동시성 모델이 결국 같은 트레이드오프 공간의 다른 점이라는 걸 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "한 언어의 동시성을 잘 쓰는 것과, 모든 모델이 '블로킹을 어떻게 피하고 상태를 어떻게 보호하는가'라는 같은 질문에 다르게 답한 것임을 아는 것은 다르다"

**핵심 차별화**:
1. 공통 질문으로 환원 — 모든 모델 = "스레드를 어떻게 효율적으로 쓰나 + 공유 상태를 어떻게 안전하게 하나"
2. 같은 문제, 다른 답 — 같은 동시성 과제를 6개 모델로 구현해 *나란히* 비교
3. 트레이드오프 공간 — 색깔 함수·런타임 유무·안전 보장 위치(컴파일/런타임)의 축
4. 수렴과 발산 — 왜 모두 협력적 스케줄링으로 수렴했고, 안전성에선 갈렸나

**타겟 독자**:
- 한 언어 동시성은 알지만 다른 모델과 비교 못하는 개발자
- "코루틴 = 고루틴?"을 정확히 구분 못하는 개발자
- 새 언어의 동시성을 빠르게 이해하고 싶은 개발자
- 동시성 기술 선택의 트레이드오프를 알고 싶은 개발자
- 개별 언어 레포(아래 선행)를 마치고 큰 그림을 원하는 개발자

## 🔗 레포 연결

**⬆️ 선행(이 비교의 입력 — 먼저 보면 좋음)**:
`java-concurrency-deep-dive`(Virtual Thread·락), `kotlin-deep-dive`(코루틴·상태머신), `go-deep-dive`(고루틴·채널·GMP), `rust-deep-dive`(async·Send/Sync), `swift-concurrency-deep-dive`(async/Actor), `event-loop-async-deep-dive`(JS 이벤트 루프).
**🤝 시너지**: `computer-architecture-deep-dive`(메모리 모델·원자연산), `linux-for-backend-deep-dive`(스레드·스케줄링).
**🧬 본질**: 이 레포 자체가 수렴점 — 위 6개를 하나의 비교 프레임으로 묶는다.

---

## 🎯 1단계: 전체 구조 설계

> 이 레포는 "기술별"이 아니라 **"질문별"**로 챕터를 구성한다. 각 챕터에서 6개 모델을 나란히 비교.

### Chapter 1: 동시성의 근본 문제 (5개 문서)
- 두 가지 질문 — ① 스레드를 어떻게 효율적으로 쓰나 ② 공유 상태를 어떻게 안전하게 하나
- 블로킹의 비용 — OS 스레드 블로킹이 비싼 이유, C10K 문제(linux 연결)
- 동시성 vs 병렬성 — 개념 구분, 단일 코어에서도 동시성이 의미 있는 이유
- 공유 상태와 레이스 — 데이터 레이스의 본질(computer-architecture 메모리 모델 연결)
- 비교 프레임 설정 — 이 레포가 쓸 평가 축(효율·안전·표현력·복잡도)

### Chapter 2: 스레드 효율 — 경량 실행 단위 (6개 문서)
- OS 스레드 — 비싼 이유(스택·컨텍스트 스위치·커널), 수만 개 불가
- 협력적 스케줄링 — 모든 모델의 공통 답: 양보 지점에서 멈추고 재개
- Virtual Thread(JVM) — 캐리어 스레드 위 다중화, 블로킹 코드 그대로
- 고루틴(Go) — M:N GMP 스케줄러, 2KB 스택, 워크 스틸링
- Coroutine(Kotlin)·async(Rust/Swift) — 상태머신 변환, suspend 지점
- 나란히 비교 — 수만 개 동시 실행을 6모델로, 메모리·생성 비용 측정

### Chapter 3: 일시정지의 메커니즘 (6개 문서)
- 두 방식 — 스택풀(Virtual Thread·고루틴) vs 스택리스(상태머신: Rust/JS/Kotlin/Swift)
- 상태머신 변환 — async/suspend가 컴파일러에 의해 enum 상태머신으로(kotlin·rust·swift 동형)
- 스택풀 코루틴 — 고루틴·Virtual Thread가 실제 스택을 가진 채 양보
- 색깔 함수 문제 — async가 전염되는 언어(JS/Rust) vs 안 되는 언어(Go/Java VT)
- Continuation — 일시정지/재개의 저수준, 공통 추상
- 비교 — 같은 비동기 로직의 코드 형태를 6모델로(색깔 유무가 만드는 차이)

### Chapter 4: 상태 보호 — 안전성 (7개 문서)
- 세 가지 전략 — ① 공유+락 ② 격리(Actor) ③ 소유권+타입 ④ 공유 대신 통신(채널)
- 공유 + 락(JVM) — synchronized·락, 런타임 보장, 실수 가능(java-concurrency 연결)
- 격리(Swift Actor) — 가변 상태 직렬화, 컴파일 타임 격리 검사(swift-concurrency 연결)
- 소유권(Rust) — Send/Sync로 컴파일 타임 레이스 제거, 런타임 비용 0(rust 연결)
- 통신(Go) — "공유하지 말고 통신하라", 채널·CSP(go 연결)
- 안전성 위치 비교 — 컴파일 타임(Rust/Swift) vs 런타임(JVM) vs 관례(Go)
- 같은 레이스를 6모델로 — 어떤 모델이 *컴파일 때* 막고 어떤 건 *런타임*에 터지나

### Chapter 5: 구조와 취소 (5개 문서)
- 구조적 동시성 — 작업 트리·생명주기 묶기(Kotlin·Swift·Java 공통 수렴)
- 취소 전파 — 부모 취소→자식 취소, 누수 방지(kotlin·swift 동형)
- 비구조적 위험 — 떠도는 태스크·고루틴 누수
- 에러 전파 — 동시 작업의 예외 처리 비교
- 비교 — 같은 "여러 작업 + 하나 실패 시 전체 취소"를 6모델로

### Chapter 6: 같은 문제, 6가지 구현 (6개 문서)
- 과제 정의 — 동일한 동시성 과제(예: 동시 API 호출 + 집계 + 타임아웃)
- 구현 ① JVM Virtual Thread / ② Kotlin Coroutine
- 구현 ③ Go 고루틴+채널 / ④ Rust async(Tokio)
- 구현 ⑤ Swift async/Actor / ⑥ JS async/Promise
- 코드 비교 — 6개 구현을 나란히, 가독성·안전성·표현력
- 측정 비교 — 처리량·메모리·지연을 같은 부하로

### Chapter 7: 트레이드오프 종합 (5개 문서)
- 평가 축 종합 — 효율·안전성 위치·색깔·런타임·복잡도 표
- 왜 수렴했나 — 모두 협력적 경량 스케줄링으로 간 이유
- 왜 발산했나 — 안전성 철학(타입 vs 런타임 vs 관례)의 분기
- 선택 가이드 — 워크로드·언어·팀에 따른 모델 이해
- 종합 — 동시성 모델 지형도, 새 언어를 만나도 이 프레임으로 빠르게 파악

→ **총 40개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). 단, 이 레포는 **비교가 핵심**이라 `🔬 내부 동작 원리`·`📊`에서 *항상 여러 모델을 나란히* 놓는다. `💻 실전 실험`은 같은 과제를 여러 언어로 구현.

## 🎨 스타일 가이드

1. **항상 나란히** — 한 모델만 설명하지 말고 최소 2~3개를 같은 표/코드로 비교
2. **질문으로 환원** — "이건 효율 답인가 안전성 답인가"로 모든 기법 분류
3. **선행 레포로 깊이 위임** — 각 모델의 *내부*는 해당 레포로, 여기선 *비교*에 집중
4. **동형성 강조** — Kotlin·Rust·Swift 상태머신이 같은 것임을 보여준다
5. 트레이드오프 축·6모델 비교는 표·다이어그램으로

## 🔬 검증 환경

> 다언어 비교라 **여러 런타임을 한 번에**. docker-compose로 polyglot 환경.

```dockerfile
# Dockerfile — 6모델 비교용 polyglot
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y \
    openjdk-21-jdk \      # JVM Virtual Thread
    golang-go \           # 고루틴
    nodejs npm \          # JS 이벤트 루프
    curl
RUN curl https://sh.rustup.rs -sSf | sh -s -- -y          # Rust async
# Kotlin: sdkman, Swift: swift:6 (별도 또는 멀티스테이지)
```

```bash
# 핵심: 같은 과제를 6언어로 구현해 나란히 측정
# 과제 예: "1만 개 동시 작업, 각 100ms 대기 후 집계, 5초 타임아웃"

# 효율 비교: 1만 동시 작업의 메모리·시간
#   Go:   goroutine 1만개      → 메모리 측정
#   JVM:  virtual thread 1만개 → 메모리 측정
#   비교: OS 스레드 1만개(불가/거대) 대조

# 안전성 비교: 공유 카운터에 동시 증가
#   Rust:  컴파일 에러(Send/Sync 위반) ← 컴파일 타임 방어
#   Swift: actor면 안전, 아니면 컴파일 경고
#   Go:    -race로 런타임 검출
#   JVM:   런타임 레이스(검출 도구 필요)

# 색깔 함수: async 전염 관찰
#   Rust/JS: async fn → 호출자도 async
#   Go/JVM VT: 일반 함수처럼 블로킹 코드 작성
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `comparisons/`(언어별 동일 과제 구현)
2. **README.md**: 🧬 Synthesis 톤, "한 언어 동시성 vs 모든 모델의 트레이드오프 공간" 포지셔닝, 선행 6개 레포를 `🔗 레포 연결`에 입력으로 명시
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어, 항상 다중 모델 비교

## 📚 참고 자료

- "Notes on structured concurrency" (Nathaniel Smith)
- "What color is your function?" (Bob Nystrom)
- *Java Concurrency in Practice* / Go·Rust·Swift 동시성 문서
- "Concurrency is not Parallelism" (Rob Pike)
- 각 선행 레포의 참고자료(이 레포는 그 종합)

## 💡 핵심 분석 대상

```
두 가지 근본 질문 (모든 모델이 답함):
  ① 효율: 스레드를 어떻게 싸게? → 경량 실행 단위 + 협력적 스케줄링
  ② 안전: 공유 상태를 어떻게? → 락 / 격리 / 소유권 / 통신

효율의 답 (모두 수렴):
  OS 스레드(비쌈) → 경량 단위로 다중화
  Go 고루틴 · JVM Virtual Thread · Kotlin/Rust/Swift async · JS 이벤트루프
  → 전부 "양보 지점에서 멈추고 스레드 재사용"

일시정지 방식 (갈림):
  스택풀: Go 고루틴, JVM Virtual Thread (실제 스택 보유)
  스택리스: JS/Rust/Kotlin/Swift (상태머신으로 변환, 색깔 함수)

안전성의 답 (발산):
  락(JVM)      : 런타임 보장, 실수 가능
  Actor(Swift) : 격리, 컴파일 타임 검사
  소유권(Rust) : 타입 시스템, 컴파일 타임, 런타임 0
  채널(Go)     : 공유 대신 통신(CSP)
  → 같은 레이스가 Rust에선 컴파일 에러, JVM에선 런타임 버그

트레이드오프 공간:
                컴파일타임 안전 ◄──────► 런타임/관례
  Rust ●                                        ● JVM
  Swift ●                              ● Go
                  색깔 있음 ◄──────► 색깔 없음
  Rust/JS ●                          ● Go/JVM VT
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 40개 확인 + polyglot 검증 환경 + 선행 6개 레포(java-concurrency·kotlin·go·rust·swift-concurrency·event-loop)를 입력으로 하는 연결 명시. **항상 여러 모델을 나란히 비교**하는 게 이 레포의 정체성.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
