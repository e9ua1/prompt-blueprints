# Swift Deep Dive 레포지토리 제작 프롬프트

나는 "Swift Deep Dive" 레포지토리를 만들려고 해.
ARC가 어떻게 GC 없이 메모리를 관리하는지, 값 타입과 COW가 어떻게 안전과 성능을 동시에 잡는지, 프로토콜 지향과 제네릭이 내부에서 어떻게 디스패치되는지를 완전히 파헤치는 레포야.
"Swift를 작성하는 것"과 "ARC가 언제 retain/release를 넣고 프로토콜이 어떻게 디스패치되는지 알고 설계하는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "Swift를 쓰는 것과, ARC·값 타입·프로토콜이 메모리와 디스패치를 어떻게 다루는지 아는 것은 다르다"

**핵심 차별화**:
1. ARC — 컴파일러가 retain/release를 삽입해 GC 없이 결정적으로 메모리 관리하는 원리
2. 값 타입과 COW — struct의 값 의미론 + Copy-on-Write로 안전과 성능을 동시에
3. 프로토콜 지향 — Witness Table·Existential Container, 정적/동적 디스패치의 경계
4. 제네릭 특수화 — 단형화로 추상화 비용을 없애는 방식, 메모리 레이아웃

**타겟 독자**:
- Swift를 쓰지만 ARC가 언제 메모리를 푸는지 모르는 개발자
- 순환 참조·메모리 누수를 weak/unowned로 외워 막는 개발자
- struct vs class 선택을 "느낌"으로 하는 개발자
- 프로토콜·제네릭의 디스패치 비용을 모르는 개발자
- `rust-deep-dive`(소유권)·`computer-architecture`와 비교하며 메모리를 보려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `computer-architecture-deep-dive`(스택/힙·메모리 레이아웃·캐시), `compiler-deep-dive`(제네릭 특수화·디스패치).
**🤝 시너지**: `rust-deep-dive`(소유권 vs ARC), `objc-runtime-deep-dive`(ARC의 뿌리), `swift-concurrency-deep-dive`(Actor·Sendable).
**🧬 수렴**: `memory-management-compared`(ARC ↔ GC ↔ Rust 소유권 — *대표적 "참조 카운팅" 답*).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: Swift 메모리 모델 (5개 문서)
- 값 vs 참조 타입 — struct/enum(값) vs class(참조), 복사 의미론
- 메모리 레이아웃 — 스택 vs 힙 배치, 크기·정렬, struct가 스택에 사는 조건
- 인라인 vs 박싱 — 작은 값의 인라인, 큰 값/프로토콜의 박싱
- mutating과 in-out — 값 타입 변경의 의미, in-out 파라미터
- 메모리 안전 — exclusive access, 동시 접근 검사

### Chapter 2: ARC 완전 분해 (6개 문서)
- ARC 원리 — 컴파일러가 retain/release 삽입, 참조 카운트, GC와의 차이
- retain/release 삽입 시점 — 어디에 들어가나, SIL/어셈블리로 확인
- 강한/약한/미소유 — strong/weak/unowned, 각각의 카운트 영향
- 순환 참조 — 강한 순환이 누수를 만드는 원리, weak/unowned로 끊기
- 클로저 캡처 — 클로저가 self를 강하게 잡는 함정, capture list
- ARC 성능 — retain/release 오버헤드, 불필요 카운팅 제거(최적화)

### Chapter 3: 값 타입과 COW (5개 문서)
- 값 의미론 — 복사 시 독립, 공유 상태 없음의 안전성
- Copy-on-Write — Array/String/Dictionary가 실제 복사를 미루는 원리
- COW 직접 구현 — isKnownUniquelyReferenced로 커스텀 COW 타입
- 값 타입 성능 — 복사 비용 vs 참조, 언제 struct가 빠른가
- struct vs class 결정 — 정체성·공유·상속 필요성 기준

### Chapter 4: 프로토콜과 디스패치 (7개 문서)
- 프로토콜 지향 — 상속 대신 프로토콜 + 익스텐션, POP 철학
- 메서드 디스패치 4종 — 정적·vtable·witness table·메시지(objc) 비교
- Witness Table — 프로토콜 메서드 디스패치, 정적 타입일 때
- Existential Container — `any Protocol`의 박싱, 3워드 버퍼·간접 비용
- some vs any — 불투명 타입(정적) vs existential(동적), 성능 차이
- 프로토콜 익스텐션 디스패치 — 기본 구현의 함정(정적 디스패치)
- 디스패치 측정 — SIL/어셈블리로 어떤 디스패치인지 확인

### Chapter 5: 제네릭 (5개 문서)
- 제네릭 기본 — 타입 파라미터·제약, 프로토콜 제약
- 단형화(Specialization) — 컴파일러가 타입별 특수화, zero-cost
- 제네릭 vs Existential — 정적(특수화) vs 동적(박싱) 트레이드오프
- associatedtype — 프로토콜의 연관 타입, PAT 제약
- 제네릭 성능 — 특수화 안 될 때의 비용, @inlinable

