# Swift Concurrency Deep Dive 레포지토리 제작 프롬프트

나는 "Swift Concurrency Deep Dive" 레포지토리를 만들려고 해.
async/await가 어떻게 상태머신이 되는지, Actor가 어떻게 데이터 레이스를 컴파일 타임에 막는지, Sendable·격리가 무엇을 보장하는지, GCD에서 structured concurrency로의 전환을 완전히 파헤치는 레포야.
"async/await를 쓰는 것"과 "await가 스레드를 어떻게 양보하고 Actor 격리가 무엇을 보장하는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "async/await를 쓰는 것과, 그것이 상태머신으로 컴파일되고 Actor가 격리로 안전을 보장하는 방식을 아는 것은 다르다"

**핵심 차별화**:
1. async/await 상태머신 — suspend 지점에서 스레드를 양보하고 재개하는 컴파일러 변환
2. structured concurrency — 작업 트리·취소 전파가 누수를 막는 메커니즘
3. Actor 격리 — 가변 상태를 직렬화해 데이터 레이스를 *컴파일 타임에* 제거
4. Sendable — 스레드 경계를 넘어도 안전한 타입을 타입 시스템으로 검증

**타겟 독자**:
- async/await를 쓰지만 어떻게 멈추고 재개되는지 모르는 개발자
- GCD `DispatchQueue`만 쓰던 개발자
- Actor·`@MainActor`를 외워 쓰는 개발자
- "Sendable 경고"의 의미를 모르는 개발자
- `kotlin-deep-dive`(코루틴)·`rust`(Send/Sync)·`go`와 비교하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `swift-deep-dive`(값 타입·클로저·ARC), `ios-lifecycle-runloop-deep-dive`(RunLoop·메인 스레드).
**🤝 시너지**: `kotlin-deep-dive`(코루틴 상태머신 동형), `rust-deep-dive`(Send/Sync 대조), `swiftui-internals-deep-dive`(@MainActor·async UI).
**🧬 수렴**: `concurrency-models-compared`(async/Actor ↔ 코루틴 ↔ 고루틴 ↔ Virtual Thread ↔ Event Loop).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 동시성의 역사와 기초 (5개 문서)
- 왜 새 모델인가 — 콜백 지옥·GCD의 한계, 안전성·구조의 필요
- GCD 복습 — 디스패치 큐·동기/비동기·QoS, 여전히 아래에 있음
- 스레드 vs 태스크 — Swift Concurrency의 협력적 스레드 풀, 스레드 폭발 방지
- 동시성 vs 병렬성 — 개념 구분, Swift의 접근
- 마이그레이션 관점 — GCD에서 async/await로의 전환 지도

### Chapter 2: async/await 내부 (6개 문서)
- async 함수 — 일시정지 가능한 함수, 호출 규약(continuation)
- await의 의미 — suspend 지점, 스레드 양보, 재개
- 상태머신 변환 — 컴파일러가 async를 상태머신으로(kotlin 코루틴과 동형)
- continuation — 일시정지/재개의 저수준, withCheckedContinuation으로 콜백 브리징
- 협력적 스레드 풀 — 스레드를 막지 않고 양보, 풀 크기 = 코어 수
- async let·병렬 — 동시 실행, 결과 await, 직렬 vs 병렬

### Chapter 3: Structured Concurrency (6개 문서)
- 구조적 동시성 — 작업이 스코프에 묶임, 부모-자식 트리
- Task — 작업 단위, 생성·우선순위, unstructured Task의 위험
- TaskGroup — 동적 병렬 작업, 자식 수집, 부분 실패
- 취소 — 협력적 취소, isCancelled·Task.checkCancellation, 전파
- 취소 전파 — 부모 취소→자식 취소, 누수 방지(kotlin 구조적 동시성과 동형)
- 우선순위 역전·승계 — QoS, 우선순위 처리

### Chapter 4: Actor와 격리 (7개 문서)
- Actor 개념 — 가변 상태를 보호하는 격리 단위, "한 번에 하나만 접근"
- Actor 격리 — 외부 접근은 await(비동기), 내부는 동기, 직렬 실행
- 데이터 레이스 제거 — 컴파일러가 격리 위반을 *컴파일 에러*로(rust Send/Sync 대조)
- reentrancy — Actor 재진입, await 중 상태 변경 가능성(함정)
- @MainActor — UI는 메인에서, 메인 액터 격리, SwiftUI와 결합
- nonisolated — 격리 제외, 언제 안전
- global actor — 커스텀 전역 액터, 도메인 격리

### Chapter 5: Sendable과 데이터 안전 (5개 문서)
- Sendable — 스레드 경계를 넘어도 안전한 타입의 표시
- Sendable 검사 — 값 타입·불변·내부 동기화의 조건, 자동/수동 준수
- @Sendable 클로저 — 캡처 안전성, 클로저가 경계를 넘을 때
- 격리와 Sendable의 결합 — 무엇이 액터 경계를 넘을 수 있나
- Swift 6 엄격 동시성 — 컴파일 타임 데이터 레이스 안전성, 마이그레이션

