# Rust Deep Dive 레포지토리 제작 프롬프트

나는 "Rust Deep Dive" 레포지토리를 만들려고 해.
소유권·빌림 검사기·라이프타임이 *컴파일 타임에* 메모리 안전과 데이터 레이스 부재를 어떻게 GC 없이 증명하는지를 완전히 파헤치는 레포야.
"Rust 문법을 외우는 것"과 "빌림 검사기가 왜 이 코드를 거부하는지, 그 거부가 무엇을 보장하는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "Rust로 컴파일을 통과시키는 것과, 소유권 모델이 무엇을 증명하는지 아는 것은 다르다"

**핵심 차별화**:
1. 소유권은 자원 관리 이론이다 — move/borrow/lifetime이 GC도 수동 free도 없이 안전을 증명하는 원리
2. 빌림 검사기는 증명기다 — aliasing XOR mutability 규칙이 데이터 레이스를 *컴파일 타임에* 제거
3. Send/Sync는 타입으로 동시성을 보장 — 런타임 없이 스레드 안전성을 타입 시스템에 인코딩
4. Zero-cost 추상화 — 트레이트·이터레이터·async가 어셈블리 레벨에서 비용이 사라지는 증명

**타겟 독자**:
- Rust를 쓰지만 빌림 검사기와 "싸우기만" 하고 *왜* 거부되는지 모르는 개발자
- GC 언어(Java/Go)만 써서 "메모리 안전 = GC"라고 생각하는 개발자
- `Arc<Mutex<T>>`를 복붙하지만 Send/Sync가 뭘 보장하는지 모르는 개발자
- async/await를 쓰지만 Future가 어떻게 상태머신이 되는지 모르는 개발자
- C++의 수동 메모리 관리 위험을 Rust가 어떻게 없앴는지 궁금한 개발자

## 🔗 레포 연결

**⬆️ 선행**: `computer-architecture-deep-dive`(스택/힙·메모리 레이아웃·캐시가 소유권 비용 이해의 토대).
**🤝 시너지**: `go-deep-dive`(GC 있는 동시성과의 대조), `java-concurrency-deep-dive`(메모리 모델·락 대조), `compiler-deep-dive`(빌림 검사기는 타입 검사의 일종).
**🧬 수렴**: `memory-management-compared`(소유권 = "GC 없는 답"의 대표), `concurrency-models-compared`(Send/Sync 격리 모델).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 소유권·이동·빌림 — Rust의 심장 (6개 문서)
- 소유권 모델 — 값은 단 하나의 소유자, 스코프 종료 시 drop, RAII의 일반화
- 이동(Move) 의미론 — 대입/전달이 복사가 아니라 이동, move 후 원본 무효화, Copy 타입의 예외
- 빌림(Borrow) — 불변 참조(&)와 가변 참조(&mut), "aliasing XOR mutability" 핵심 규칙
- 빌림 검사기의 거부 이유 — 동시 가변 참조·dangling이 막히는 시나리오를 코드로 재현
- Drop과 소멸 순서 — Drop 트레이트, 결정적 해제, 순환 참조가 누수를 만드는 경우
- 소유권 vs GC vs 수동 — 같은 문제를 세 언어가 푸는 방식 비교(누수·UAF·이중해제 제거)

### Chapter 2: 라이프타임 (5개 문서)
- 라이프타임이 왜 필요한가 — 참조가 가리키는 데이터보다 오래 살면 안 됨, 컴파일러의 검증
- 라이프타임 표기 — `'a`의 의미, 함수 시그니처의 라이프타임, 생략 규칙(elision)
- 구조체와 라이프타임 — 참조를 담는 구조체, 라이프타임 파라미터 전파
- 빌림 검사의 발전 — NLL(Non-Lexical Lifetimes), Polonius, 왜 예전엔 거부되던 코드가 통과되나
- 어려운 케이스 — self-referential struct가 어려운 이유, Pin의 등장 배경