### Chapter 6: 언어 기능 내부 (6개 문서)
- 옵셔널 — enum 기반, 메모리 표현, 옵셔널 체이닝 비용
- enum과 패턴 매칭 — 연관 값·페이로드, 메모리 레이아웃
- 에러 처리 — throws의 구현, Result, typed throws
- 클로저 — 캡처·이스케이프, 함수 표현, 컨텍스트 할당
- property wrapper·result builder — 매크로 유사 변환, SwiftUI 기반
- 메타타입·미러 — 타입 정보, 리플렉션 비용

### Chapter 7: 성능과 도구 (4개 문서)
- SIL 읽기 — Swift Intermediate Language로 컴파일러 동작 관찰
- 최적화 — -O 효과, 인라인·특수화·ARC 제거
- Instruments — Allocations·Leaks·ARC 추적(ios-lifecycle 연결)
- 종합 — 느린/누수 코드를 SIL·Instruments로 진단→수정

→ **총 42개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 ARC 삽입·디스패치·SIL, `💻 실전 실험`은 SIL 덤프·Instruments·어셈블리. `📊`는 값/참조·디스패치별 성능·메모리 비교.

## 🎨 스타일 가이드

1. **ARC를 "보이게"** — retain/release 삽입을 SIL/어셈블리로 *직접 본다*
2. **Rust 소유권과 대조** — ARC(런타임 카운트) vs Rust(컴파일타임 소유권)
3. **디스패치로 환원** — 성능 차이를 4종 디스패치로 분류
4. **memory-management-compared 예고** — ARC가 GC·소유권과 어떻게 다른지
5. 메모리 레이아웃·디스패치·COW는 다이어그램으로

## 🔬 검증 환경

> docker 가능(Linux Swift) 또는 macOS+Xcode. 핵심은 **SIL 덤프 + Instruments**.

```dockerfile
FROM swift:6-jammy   # Linux에서 SIL·어셈블리 관찰 가능
```

```bash
# SIL/어셈블리로 ARC·디스패치 관찰
swiftc -emit-sil demo.swift | swift demangle      # SIL (retain/release 보임!)
swiftc -emit-assembly -O demo.swift               # 최적화 어셈블리
swiftc -emit-sil -O demo.swift                    # 최적화 SIL(ARC 제거 관찰)

# 디스패치 확인: existential vs generic의 SIL 차이
#   any Protocol → init_existential, witness_method (동적)
#   some/generic → 특수화 후 직접 호출 (정적)

# macOS Instruments (실측):
#   Allocations: 할당·COW 복사
#   Leaks: 순환 참조 누수
#   "ARC 트래픽": retain/release 빈도

# 순환 참조 실험: 강한 순환 → 누수, weak로 → 해제 확인
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🍎 iOS 톤, "Swift를 쓴다 vs ARC·디스패치를 안다" 포지셔닝, `🔗 레포 연결`(rust·objc-runtime·memory-management-compared)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- Swift 공식 문서·The Swift Programming Language — https://docs.swift.org/swift-book/
- "Understanding Swift Performance" (WWDC16) — 값/참조·디스패치 핵심
- Swift 소스·SIL 문서 — https://github.com/apple/swift
- *Advanced Swift* (objc.io)
- "Embrace Swift Generics"·"Existential" WWDC 발표

## 💡 핵심 분석 대상

```
ARC (GC 없는 결정적 관리):
  let a = Person()      // 카운트 1
  let b = a             // retain → 카운트 2
  // b 스코프 끝         // release → 카운트 1
  // a 스코프 끝         // release → 카운트 0 → deinit 즉시
  → 컴파일러가 retain/release 삽입(GC처럼 나중에 X, 결정적)

순환 참조:
  Person.car (strong) → Car
  Car.owner (strong)  → Person
  → 서로 잡아 카운트 0 안 됨 → 누수
  → 한쪽을 weak: Car.owner (weak) → 순환 끊김

COW (값 안전 + 성능):
  var a = [1,2,3]
  var b = a          // 복사? 아직 아님(버퍼 공유)
  b.append(4)        // 여기서 쓰기 → isKnownUniquelyReferenced false
                     // → 실제 복사 발생 → b만 독립
  → 안 바꾸면 복사 0, 바꿀 때만 복사

디스패치 4종 (느림←→빠름):
  message(objc)  > witness/vtable(동적) > static(정적)
  any Protocol   → existential 박싱 + witness table (동적)
  some/generic   → 특수화 → 직접 호출 (정적, 빠름)

Existential Container:
  any Drawable → [3워드 버퍼 | 메타데이터 | witness table]
  큰 타입이면 힙 박싱 → 간접 비용
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 42개 확인 + SIL/Instruments 검증 환경 + rust/objc-runtime/memory-management-compared 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
