# Compilation Strategies Compared 레포지토리 제작 프롬프트

나는 "Compilation Strategies Compared" 레포지토리를 만들려고 해.
이건 **횡단 비교(Synthesis)** 레포야. "소스 코드를 어떻게 실행 가능한 형태로 바꾸는가"라는 한 질문에 여러 런타임이 *서로 다르게* 답한 방식 — JVM HotSpot, V8, ART, Dart VM, Hermes, GraalVM, 그리고 순수 AOT 언어(Rust/Go/Swift)를 한자리에 놓고 비교한다.
"한 엔진의 JIT를 아는 것"과 "모든 런타임이 같은 트레이드오프 공간(빠른 시작·빠른 피크·메모리·이식성)에서 다른 점을 골랐다는 걸 아는 것"의 차이를 만드는 레포다.

> 이 레포는 **Memory Management Compared와 한 쌍**으로 "실행엔진" 횡단 렌즈를 완성한다. 한쪽은 *어떻게 코드를 실행하나*(이 레포), 다른 쪽은 *어떻게 메모리를 회수하나*(memory-management-compared).

## 📋 프로젝트 목표

**컨셉**: "한 런타임의 JIT를 아는 것과, 모든 컴파일 전략이 '언제·얼마나 최적화하나'라는 같은 질문에 다르게 답한 것임을 아는 것은 다르다"

**핵심 차별화**:
1. 공통 축 — 모든 전략 = "빠른 시작 vs 빠른 피크 vs 메모리 vs 이식성"의 4축 공간 위의 점
2. 티어 컴파일의 보편성 — V8 4티어·JVM C1/C2·ART JIT/AOT가 *같은 패턴*(시작은 빠르게, 핫은 공격적으로)
3. 투기와 역최적화 — 동적 언어가 정적 성능을 흉내내는 공통 메커니즘
4. 프로파일 기반 — JVM PGO·V8 피드백·ART Baseline Profile·Hermes 바이트코드가 같은 아이디어

**타겟 독자**:
- 한 엔진의 JIT는 알지만 다른 런타임과 비교 못하는 개발자
- "AOT vs JIT"를 단순 이분법으로만 아는 개발자
- 티어드 컴파일이 보편 패턴이라는 걸 모르는 개발자
- 새 런타임(Hermes·Dart·GraalVM)을 빠르게 평가하고 싶은 개발자
- 개별 런타임 레포(아래 선행)를 마치고 큰 그림을 원하는 개발자

## 🔗 레포 연결

**⬆️ 선행(이 비교의 입력)**:
`compiler-deep-dive`(컴파일러 일반 이론), `jvm-deep-dive`(HotSpot C1/C2), `v8-engine-deep-dive`(Ignition→TurboFan 4티어), `android-runtime-deep-dive`(ART AOT/JIT·Baseline Profile), `swift-deep-dive`(AOT·SIL), `rust-deep-dive`(AOT·단형화), `go-deep-dive`(AOT·인라이닝), `flutter-deep-dive`(Dart JIT/AOT 하이브리드), `react-native-deep-dive`(Hermes 바이트코드).
**🤝 시너지**: `computer-architecture-deep-dive`(생성 기계어가 도는 곳), `web-apis-wasm-deep-dive`(WASM = 이식 가능 바이트코드).
**🧬 본질**: 이 레포가 수렴점 — Memory Management Compared와 함께 *실행엔진* 횡단 렌즈를 완성한다.

---

## 🎯 1단계: 전체 구조 설계

> "런타임별"이 아니라 **"전략 축별"**로 구성, 각 축에서 여러 런타임을 나란히.

### Chapter 1: 컴파일의 근본 질문 (5개 문서)
- 무엇을 결정해야 하나 — 언제 컴파일·얼마나 최적화·어디 저장
- 4축 트레이드오프 공간 — 콜드 스타트·피크 처리량·바이너리/메모리·이식성·개발 반복
- 인터프리터 vs 컴파일 vs 둘 다 — 양극단의 비용과 모든 현대 런타임이 둘 다 쓰는 이유
- 동적 언어의 도전 — 타입이 런타임에만 있는 언어를 어떻게 정적 성능에 가깝게(v8 연결)
- 비교 프레임 — 평가 축 정의(이 레포가 쓸 측정 기준)

### Chapter 2: 인터프리터 — 모두의 출발점 (5개 문서)
- 인터프리터 역할 — 빠른 시작·메모리 절약, 모든 동적 런타임의 1티어
- V8 Ignition·JVM 인터프리터·ART mterp·Dart 커널 — 같은 역할의 다른 구현
- 바이트코드 vs 트리 워킹 — Ignition·JVM bytecode·Dart kernel vs 트리 인터프리터
- 프로파일 수집 — 인터프리터가 피드백을 모아 다음 티어에 넘기는 공통 패턴
- 나란히 비교 — 같은 함수의 인터프리터 시작 시간·메모리를 4런타임으로

