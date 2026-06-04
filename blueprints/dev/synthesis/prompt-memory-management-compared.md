# Memory Management Compared 레포지토리 제작 프롬프트

나는 "Memory Management Compared" 레포지토리를 만들려고 해.
이건 **횡단 비교(Synthesis)** 레포야. "메모리를 어떻게 안전하게 회수하는가"라는 한 질문에 여러 런타임이 *서로 다르게* 답한 방식 — JVM GC, V8 Orinoco, Go GC, ART, Swift ARC, Rust 소유권(GC 없음)을 한자리에 놓고 비교한다.
"한 언어의 메모리 관리를 아는 것"과 "GC·ARC·소유권이 같은 문제의 트레이드오프 다른 점이라는 걸 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "한 런타임의 GC를 아는 것과, 모든 메모리 관리가 '언제 회수하나 + 누가 비용을 치르나'라는 질문에 다르게 답한 것임을 아는 것은 다르다"

**핵심 차별화**:
1. 공통 질문 — 모두 "더 이상 안 쓰는 메모리를 어떻게 안전하게 회수하나"를 푼다
2. 세 가지 답 — 추적 GC(JVM/V8/Go/ART) · 참조 카운팅(Swift ARC) · 소유권(Rust, 런타임 0)
3. 비용의 위치 — 런타임 STW(GC) vs 카운팅 오버헤드(ARC) vs 컴파일 타임 복잡도(Rust)
4. 같은 누수, 다른 약점 — 순환 참조·메모리 누수가 각 모델에서 어떻게 다르게 나타나나

**타겟 독자**:
- 한 언어 GC는 알지만 ARC·소유권과 비교 못하는 개발자
- "GC vs ARC vs Rust"를 정확히 구분 못하는 개발자
- 메모리 누수가 모델마다 어떻게 다른지 모르는 개발자
- 런타임 선택의 메모리 트레이드오프를 알고 싶은 개발자
- 개별 런타임 레포를 마치고 큰 그림을 원하는 개발자

## 🔗 레포 연결

**⬆️ 선행(이 비교의 입력)**:
`jvm-deep-dive`(JVM GC), `v8-engine-deep-dive`(Orinoco GC), `go-deep-dive`(Go GC), `android-runtime-deep-dive`(ART GC), `swift-deep-dive`(ARC), `rust-deep-dive`(소유권).
**🤝 시너지**: `computer-architecture-deep-dive`(메모리 계층·캐시 — 모든 GC 비용의 바닥), `compiler-deep-dive`(소유권·ARC는 컴파일러가 삽입).
**🧬 본질**: 이 레포가 수렴점 — 6개 런타임의 메모리 전략을 하나의 프레임으로.

---

## 🎯 1단계: 전체 구조 설계

> "런타임별"이 아니라 **"질문별"**로 구성, 각 챕터에서 모델을 나란히 비교.

### Chapter 1: 메모리 관리의 근본 문제 (5개 문서)
- 두 가지 질문 — ① 언제 회수가 안전한가(도달 불가) ② 누가 비용을 치르나
- 수동 관리의 문제 — C/C++의 UAF·이중해제·누수, 왜 자동화가 필요한가
- 메모리 계층 — 회수 비용이 캐시·대역폭과 얽히는 지점(computer-architecture 연결)
- 세 가지 접근 — 추적 GC·참조 카운팅·소유권의 개념 소개
- 비교 프레임 — 평가 축(지연·처리량·메모리 오버헤드·결정성·복잡도)

### Chapter 2: 추적 GC — "도달 가능성" (6개 문서)
- 추적의 원리 — 루트에서 도달 가능한 객체만 살리기, 나머지 회수
- 표시-쓸기·복사·압축 — 기본 알고리즘, 단편화 해결
- 세대 가설 — 대부분 객체는 일찍 죽는다, young/old 분리(JVM·V8·ART 공통)
- JVM GC — G1/ZGC, STW 최소화, 힙 영역화(jvm 연결)
- V8 Orinoco — Scavenge + Mark-Compact, 동시·병렬(v8 연결)
- Go GC — 동시 삼색 표시, 세대 없음, 짧은 STW 목표(go 연결)

