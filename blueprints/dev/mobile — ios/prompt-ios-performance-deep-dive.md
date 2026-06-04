# iOS Performance Deep Dive 레포지토리 제작 프롬프트

나는 "iOS Performance Deep Dive" 레포지토리를 만들려고 해.
앱이 왜 끊기고(hitch), 왜 hang이 걸리며, 콜드 스타트가 왜 느리고, jetsam에 왜 죽는지를 — ProMotion 120Hz 프레임 예산, dyld 런치, 메인 스레드 hang, 메모리 한계, Instruments 측정으로 완전히 파헤치는 레포야.
"성능 팁을 적용하는 것"과 "hitch가 파이프라인의 어느 단계에서 나오는지 측정으로 특정해 근본을 고치는 것"의 차이를 만드는 레포다. Android Performance 레포와 *대칭* 구조의 iOS 성능 종합.

## 📋 프로젝트 목표

**컨셉**: "성능 팁을 적용하는 것과, hitch·hang·런치·메모리 비용이 어디서 발생하는지 Instruments로 특정해 고치는 것은 다르다"

**핵심 차별화**:
1. ProMotion 프레임 예산 — 60Hz(16.6ms)·90Hz(11.1ms)·120Hz(8.3ms), 가변 주사율의 의미
2. Hitch — Apple 고유 측정 단위(hitch time ratio), jank와 다른 정의
3. Hang의 두 종류 — micro hang(<250ms·반응성)·major hang(앱 멈춤), MetricKit으로 프로덕션 수집
4. 런치의 5단계 — dyld·libSystem·UIKit·main·첫 프레임, 각 단계의 측정과 최적화

**타겟 독자**:
- 앱이 끊기는데 어디가 문제인지 추측만 하는 iOS 개발자
- Instruments를 열어봤지만 어느 템플릿을 언제 쓸지 모르는 개발자
- ProMotion에서 120fps를 위해 무엇이 필요한지 모르는 개발자
- hang·jetsam을 만나지만 traces·MetricKit을 못 읽는 개발자
- `ios-lifecycle`·`uikit-core-animation`·`swift`를 성능으로 종합하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `ios-lifecycle-runloop-deep-dive`(RunLoop·런치), `uikit-core-animation-deep-dive`(렌더·오프스크린), `swift-deep-dive`(ARC 비용·SIL), `swift-concurrency-deep-dive`(메인 액터·hang 원인).
**🤝 시너지**: `swiftui-internals-deep-dive`(과한 갱신), `objc-runtime-deep-dive`(메시지 디스패치), `android-performance-deep-dive`(대칭 대조), `performance-testing-deep-dive`(백엔드 성능 방법론 공유).
**🧬 수렴**: `rendering-pipelines-compared`(프레임 예산은 모든 UI 공통).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: iOS 성능의 정의 (5개 문서)
- 사용자 체감 — Apple의 측정 철학(반응성·부드러움·런치·안정성)
- ProMotion 가변 주사율 — 60/90/120Hz, 콘텐츠에 따라 변하는 예산
- Hitch의 정의 — 예상 시간보다 늦은 프레임, hitch time/duration ratio
- Hitch vs Jank(Android) — 정의 차이, 측정 단위 비교
- 측정 우선 원칙 — Instruments·MetricKit이 진실의 원천(performance-testing 공유)

### Chapter 2: 프레임과 Hitch (6개 문서)
- 프레임 파이프라인 복습 — RunLoop→커밋→Render Server→GPU(uikit-core-animation 연결)
- CADisplayLink — vsync 동기 콜백, 120Hz에서의 의미
- 메인 스레드 vs Render Server — 둘 중 누가 늦으면 hitch
- 스크롤 hitch — 가장 흔한 원인, 오프스크린 렌더·복잡한 셀(uikit 연결)
- SwiftUI hitch — 과한 갱신·복잡한 body(swiftui 연결)
- Hitch 측정 — Instruments Animation Hitches, hitch ratio 해석