### Chapter 6: 비동기 시퀀스와 통합 (5개 문서)
- AsyncSequence — 비동기 이터레이션, for await
- AsyncStream — 콜백·델리게이트를 async 스트림으로 브리징
- Combine과의 관계 — 기존 Combine과 async/await, 전환
- 기존 API 브리징 — completion 핸들러를 async로, continuation
- 테스트 — async 코드 테스트, 액터 격리 테스트

### Chapter 7: 성능·디버깅·실전 (4개 문서)
- 성능 — 태스크 생성 비용, 과한 액터 홉(hop), 측정
- 디버깅 — Instruments Swift Concurrency, 태스크·액터 추적
- 흔한 함정 — 액터 재진입·우선순위 역전·메인 액터 블로킹
- 종합 — GCD 기반 코드를 async/await + Actor로 마이그레이션·검증

→ **총 40개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 상태머신·액터 격리·스레드 풀, `💻 실전 실험`은 Instruments Concurrency·격리 위반 컴파일 에러 재현. `📊`는 GCD vs async/await·액터 홉 비용.

## 🎨 스타일 가이드

1. **컴파일 에러로 가르친다** — 격리·Sendable 위반을 *컴파일러가 잡는* 예로
2. **상태머신으로 환원** — async를 kotlin 코루틴과 같은 상태머신으로
3. **3+언어 대조** — Actor(Swift) vs Send/Sync(Rust) vs 채널(Go) vs 코루틴(Kotlin)
4. **GCD와 연결** — 새 모델이 GCD 위에서 어떻게 도는지
5. 상태머신·액터 격리·태스크 트리는 다이어그램으로

## 🔬 검증 환경

> docker 가능(Linux Swift) 또는 macOS+Xcode. **컴파일러 진단 + Instruments**.

```dockerfile
FROM swift:6-jammy   # Swift 6 엄격 동시성 검사
```

```bash
# 엄격 동시성으로 데이터 레이스 컴파일 검출 (핵심)
swiftc -strict-concurrency=complete demo.swift
#  → Sendable 위반·격리 위반이 "컴파일 에러/경고"로 나옴
#  → 의도적으로 공유 가변 상태 접근 → 컴파일러가 막는 것 확인

# 상태머신 관찰 (kotlin 코루틴처럼)
swiftc -emit-sil demo.swift | grep -A5 async    # async 함수의 SIL

# macOS Instruments:
#  Swift Concurrency 템플릿 → 태스크 생성·액터 홉·suspend 지점 시각화

# 실험:
#  - actor 카운터에 동시 접근 → 격리로 직렬화 확인(레이스 없음)
#  - GCD 동일 코드 vs actor → 안전성·코드 비교
#  - async let 병렬 vs 순차 await → 시간 측정
#  - 협력적 풀: 수천 Task vs 수천 스레드 비교
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🍎 iOS 톤, "async를 쓴다 vs 상태머신·격리를 안다" 포지셔닝, `🔗 레포 연결`(swift·kotlin·rust·concurrency-models-compared)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- Swift Concurrency 문서 — https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/
- "Meet async/await"·"Protect mutable state with actors" WWDC21
- "Eliminate data races using Swift Concurrency" WWDC22
- Swift Evolution 동시성 제안(SE-0296 등)
- "Swift Concurrency: Behind the Scenes" WWDC21

## 💡 핵심 분석 대상

```
async/await = 상태머신 (kotlin 코루틴과 동형):
  func load() async {
    let a = await fetchA()   // suspend 지점 1
    let b = await fetchB(a)  // suspend 지점 2
    show(a, b)
  }
  → 컴파일러가 상태머신으로: await에서 스레드 양보, 재개 시 점프
  → 스레드를 막지 않음(협력적 풀의 스레드 재사용)

Actor (데이터 레이스 컴파일 타임 제거):
  actor Counter {
    var value = 0
    func inc() { value += 1 }   // 내부: 동기
  }
  let c = Counter()
  await c.inc()    // 외부: await 필수(직렬화)
  → 동시 접근해도 actor가 한 번에 하나만 → 레이스 불가
  → 직접 c.value 접근 시 컴파일 에러

격리 vs Rust Send/Sync:
  Swift: actor 격리 + Sendable (런타임 직렬화 + 타입 검사)
  Rust : 소유권 + Send/Sync (순수 타입 검사, 런타임 0)
  Go   : 채널/고루틴 (공유 대신 통신)
  → 같은 목표(레이스 제거), 다른 메커니즘(concurrency-models-compared)

structured concurrency:
  withTaskGroup { group in
    group.addTask { ... }   // 자식
    group.addTask { ... }
  }  // 스코프 끝 = 모든 자식 완료/취소 대기
  → 누수 0, 취소 전파(kotlin과 동형)
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 40개 확인 + 엄격동시성/Instruments 검증 환경 + swift/kotlin/rust/concurrency-models-compared 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