### Chapter 3: JIT 전략 — 티어드의 보편성 (6개 문서)
- 티어드 컴파일 철학 — 빠른 시작 + 빠른 피크를 둘 다 잡는 방식
- V8 4티어 — Ignition→Sparkplug→Maglev→TurboFan, 각 단계의 비용/이득
- JVM HotSpot — 인터프리터→C1(빠른 컴파일)→C2(공격적), 티어드 컴파일러 옵션
- ART JIT — 인터프리터+JIT, 프로파일 수집해 다음 실행에 쓰기
- 티어업/티어다운 — 언제 다음 단계로, 가정 깨지면 어떻게 내려오나
- 나란히 — V8·JVM·ART 티어 구조를 같은 다이어그램에 매핑(놀라울 정도로 동형)

### Chapter 4: AOT 전략 (5개 문서)
- 순수 AOT — Rust·Go·Swift·Dart 릴리스, 빌드 시 모든 컴파일
- AOT의 득실 — 빠른 시작·예측 가능 vs 빌드 시간·정적 정보 한계
- 정적 언어 단형화 — Rust·Swift 제네릭 특수화(rust·swift 연결)
- GraalVM Native Image — JVM의 AOT 선회, 트레이드오프
- 모바일 AOT — ART speed 모드·Swift 릴리스, 모바일이 AOT를 사랑하는 이유

### Chapter 5: 하이브리드 — JIT + AOT (6개 문서)
- 하이브리드가 답인 이유 — 개발(JIT 반복) + 릴리스(AOT 시작)의 양면
- Dart VM — JIT(개발 hot reload) + AOT(릴리스), 같은 언어 두 모드(flutter 연결)
- ART 하이브리드 — 설치 시 verify·실행 시 JIT·유휴 시 AOT·Baseline Profile(android-runtime 연결)
- Hermes — 빌드 타임 바이트코드 사전 컴파일, RN 시작 가속(react-native 연결)
- Baseline/Cloud Profile — 프로파일을 빌드 산출물처럼 배포(ART), 같은 아이디어가 V8/JVM에도
- 나란히 — Dart·ART·Hermes의 "개발/릴리스 분리"가 같은 아이디어임을 보여준다

### Chapter 6: 특수화와 투기 — 동적 언어를 정적처럼 (6개 문서)
- 투기 최적화 — "이 타입일 거라 가정하고 컴파일"이 모든 동적 JIT의 공통 무기
- V8 Hidden Class & IC — 객체 형태 가정(v8 연결), monomorphic/polymorphic/megamorphic
- JVM inline cache·콜 사이트 특수화 — 인터페이스 호출 가속
- 역최적화(Deopt) — V8·JVM·ART가 *같은 방식*으로 가정 위반 시 되돌리기
- 정적 단형화 vs 동적 특수화 — Rust 단형화(컴파일 타임) vs V8 IC(런타임), 같은 목표 다른 시점
- 같은 패턴 검증 — `--trace-opt`/`--trace-deopt`·JVM CompileLog로 *같은 메커니즘* 관찰

### Chapter 7: 트레이드오프 종합 (5개 문서)
- 4축 종합 — 콜드 스타트·피크·메모리·이식성을 표로(같은 워크로드 측정)
- 같은 프로그램, 다른 런타임 — 동일 로직을 5+런타임으로 측정 비교
- 왜 수렴했나 — 모두 "티어 + 프로파일 + 투기"로 간 이유
- WASM의 자리 — 이식 가능 바이트코드로서의 WASM, 모든 전략의 신선한 점(web-apis 연결)
- 종합 — 컴파일 전략 지형도, 새 런타임을 만나면 이 4축으로 분류

→ **총 38개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). 비교가 핵심이라 `🔬`·`📊`에서 *항상 여러 런타임을 나란히*. `💻 실전 실험`은 같은 함수를 여러 엔진으로 실행해 티어 전이·deopt 관찰.

## 🎨 스타일 가이드

1. **항상 나란히** — 한 런타임만 말하지 말고 최소 2~3개 엔진을 같은 축으로
2. **티어 구조 동형성 강조** — V8·JVM·ART의 티어를 같은 다이어그램에 매핑
3. **선행 레포로 깊이 위임** — 각 엔진 *내부*는 해당 레포로, 여기선 *비교*
4. **deopt를 보여준다** — 가정 위반→되돌리기를 여러 엔진에서 *직접 관찰*
5. **memory-management-compared와 쌍** — "실행엔진 = 컴파일(이 레포) + 메모리(저 레포)" 명시
6. 4축 트레이드오프·티어 매핑은 표/다이어그램으로

## 🔬 검증 환경

> polyglot. 같은 함수를 여러 런타임으로 실행해 티어·deopt·시작 측정.

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y \
    openjdk-21-jdk \           # JVM (-XX:+PrintCompilation 등)
    nodejs npm \               # V8 (--trace-opt --trace-deopt)
    golang-go                  # Go AOT