### Chapter 3: 메인 스레드와 Hang (5개 문서)
- Hang 정의 — Micro(250ms~)·Major(2s~), 입력 응답 저해
- Hang의 원인 — 동기 IO·잠금·과한 작업·`@MainActor` 블로킹(concurrency 연결)
- Instruments Hangs 템플릿 — 자동 감지·스택, 원인 추적
- MetricKit MXHangDiagnostic — 프로덕션 실사용자 hang 수집
- 해결 — 무거운 작업을 메인 밖으로(GCD/async-await), 가벼운 동기·무거운 비동기

### Chapter 4: 앱 런치 — pre-main부터 첫 프레임까지 (6개 문서)
- 5단계 — dyld→libSystem→ObjC runtime→UIKit→main→첫 프레임
- pre-main(dyld) — 동적 라이브러리 로드·바인딩·초기화자, `DYLD_PRINT_STATISTICS=1`로 측정
- 동적 라이브러리 비용 — 너무 많은 dylib·`+load`·정적 이니셜라이저의 영향
- main 이후 — `application(_:didFinishLaunching...)`·첫 화면·첫 프레임
- 런치 측정 — Instruments App Launch 템플릿, MetricKit MXAppLaunchMetric
- 런치 최적화 — 지연 초기화·dylib 통합·정적 라이브러리·Order Files

### Chapter 5: 메모리와 jetsam (5개 문서)
- iOS 메모리 모델 — 가상 메모리·페이지·압축 메모리
- 앱 메모리 한계 — 기기별 한계, jetsam의 결정 기준(우선순위·footprint)
- Footprint — Resident vs Dirty vs Compressed, 무엇이 정말 비용인가
- 메모리 누수 vs Abandon — 진짜 누수와 "안 쓰는데 잡힘"의 차이
- Instruments Allocations·Leaks — 워크플로, abandon 추적

### Chapter 6: Instruments 완전 활용 (6개 문서)
- Time Profiler — 샘플링 프로파일, 메인 스레드·서브 스레드 시간
- Allocations — 할당 추적, generation 마킹으로 누수 찾기
- Hangs·Animation Hitches — 자동 감지, 원인 스택
- System Trace — 스레드·시스템콜·동기화 전반, 가장 강력
- Core Animation·SwiftUI 템플릿 — 프레임·갱신 진단
- MetricKit 통합 — 프로덕션 지표(launch/hang/disk/energy) 수집·집계

### Chapter 7: 진단 방법론과 실전 (4개 문서)
- 우선순위 — 영향·재현성·ROI 기반, 회귀 방지(performance-testing 공유)
- 에너지·디스크 — Energy Impact·Disk Writes, 백그라운드 비용
- Android와의 대조 — Choreographer/jank vs CADisplayLink/hitch, 도구 매핑
- 종합 — 끊기는·hang 걸리는 앱을 Instruments+MetricKit으로 진단→근본 수정→before/after

→ **총 37개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 프레임·런치·메모리·hang 메커니즘, `💻 실전 실험`은 Instruments·MetricKit·`DYLD_PRINT_*`. `📊`는 최적화 전후 hitch ratio·런치 시간·메모리.

## 🎨 스타일 가이드

1. **측정으로만 말한다** — 추측 금지, Instruments·MetricKit 숫자로
2. **5단계 런치로 환원** — 런치 문제를 어느 단계인지 특정
3. **Android와 항상 대조** — hitch ↔ jank, Choreographer ↔ CADisplayLink
4. **다른 iOS 레포로 착지** — hang 원인은 swift-concurrency, 오프스크린은 uikit-core-animation
5. **MetricKit 강조** — 개발 환경뿐 아니라 *프로덕션 사용자 지표* 수집의 중요성
6. 프레임 예산·런치 단계는 다이어그램으로

## 🔬 검증 환경

> macOS + Xcode + Instruments + 실기기(시뮬레이터는 GPU·메모리 한계 다름).