### Chapter 3: 동시·증분 GC — STW 줄이기 (5개 문서)
- STW 문제 — Stop-The-World가 지연·jank를 만드는 이유(android-performance 연결)
- 삼색 표시 — white/grey/black, 동시 표시의 정확성
- 쓰기 배리어 — 동시 표시 중 변이 처리, 모든 동시 GC의 공통 도구
- ART GC — Concurrent Copying, 모바일 jank 최소화(android-runtime 연결)
- 동시 GC 비교 — JVM ZGC·Go·ART의 STW·처리량 트레이드오프

### Chapter 4: 참조 카운팅 — Swift ARC (5개 문서)
- 참조 카운팅 원리 — 참조 수를 세어 0이면 즉시 해제, 결정적
- ARC — 컴파일러가 retain/release 삽입(swift 연결), GC 스레드 없음
- GC vs ARC — 결정성·메모리 즉시 회수 vs 카운팅 오버헤드·순환 약점
- 순환 참조 — RC의 근본 약점, weak/unowned로 수동 해결
- ARC 비용 — retain/release 빈도, 원자적 카운팅, 최적화

### Chapter 5: 소유권 — Rust (런타임 0) (5개 문서)
- 소유권 모델 — 컴파일 타임에 수명 결정, GC도 RC도 없음(rust 연결)
- RAII — 스코프 종료 시 결정적 해제, drop
- 빌림 검사 — 컴파일 타임에 UAF·이중해제 제거
- Rc/Arc — Rust도 필요시 참조 카운팅, 선택적
- 소유권의 비용 — 런타임 0 대신 컴파일 타임 복잡도(학습·표현 제약)

### Chapter 6: 같은 문제, 다른 약점 (6개 문서)
- 순환 참조 — GC(자동 처리) vs ARC(누수·수동 weak) vs Rust(Rc 순환 주의)
- 메모리 누수 — 각 모델에서 누수가 생기는 다른 방식
- 지연 vs 처리량 — GC의 STW vs ARC의 분산 비용 vs Rust의 예측성
- 메모리 오버헤드 — GC 헤드룸(여유 힙) vs ARC 카운트 필드 vs Rust 최소
- 결정성 — 언제 해제되나(GC 비결정 vs ARC/Rust 결정적)
- 같은 프로그램 비교 — 동일 데이터 구조를 6런타임으로 메모리·지연 측정

### Chapter 7: 트레이드오프 종합 (4개 문서)
- 평가 축 종합 — 지연·처리량·오버헤드·결정성·복잡도 표
- 왜 갈렸나 — 서버(처리량 GC)·모바일(ARC/저지연 GC)·시스템(소유권)의 요구 차이
- 선택의 논리 — 워크로드가 메모리 전략을 결정하는 방식
- 종합 — 메모리 관리 지형도, 새 런타임을 만나도 이 프레임으로 분류

→ **총 36개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). 비교가 핵심이라 `🔬`·`📊`에서 *항상 여러 모델을 나란히*. `💻 실전 실험`은 같은 구조를 여러 런타임으로 측정.

## 🎨 스타일 가이드

1. **항상 나란히** — 한 모델만 말하지 말고 GC vs ARC vs 소유권을 같은 축으로
2. **질문으로 환원** — "언제 회수? 누가 비용?"으로 모든 기법 분류
3. **선행 레포로 깊이 위임** — 각 GC 내부는 해당 레포로, 여기선 *비교*
4. **약점을 나란히** — 순환 참조를 6모델에서 어떻게 다루나
5. 트레이드오프 축·삼색 표시·소유권은 다이어그램으로

## 🔬 검증 환경