### Chapter 3: 타입 시스템과 트레이트 (6개 문서)
- 트레이트 — 인터페이스 + 더, 정적 디스패치(제네릭) vs 동적 디스패치(dyn), vtable
- 제네릭과 단형화(Monomorphization) — 컴파일 타임 특수화, 코드 팽창 vs zero-cost
- 연관 타입 vs 제네릭 — 언제 무엇을, Iterator의 Item이 연관 타입인 이유
- 트레이트 객체와 객체 안전성 — `dyn Trait`, 객체 안전 규칙, fat pointer
- 트레이트 일관성(Coherence) — orphan rule, 왜 이 제약이 있는가
- 표준 트레이트 — Deref·From/Into·AsRef, 트레이트로 짜는 관용 API

### Chapter 4: 스마트 포인터와 내부 가변성 (6개 문서)
- Box — 힙 할당, 재귀 타입, 트레이트 객체 보관
- Rc/Arc — 참조 카운팅 공유 소유권, Arc의 원자적 카운터, 순환 참조와 Weak
- Cell/RefCell — 내부 가변성, "빌림 규칙을 런타임으로 미루기", RefCell 패닉
- Mutex/RwLock — 락이 데이터를 감싸는 Rust식 설계, 포이즌(poisoning)
- 내부 가변성의 안전성 — UnsafeCell이 토대, 왜 RefCell은 Sync가 아닌가
- 스마트 포인터 조합 — `Rc<RefCell<T>>`·`Arc<Mutex<T>>`가 필요한 이유와 비용

### Chapter 5: 동시성 — 런타임 없는 안전성 (6개 문서)
- 스레드와 데이터 공유 — thread::spawn, move 클로저, 소유권으로 데이터 레이스 방지
- Send와 Sync — 타입이 스레드 경계를 넘을/공유될 안전성, 자동 파생과 부정
- "두려움 없는 동시성"의 실체 — 컴파일러가 거부하는 레이스 시나리오를 코드로
- 메시지 패싱 — 채널(mpsc), 소유권 이동으로 공유 없는 동시성
- 공유 상태 — Atomic 타입, Ordering(Relaxed/Acquire/Release), 메모리 모델
- 동시성 비교 — Rust(타입 보장) vs Go(고루틴+채널) vs Java(런타임 락) 대조

### Chapter 6: async/await — 런타임 없는 비동기 (6개 문서)
- Future 트레이트 — poll 기반, "언어는 문법만, 실행은 런타임이" 분리 설계
- async가 상태머신이 되는 과정 — async fn → 컴파일러 생성 enum 상태머신, await 지점
- 실행기(Executor)와 런타임 — Rust는 왜 런타임을 기본 제공하지 않나, Tokio의 역할
- Pin과 자기참조 — async 상태머신이 자기참조라 Pin이 필요한 이유
- Waker와 깨우기 — poll이 Pending일 때 어떻게 다시 깨워지나
- async 비교 — JS의 Promise·Go의 고루틴과 모델 대조, 색깔 함수 문제

### Chapter 7: unsafe·FFI·성능 (7개 문서)
- unsafe의 의미 — "컴파일러가 검증 못 하는 불변식을 내가 보증", 무엇이 풀리고 무엇은 안 풀리나
- unsafe의 5가지 능력 — raw pointer 역참조, FFI 호출, static mut 등
- 안전 추상화 만들기 — unsafe 내부를 안전 API로 감싸기(Vec의 내부)
- FFI — C와의 상호운용, ABI, repr(C), 메모리 소유권 경계
- UB와 miri — 미정의 동작 종류, miri로 unsafe 코드의 UB 검출
- 성능과 어셈블리 — zero-cost 추상화를 godbolt/cargo-asm으로 증명(이터레이터 = 수동 루프)
- 최적화 함정 — 불필요한 clone·Box·동적 디스패치 비용 측정

