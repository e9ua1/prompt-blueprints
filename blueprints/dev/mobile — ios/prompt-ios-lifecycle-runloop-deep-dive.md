# iOS Lifecycle & RunLoop Deep Dive 레포지토리 제작 프롬프트

나는 "iOS Lifecycle & RunLoop Deep Dive" 레포지토리를 만들려고 해.
앱이 어떻게 시작되고 상태가 전환되는지, RunLoop가 메인 스레드를 어떻게 돌리며 입력·타이머·렌더가 어떤 순서로 처리되는지, Autorelease Pool이 언제 비워지는지를 완전히 파헤치는 레포야.
"생명주기 메서드를 구현하는 것"과 "RunLoop가 무엇을 언제 처리하고 앱 상태가 누구에 의해 전환되는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "생명주기 콜백을 구현하는 것과, RunLoop가 메인 스레드를 어떻게 돌리고 시스템이 상태를 어떻게 전환하는지 아는 것은 다르다"

**핵심 차별화**:
1. RunLoop의 실체 — 메인 스레드가 종료되지 않고 입력·타이머·소스를 처리하는 루프
2. 앱 생명주기 — 시작·foreground·background·종료가 시스템에 의해 전환되는 흐름
3. Autorelease Pool — objc/Swift 메모리 해제 타이밍, RunLoop와의 결합
4. 렌더 타이밍 — RunLoop 모드·CADisplayLink가 화면 갱신과 맞물리는 방식

**타겟 독자**:
- 생명주기 메서드를 구현하지만 누가 호출하는지 모르는 iOS 개발자
- RunLoop를 단어로만 아는 개발자
- 스크롤 중 타이머가 안 도는 이유(RunLoop 모드)를 모르는 개발자
- Autorelease Pool이 언제 비워지는지 모르는 개발자
- `objc-runtime`·`android-framework`(Looper)와 비교하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `objc-runtime-deep-dive`(런타임 토대), `swift-deep-dive`(ARC·Autorelease).
**🤝 시너지**: `uikit-core-animation-deep-dive`(렌더 타이밍), `swift-concurrency-deep-dive`(메인 액터·RunLoop), `android-framework-internals-deep-dive`(Looper와 대조).
**🧬 수렴**: `concurrency-models-compared`(RunLoop ↔ Android Looper ↔ JS Event Loop — *같은 메시지 루프 패턴*).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 앱 시작과 진입점 (5개 문서)
- main과 진입 — UIApplicationMain, @main, 앱 객체 생성
- 시작 시퀀스 — 프로세스 생성→앱 객체→delegate→첫 화면, 단계별
- AppDelegate vs SceneDelegate — 앱 vs 씬 분리, 멀티 윈도우
- 시작 최적화 — pre-main 시간(dyld·로드), 시작 측정
- 런치 타입 — cold/warm, 시스템이 시작을 관리하는 방식

### Chapter 2: 앱 생명주기 상태 (5개 문서)
- 상태 모델 — Not Running/Inactive/Active/Background/Suspended
- 상태 전환 — 누가 전환시키나(시스템), 각 전환 콜백
- foreground/background — 진입·이탈 시 해야 할 일, 시간 제한
- 백그라운드 실행 — 백그라운드 태스크·갱신, 제한과 종료
- 종료·복원 — 시스템 종료, 상태 복원(state restoration)

### Chapter 3: RunLoop 완전 분해 (6개 문서)
- RunLoop란 — 스레드가 살아있게 하는 이벤트 처리 루프
- 메인 RunLoop — UIApplicationMain이 시작, 앱이 안 끝나는 이유
- 입력 소스·타이머 — source0/source1·타이머, RunLoop가 처리하는 것
- RunLoop 모드 — default/tracking/common, 스크롤 중 모드 전환
- 모드와 타이머 함정 — 스크롤 중 타이머가 멈추는 이유, common mode
- RunLoop 한 사이클 — 깨우기·소스 처리·타이머·sleep, 내부 단계

### Chapter 4: 스레드와 RunLoop (5개 문서)
- 스레드별 RunLoop — 메인은 자동, 서브 스레드는 수동 시작
- 서브 스레드 RunLoop — 언제 필요, 직접 run, 종료
- RunLoop와 GCD — 디스패치 큐와의 관계·차이
- 메인 스레드 마샬링 — UI는 메인에서, 다른 스레드→메인 전환
- 좀비 스레드·블로킹 — RunLoop 없는 스레드, 블로킹 영향

### Chapter 5: Autorelease Pool (5개 문서)
- Autorelease란 — 즉시 release가 아닌 풀에 등록, 나중에 해제
- 풀과 RunLoop — 메인 RunLoop가 매 사이클 풀을 비우는 결합
- @autoreleasepool — 명시적 풀, 루프에서 메모리 급증 방지
- Swift에서의 의미 — Swift ARC와 autorelease의 관계, objc 상호운용
- 함정 — 큰 루프의 메모리 피크, 풀 누락

