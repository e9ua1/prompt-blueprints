# Kotlin Multiplatform (KMP) Deep Dive 레포지토리 제작 프롬프트

나는 "Kotlin Multiplatform (KMP) Deep Dive" 레포지토리를 만들려고 해.
한 Kotlin 코드가 어떻게 JVM·Android·iOS(Native)·JS·WASM 여러 타깃으로 컴파일되는지 — expect/actual, 공유 로직 컴파일, Kotlin/Native 메모리 모델, 플랫폼 인터롭을 완전히 파헤치는 레포야.
"KMP로 코드를 공유하는 것"과 "한 소스가 어떻게 여러 백엔드로 컴파일되고 Native 메모리 모델이 무엇을 보장하는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "KMP로 코드를 공유하는 것과, 한 Kotlin 소스가 여러 타깃 백엔드로 어떻게 컴파일되고 플랫폼과 어떻게 상호작용하는지 아는 것은 다르다"

**핵심 차별화**:
1. 멀티 백엔드 컴파일 — 같은 IR이 JVM 바이트코드·LLVM(Native)·JS로 갈라지는 원리
2. expect/actual — 공통 선언과 플랫폼별 구현의 분리, 컴파일 타임 결합
3. Kotlin/Native 메모리 모델 — 옛 동결(freezing) 모델에서 새 모델로, ARC 상호작용
4. 인터롭 — Swift/ObjC·JS와의 양방향 상호운용, 경계 비용

**타겟 독자**:
- KMP를 쓰지만 컴파일이 어떻게 갈라지는지 모르는 개발자
- expect/actual을 외워 쓰는 개발자
- Kotlin/Native 메모리 모델·iOS 상호운용을 모르는 개발자
- "어디까지 공유할까" 경계를 못 정하는 개발자
- `kotlin-deep-dive`·`compiler`·`swift`를 멀티플랫폼으로 연결하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `kotlin-deep-dive`(언어·코루틴·컴파일), `compiler-deep-dive`(IR·멀티 타깃 코드젠).
**🤝 시너지**: `swift-deep-dive`(iOS 인터롭·ARC 상호작용), `android-architecture-deep-dive`(공유 데이터/도메인 레이어), `web-apis-wasm-deep-dive`(JS/WASM 타깃).
**🧬 수렴**: `compiler-deep-dive`의 멀티 백엔드 실전 사례. "한 소스, 여러 런타임".

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: KMP 개요와 컴파일 모델 (5개 문서)
- KMP란 — 로직 공유 + 플랫폼 UI, "공유 가능한 것만 공유" 철학
- 타깃 — JVM/Android/iOS(Native)/JS/WASM, 무엇을 공유하나
- 컴파일 백엔드 — 공통 프론트엔드(IR)→타깃별 백엔드(JVM/LLVM/JS)(compiler 연결)
- 소스셋 계층 — common·플랫폼별·중간 소스셋, 의존 구조
- 무엇이 공유 가능한가 — 순수 로직·도메인은 쉽고, 플랫폼 API는 expect/actual

### Chapter 2: expect/actual (5개 문서)
- expect 선언 — 공통 코드의 "약속", 시그니처만
- actual 구현 — 플랫폼별 실제 구현, 컴파일 타임 결합
- 무엇에 쓰나 — 플랫폼 API(파일·시간·난수), 점진적 공유
- 대안 — 인터페이스 + 플랫폼 주입 vs expect/actual, 선택 기준
- 함정 — 과한 expect/actual, 경계 설계 실수

### Chapter 3: Kotlin/Native (6개 문서)
- Kotlin/Native — LLVM 기반 AOT 컴파일(compiler 연결), 네이티브 바이너리
- 메모리 관리 — Kotlin/Native GC, Swift ARC와의 상호작용(swift 연결)
- 옛 동결 모델 — 과거 freezing·스레드 제약의 역사와 폐지
- 새 메모리 모델 — 현재 모델, 코루틴·동시성 제약 완화
- 상호운용 표현 — Kotlin 타입이 ObjC/Swift로 어떻게 노출되나
- 성능·바이너리 — 크기·시작, 네이티브 컴파일 트레이드오프

### Chapter 4: 플랫폼 인터롭 (6개 문서)
- iOS 인터롭 — Kotlin→ObjC 헤더 생성, Swift에서 사용, 네이밍 매핑
- 인터롭 제약 — Kotlin 기능 중 ObjC로 안 가는 것(제네릭·sealed 등)
- 코루틴 ↔ Swift async — suspend 함수를 iOS에서 호출, async/Combine 브리징
- Android/JVM 인터롭 — 자연스러운 Java 상호운용
- JS/WASM 인터롭 — JS 호출, 타입 매핑, web-apis 연결
- C 인터롭 — cinterop으로 C 라이브러리 사용