→ **총 42개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 빌림 검사기 동작·생성 어셈블리, `💻 실전 실험`은 "컴파일러가 거부하는 코드"를 직접 작성해 에러를 읽고 godbolt/miri로 검증. `📊`는 추상화의 비용을 어셈블리·벤치로 정량화.

## 🎨 스타일 가이드

1. **거부당하는 코드로 가르친다** — "컴파일 안 되는 예 → 왜 → 어떻게 고치나" 순서, 에러 메시지 직접 인용
2. **zero-cost를 증명** — 추상화 코드와 수동 코드의 어셈블리가 같음을 godbolt로
3. **GC 언어와 항상 대조** — 같은 패턴을 Java/Go로 짜면 어떻게 다른지
4. **소유권 다이어그램** — move/borrow/lifetime을 스택·힙 그림으로
5. unsafe는 "왜 안전한지" 불변식을 명시적으로 서술

## 🔬 검증 환경

> docker 불필요. **cargo 툴체인 + godbolt + miri**가 검증 도구.

```dockerfile
FROM rust:1-slim
RUN rustup component add miri rust-src clippy && \
    cargo install cargo-asm cargo-expand
```

```bash
# 빌림 검사기 에러 읽기 (학습의 핵심)
cargo build            # E0502, E0382 등 에러 코드와 설명
rustc --explain E0502  # 에러의 의미

# zero-cost 증명: 생성 어셈블리
cargo asm --rust <function>     # 또는 godbolt.org (rustc -O)
cargo expand                    # async/매크로가 펼쳐진 코드(상태머신) 확인

# UB 검출
cargo +nightly miri run         # unsafe 코드의 미정의 동작 잡기

# 성능
cargo bench                     # criterion
perf stat ./target/release/app
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🧱 Languages 톤, "컴파일 통과 vs 무엇을 증명하나" 포지셔닝, `🔗 레포 연결`(memory/concurrency 수렴)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- *The Rust Programming Language* (공식 책) — https://doc.rust-lang.org/book/
- *Rust for Rustaceans* (Jon Gjengset)
- *Programming Rust* (Blandy, Orendorff, Tindall)
- Rustonomicon (unsafe) — https://doc.rust-lang.org/nomicon/
- Async Book — https://rust-lang.github.io/async-book/
- Jon Gjengset 유튜브(Crust of Rust)

## 💡 핵심 분석 대상

```
소유권 — aliasing XOR mutability:
  let mut s = String::from("hi");
  let r1 = &s;       // 불변 빌림 OK
  let r2 = &s;       // 여러 불변 빌림 OK
  let r3 = &mut s;   // ✗ 불변 빌림이 살아있는 동안 가변 빌림 불가
  → "공유하거나(읽기) 변경하거나(쓰기), 동시엔 안 됨"
  → 이 규칙 하나가 데이터 레이스를 컴파일 타임에 제거

세 가지 메모리 관리:
  C++   : 수동 free → UAF·이중해제·누수 위험
  Java  : GC → 안전하지만 런타임 비용·STW
  Rust  : 소유권 → 컴파일 타임 검증, 런타임 비용 0

async → 상태머신:
  async fn f() { a().await; b().await; }
  컴파일러가 생성:
  enum F { Start, AfterA, AfterB, Done }
  poll() 호출마다 상태 전이 → await에서 Pending 반환하고 양보
  → 런타임(Tokio)이 Waker로 다시 poll

Send/Sync:
  Send  : 다른 스레드로 소유권 이동 가능 (Rc는 ✗, Arc는 ✓)
  Sync  : 여러 스레드가 &T 공유 가능 (RefCell ✗, Mutex ✓)
  → 컴파일러가 타입만 보고 스레드 안전성 판정
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 42개 확인 + cargo/godbolt/miri 검증 환경 + go/java-concurrency/memory-management-compared 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