> polyglot. 같은 데이터 구조를 6런타임으로 측정.

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y openjdk-21-jdk golang-go nodejs npm curl
RUN curl https://sh.rustup.rs -sSf | sh -s -- -y
# Swift: swift:6, Kotlin/ART: 별도
```

```bash
# 핵심: 같은 메모리 패턴을 6런타임으로 비교
# 패턴 예: "대량 객체 할당 → 일부 해제 → 회수 관찰"

# GC 관찰
java -Xlog:gc demo.java          # JVM GC 로그·STW
GODEBUG=gctrace=1 ./go-demo      # Go GC·STW
node --trace-gc demo.js          # V8 GC

# ARC 관찰 (Swift)
swiftc -emit-sil demo.swift | grep -E 'retain|release'   # 카운팅 삽입
# Instruments: ARC 트래픽·순환 누수

# Rust (런타임 0)
# drop 시점이 컴파일 타임 결정 → GC/RC 로그 없음
cargo build   # 누수는 컴파일 타임 또는 Rc 순환만

# 순환 참조 비교:
#   Java/Go/JS: 순환이어도 GC가 회수(도달 불가면)
#   Swift: 강한 순환 → 누수(weak 필요)
#   Rust: Rc 순환 → 누수(Weak 필요)

# 측정: 동일 객체 그래프의 메모리 피크·해제 지연 6런타임 비교
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `comparisons/`(런타임별 동일 패턴)
2. **README.md**: 🧬 Synthesis 톤, "한 런타임 GC vs 모든 메모리 전략의 트레이드오프" 포지셔닝, 선행 6개 레포를 `🔗 레포 연결` 입력으로
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어, 항상 다중 모델 비교

## 📚 참고 자료

- *The Garbage Collection Handbook* (Jones, Hosking, Moss)
- 각 런타임 GC 문서(JVM·V8·Go·ART), Swift ARC 문서, Rust 소유권 문서
- "A Unified Theory of Garbage Collection" (Bacon et al. — GC와 RC는 쌍대)
- 각 선행 레포의 참고자료(이 레포는 그 종합)

## 💡 핵심 분석 대상

```
한 질문, 세 답:
  "안 쓰는 메모리를 언제·어떻게 회수하나?"
  ① 추적 GC : 주기적으로 도달 불가 객체 회수 (JVM/V8/Go/ART)
  ② 참조 카운팅: 참조 0 되면 즉시 (Swift ARC)
  ③ 소유권   : 컴파일 타임에 수명 결정, 스코프 끝 해제 (Rust)

비용의 위치:
  GC  : 런타임 STW + 여유 힙 메모리 + GC 스레드 CPU
  ARC : retain/release 카운팅 오버헤드(분산), 순환 누수 위험
  Rust: 런타임 0, 대신 컴파일 타임 복잡도·표현 제약

순환 참조 (같은 문제, 다른 약점):
  A→B→A 순환:
    추적 GC: 루트에서 도달 불가면 회수(순환 OK) ✓
    ARC    : 카운트가 0 안 됨 → 누수 → weak 필요 ✗
    Rust Rc: 마찬가지 누수 → Weak 필요 ✗
  → GC의 강점이 RC/소유권의 약점

결정성:
  GC  : 언제 해제될지 모름(비결정적), finalizer 위험
  ARC : 참조 0 되는 즉시(결정적)
  Rust: 스코프 종료 시(결정적, 컴파일 타임 예측)

왜 갈렸나:
  서버(JVM/Go): 처리량 중시 → 추적 GC
  모바일(Swift/ART): 저지연·메모리 제약 → ARC / 저지연 GC
  시스템(Rust): 예측성·런타임 0 → 소유권
  → 워크로드가 메모리 전략을 결정
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 36개 확인 + polyglot 검증 환경 + 선행 6개 레포(jvm·v8·go·android-runtime·swift·rust)를 입력으로 하는 연결 명시. **항상 GC vs ARC vs 소유권을 나란히** 놓는 게 이 레포의 정체성.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