### Chapter 5: 공유 아키텍처 (5개 문서)
- 공유 레이어 설계 — 도메인·데이터를 공유, UI는 플랫폼별(android-architecture 연결)
- Repository 공유 — 네트워크(Ktor)·DB(SQLDelight) 공유
- 공유 ViewModel — KMP-ViewModel·상태 공유, 플랫폼 바인딩
- 의존성 주입 — Koin 등 멀티플랫폼 DI
- 경계 결정 — 공유 vs 플랫폼, ROI 기반 판단

### Chapter 6: 생태계와 빌드 (5개 문서)
- 멀티플랫폼 라이브러리 — Ktor·SQLDelight·kotlinx.serialization·coroutines
- Gradle 멀티플랫폼 — 소스셋·타깃 설정, 빌드 구조
- iOS 통합 — XCFramework·CocoaPods·SPM 배포
- Compose Multiplatform — UI까지 공유(선택), 데스크톱·웹
- 빌드·CI — 멀티 타깃 빌드, 캐시

### Chapter 7: 측정과 실전 (4개 문서)
- 디버깅 — 공유 코드 디버깅(Android Studio·Xcode), 크로스 플랫폼
- 성능 — 공유 로직의 플랫폼별 성능, 인터롭 오버헤드
- 마이그레이션 — 기존 앱에 KMP 점진 도입
- 종합 — 공유 도메인 + 네이티브 UI 앱(Android+iOS)을 끝까지 구현

→ **총 34개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 컴파일 백엔드·인터롭 표현·메모리 모델, `💻 실전 실험`은 멀티 타깃 빌드·ObjC 헤더 확인·인터롭. `📊`는 공유율·인터롭 비용·바이너리 비교.

## 🎨 스타일 가이드

1. **컴파일 갈래로 환원** — 공유 코드가 어떤 백엔드로 가는지(compiler 연결)
2. **"공유 가능 vs 아님"** — 모든 결정을 공유 경계로
3. **swift/kotlin 레포로 착지** — 인터롭은 swift, 언어는 kotlin
4. **인터롭 비용 명시** — 경계가 공짜가 아님을 측정으로
5. 소스셋 계층·컴파일 백엔드는 다이어그램으로

## 🔬 검증 환경

> Kotlin Multiplatform + Android Studio + Xcode.

```bash
# KMP 프로젝트 (Android Studio KMP 템플릿)
# 멀티 타깃 빌드
./gradlew build                        # 모든 타깃
./gradlew :shared:assembleXCFramework  # iOS용 프레임워크

# 검증 방법
# 1) 컴파일 갈래: 같은 common 코드가
#    → JVM 바이트코드(javap), Native(LLVM), JS로 나오는지 확인
# 2) expect/actual: common의 expect → 각 플랫폼 actual 결합 확인
# 3) iOS 인터롭: 생성된 ObjC 헤더(.h) 열어 Kotlin 타입 매핑 확인
#    suspend 함수가 completion/async로 노출되는지
# 4) 메모리: Kotlin/Native 객체가 Swift ARC와 상호작용하는지(Instruments)
# 5) 공유율 측정: common vs 플랫폼별 코드 라인 비율
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🔀 Cross-Platform 톤, "코드를 공유한다 vs 어떻게 여러 타깃으로 컴파일되나" 포지셔닝, `🔗 레포 연결`(kotlin·compiler·swift)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- KMP 공식 문서 — https://www.jetbrains.com/kotlin-multiplatform/
- Kotlin/Native 메모리 모델 문서 — https://kotlinlang.org/docs/native-memory-manager.html
- "KMP under the hood" 발표들
- Ktor·SQLDelight·Compose Multiplatform 문서
- Touchlab 블로그(KMP iOS 실전)

## 💡 핵심 분석 대상

```
멀티 백엔드 컴파일 (한 소스 → 여러 타깃):
  common Kotlin
       │ 공통 프론트엔드(파싱·타입·IR) (compiler 레포)
       ├─► JVM 백엔드  → .class 바이트코드 (Android)
       ├─► Native 백엔드(LLVM) → 네이티브 바이너리 (iOS)
       ├─► JS 백엔드   → JavaScript
       └─► WASM 백엔드 → WebAssembly

expect/actual:
  // commonMain
  expect fun platformName(): String
  // androidMain
  actual fun platformName() = "Android"
  // iosMain
  actual fun platformName() = "iOS"
  → 컴파일 타임에 타깃별 actual 결합

iOS 인터롭:
  Kotlin suspend fun → ObjC: completion handler 또는 Swift async
  Kotlin sealed/제네릭 → ObjC로 일부만 노출(제약)
  Kotlin/Native 객체 ↔ Swift ARC: 참조 카운팅 상호작용

공유 경계 (현실적):
  공유 쉬움: 도메인 로직·네트워크·DB·검증
  공유 어려움/안 함: UI(네이티브가 나음)·플랫폼 특화 API
  → "로직 공유, UI 네이티브"가 일반적 선택
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 34개 확인 + 멀티타깃빌드/ObjC헤더 검증 환경 + kotlin/compiler/swift 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