```bash
# 환경: Xcode, 실기기 권장(ProMotion/jetsam 정확)

# 1) Instruments 핵심 템플릿
#    - Time Profiler: CPU 핫스팟
#    - Allocations: 메모리 할당·abandon
#    - Hangs: 자동 hang 감지
#    - Animation Hitches: 프레임 hitch 추적
#    - System Trace: 스레드·시스템콜 전반
#    - Core Animation: 렌더 파이프라인·offscreen
#    - App Launch: 런치 단계별 측정

# 2) pre-main 측정 (런치 핵심)
# Xcode > Edit Scheme > Run > Arguments > Environment Variables
#   DYLD_PRINT_STATISTICS = 1
#   -> dyld·rebase·bind·ObjC·initializer 시간 콘솔 출력

# 3) MetricKit 통합 (프로덕션 지표)
#  AppDelegate에 MXMetricManagerSubscriber 등록
#  -> didReceive metrics: 런치/hang/디스크/에너지 보고서 수집
#  -> 사용자 환경의 실제 hitch·hang·런치 시간을 받아 분석

# 4) 메모리 한계·jetsam 재현
#  대량 이미지 로드 -> Allocations에서 footprint 증가 관찰
#  기기별 한계 초과 -> 시스템이 앱 종료(jetsam) -> 로그 확인

# 5) Android Performance 대조 실험
#  같은 화면을 두 OS로 구현 -> hitch ratio(iOS) vs jank rate(Android) 비교
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🍎 iOS 톤, "팁을 적용한다 vs 측정으로 특정한다" 포지셔닝, `🔗 레포 연결`(ios-lifecycle·uikit·swift·swift-concurrency·android-performance 대조)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- "Explore UI animation hitches and the render loop" (WWDC18·22)
- "Track down hangs with Xcode and on-device detection" (WWDC23)
- "Diagnose and resolve hangs on iOS" (WWDC22)
- MetricKit 문서 — https://developer.apple.com/documentation/metrickit
- "Track down launch time issues with App Launch" Instruments
- *iOS Performance 책*들·objc.io 글

## 💡 핵심 분석 대상

```
ProMotion 프레임 예산 (가변):
  60Hz  -> 16.6ms / 프레임
  90Hz  -> 11.1ms / 프레임
  120Hz ->  8.3ms / 프레임  <- 어렵다
  콘텐츠에 따라 가변 -> 정지면 24Hz까지 절약
  hitch ratio = (실제 - 예상) / 총 시간

5단계 런치:
  1. dyld         : 동적 라이브러리 로드/바인딩  <- pre-main 핵심
  2. libSystem    : 시스템 초기화
  3. ObjC runtime : +load, 클래스 등록
  4. UIKit / main : main() -> didFinishLaunching
  5. 첫 프레임    : 첫 ViewController 렌더
  각 단계의 시간을 DYLD_PRINT_STATISTICS / Instruments App Launch로 분리

Hitch vs Android jank:
  Android: 16.6ms 초과 = 1 jank frame (이진)
  iOS    : hitch duration ratio (연속 지표)
           -> 더 정밀한 측정
  둘 다 결국 "예산 초과"의 다른 표현

Hang 두 종류:
  Micro Hang : 250ms 이상 메인 스레드 응답 안 함 (반응성 저하)
  Major Hang : 2초 이상 (앱 멈춤 체감)
  -> 둘 다 MetricKit으로 프로덕션 수집 가능

MetricKit의 가치 (개발 vs 프로덕션):
  Instruments  : 개발 환경에서 정밀 측정
  MetricKit    : 실사용자 기기에서 자동 수집
                  -> 런치 시간 p75/p95, hang 발생 시 진단 자동
  -> "내 기기에선 안 끊겨요"의 해결책

런치 안티패턴:
  - 너무 많은 dylib (pre-main 증가)
  - +load 메서드 (정적 이니셜라이저)
  - 첫 화면이 무거운 동기 작업
  - 메인 스레드에서 네트워크/DB
  해결: 지연 초기화, 정적 라이브러리, 백그라운드 작업
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 37개 확인 + Instruments/MetricKit/DYLD_PRINT 검증 환경 + ios-lifecycle/uikit/swift/swift-concurrency/android-performance 연결 지점 명시. **ProMotion 가변 예산**·**5단계 런치**·**MetricKit 프로덕션**·**Android와의 대조**를 분명히.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