RUN curl https://sh.rustup.rs -sSf | sh -s -- -y    # Rust AOT
# Dart/Hermes/Swift/ART는 별도 환경
```

```bash
# 핵심: 같은 함수의 컴파일 동작을 여러 엔진에서 관찰

# V8 티어 추적
node --trace-opt --trace-deopt --print-bytecode app.js
node --allow-natives-syntax app.js   # %OptimizeFunctionOnNextCall 등

# JVM 티어드 컴파일
java -XX:+PrintCompilation -XX:+UnlockDiagnosticVMOptions \
     -XX:+PrintInlining App
# C1/C2 전환·OSR·deopt(uncommon trap) 로그

# ART 컴파일 모드 강제
adb shell cmd package compile -m speed -f com.example.app   # AOT
adb shell cmd package compile -m verify -f com.example.app  # 최소

# AOT 산출물 어셈블리 (Rust/Go/Swift)
cargo rustc --release -- --emit asm    # Rust
go build -gcflags='-S' .                # Go
swiftc -emit-assembly -O demo.swift     # Swift

# WASM과의 대조
# 같은 함수를 JS vs WASM으로 → 시작·실행 비교

# 측정 매트릭스: 같은 워크로드 5런타임
#   콜드 스타트(ms) · 피크 처리량(ops/s) · 바이너리/메모리 · 빌드 시간
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `comparisons/`(런타임별 동일 함수)
2. **README.md**: 🧬 Synthesis 톤, "한 엔진 JIT vs 모든 전략의 트레이드오프" 포지셔닝, 선행 8+개 레포를 `🔗 레포 연결` 입력으로, **memory-management-compared와 함께 실행엔진 렌즈를 완성함을 명시**
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어, 항상 다중 런타임 비교

## 📚 참고 자료

- 각 선행 레포의 참고자료(이 레포는 그 종합)
- V8 블로그(티어드 컴파일 글 시리즈) — https://v8.dev/blog
- "Compiling at Scale" / HotSpot JIT 발표들
- "GraalVM Native Image" 문서
- "Hermes: An Optimized JavaScript Engine for React Native"
- Dart 컴파일 모델 문서

## 💡 핵심 분석 대상

```
4축 트레이드오프 공간 (모든 런타임의 위치):
                  콜드 스타트 ◄────────► 빠른 피크
  인터프리터만 ●                                  ● 순수 AOT(Rust/Go/Swift)
                                                 ● ART speed
            ● Hermes(사전 바이트코드)
                  ● V8/JVM 티어드(워밍업 필요)
                  ● Dart 하이브리드

  ◄────────────────────────────────────────────►
  큰 바이너리/메모리                    작은 바이너리

티어드 컴파일 동형성:
  V8 :  Ignition  →  Sparkplug  →  Maglev   →  TurboFan
  JVM:  Interp.   →  C1                     →  C2
  ART:  Interp.   →  JIT(빠른)              →  AOT(speed)
                  ↑──── 같은 패턴 ────↑
  공통 아이디어: "측정하며 점점 공격적으로, 가정 깨지면 되돌림"

투기 + 역최적화 (동적 언어가 정적처럼):
  V8 : Hidden Class 가정 → 깨지면 Deopt → 인터프리터 복귀
  JVM: inline cache·class hierarchy 가정 → uncommon trap → C1로
  ART: 프로파일 가정 → deopt → 인터프리터
  → 모두 "낙관 + 안전망"

프로파일 기반의 보편성:
  V8 피드백 벡터 — 인터프리터가 모은 타입 정보
  JVM PGO       — -XX:+UseProfilingDriven...
  ART Baseline Profile — Play가 집계해 배포
  Hermes 바이트코드    — 빌드 시 사전 변환
  → "프로파일을 빌드 산출물로" 점점 확장(클라우드 프로파일)

같은 함수, 다른 답:
  fun add(a, b) = a + b
  Rust   : 컴파일 타임 단형화 → 직접 inline (런타임 0)
  Swift  : SIL 특수화 → 인라인 (런타임 0)
  V8     : Ignition 실행 → 핫 → Maglev에 IC 정보로 inline → TurboFan
  JVM    : 인터프리터 → C1 → C2가 inline cache로 inline
  Hermes : 빌드 시 바이트코드, 런타임은 인터프리터
  → 같은 결과를 *다른 시점*에 만든다
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 38개 확인 + polyglot 검증 환경(V8/JVM/ART 플래그·AOT 어셈블리) + 선행 8+개 레포(compiler·jvm·v8·android-runtime·swift·rust·go·flutter·react-native)를 입력으로 하는 연결 명시. **memory-management-compared와 한 쌍으로 실행엔진 렌즈 완성**을 분명히. **티어드 컴파일의 보편성**을 보여주는 게 이 레포의 핵심 정체성.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