### Chapter 6: 렌더와 타이밍 (4개 문서)
- CADisplayLink — 화면 주사율 동기 콜백, RunLoop에 등록(uikit 연결)
- 렌더 타이밍 — RunLoop·트랜잭션 커밋·렌더 서버(uikit-core-animation 연결)
- 타이머 정확도 — RunLoop 타이머의 한계, 정밀 타이밍 대안
- 메인 스레드 예산 — RunLoop 한 사이클이 길면 끊김·응답성 저하

### Chapter 7: 디버깅과 실전 (4개 문서)
- RunLoop 관찰 — CFRunLoopObserver로 단계 관찰, 현재 모드 확인
- 행(hang) 진단 — 메인 스레드 블로킹·긴 사이클, Instruments Hang
- 생명주기 디버깅 — 상태 전환 로깅, 백그라운드 처리 검증
- 종합 — 시작→전환→RunLoop 처리를 추적해 전체 흐름 해부

→ **총 34개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 RunLoop 사이클·상태 전환 흐름, `💻 실전 실험`은 RunLoopObserver·Instruments·로깅. `📊`는 RunLoop 모드·풀 유무의 메모리/타이밍.

## 🎨 스타일 가이드

1. **"누가·언제"에 답한다** — 생명주기·렌더를 시스템·RunLoop 주도로
2. **RunLoop를 관찰** — CFRunLoopObserver로 단계를 *직접 본다*
3. **Android Looper와 대조** — 같은 메시지 루프, 다른 구현
4. **모드 함정 강조** — 스크롤 중 타이머 정지를 RunLoop 모드로 설명
5. RunLoop 사이클·상태 머신은 다이어그램으로

## 🔬 검증 환경

> macOS + Xcode. RunLoop·Instruments로 관찰.

```bash
# 환경: macOS, Xcode, 시뮬레이터/기기

# RunLoop 단계 관찰
#  CFRunLoopObserver로 각 활동(beforeTimers/beforeSources/beforeWaiting/...) 로깅
#  → 한 사이클에 무엇이 어느 순서로 처리되는지 확인

# RunLoop 모드 함정 실험:
#  Timer를 default 모드로 등록 → 스크롤(tracking 모드) 중 멈춤 관찰
#  → .common 모드로 등록 → 스크롤 중에도 동작

# Autorelease 실험:
#  큰 루프에서 객체 생성 → 메모리 급증
#  @autoreleasepool로 감싸기 → 피크 감소(Instruments Allocations)

# 생명주기 로깅: 각 콜백·상태 전환 로그, 백그라운드 진입/복귀
# Instruments: Time Profiler(메인 스레드), Hangs(행 감지)
# 시작 시간: pre-main은 DYLD_PRINT_STATISTICS=1
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🍎 iOS 톤, "콜백을 구현한다 vs RunLoop·시스템이 무엇을 언제 하나" 포지셔닝, `🔗 레포 연결`(objc-runtime·uikit·concurrency-models-compared)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- "Threading Programming Guide: Run Loops" (Apple)
- App 생명주기 문서 — https://developer.apple.com/documentation/uikit/app_and_environment
- CFRunLoop 소스(opensource.apple.com)
- "이벤트 처리·RunLoop" WWDC 발표
- Mike Ash "Friday Q&A: RunLoop"

## 💡 핵심 분석 대상

```
RunLoop (메인 스레드가 안 끝나는 이유):
  UIApplicationMain {
    RunLoop.main.run()   // 무한 루프
  }
  한 사이클:
    깨우기 → 타이머 처리 → 입력 소스(터치 등) → (렌더 트랜잭션 커밋)
    → 처리할 것 없으면 sleep(CPU 안 씀) → 이벤트 오면 깨움
  → 이벤트 기반으로 돌며 앱을 살아있게

앱 상태 전환 (시스템 주도):
  Not Running → (실행) → Inactive → Active
  Active → (홈 버튼) → Inactive → Background → Suspended
  → "내가" 전환 X, 시스템이 전환하고 콜백 호출

RunLoop 모드 함정:
  Timer를 .default 모드 등록
  → 스크롤 시작하면 RunLoop가 .tracking 모드로 전환
  → .default 타이머는 안 돔(멈춤!)
  → .common 모드로 등록해야 스크롤 중에도 동작

Autorelease Pool + RunLoop:
  autorelease된 객체 → 풀에 등록
  메인 RunLoop가 매 사이클 끝에 풀 drain → 해제
  큰 루프(한 사이클 내)면 풀이 안 비워져 메모리 급증
  → @autoreleasepool { } 로 중간중간 해제
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 34개 확인 + RunLoopObserver/Instruments 검증 환경 + objc-runtime/uikit/concurrency-models-compared 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
