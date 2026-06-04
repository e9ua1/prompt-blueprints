# Android Performance Deep Dive 레포지토리 제작 프롬프트

나는 "Android Performance Deep Dive" 레포지토리를 만들려고 해.
프레임이 왜 끊기고(jank), ANR이 왜 나며, 콜드 스타트가 왜 느린지를 — Choreographer·vsync·메인 스레드 예산·메모리 압박 관점에서 완전히 파헤치는 레포야.
"성능 팁을 적용하는 것"과 "프레임이 16.6ms 예산의 어디서 깨지는지 측정으로 특정하고 근본을 고치는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "성능 팁을 적용하는 것과, 프레임·시작·메모리 비용이 어디서 발생하는지 측정으로 특정해 고치는 것은 다르다"

**핵심 차별화**:
1. 프레임 예산 — Choreographer·vsync가 정한 16.6ms 안에서 무엇이 jank를 만드나
2. 메인 스레드의 신성함 — UI·입력·draw가 한 스레드에서 경쟁, 블로킹이 ANR로
3. 측정이 먼저 — 추측 금지, Perfetto·Macrobenchmark로 병목을 *데이터로* 특정
4. 콜드 스타트·메모리 — 시작 단계·GC 압박·누수가 체감 성능을 결정

**타겟 독자**:
- 앱이 끊기는데 어디가 문제인지 추측만 하는 개발자
- ANR을 만나지만 traces를 못 읽는 개발자
- "성능 최적화 팁"을 측정 없이 적용하는 개발자
- jank·GC·콜드 스타트를 단어로만 아는 개발자
- `android-runtime`·`framework`·`compose`를 성능으로 종합하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `android-framework-internals-deep-dive`(메인 스레드·Choreographer·ANR), `android-runtime-deep-dive`(GC·시작), `computer-architecture-deep-dive`(캐시·메모리).
**🤝 시너지**: `jetpack-compose-internals-deep-dive`(리컴포지션 jank), `gpu-graphics-deep-dive`(렌더 단계), `performance-testing-deep-dive`(백엔드 성능 방법론 공유).
**🧬 수렴**: `rendering-pipelines-compared`(프레임 예산은 모든 UI 플랫폼 공통).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 성능의 정의와 프레임 (5개 문서)
- 사용자 체감 성능 — 부드러움·응답성·시작·안정성, 무엇을 측정하나
- vsync와 Choreographer — 디스플레이 주사율, 프레임 콜백, 16.6ms(60Hz)/8.3ms(120Hz)
- 프레임 생명주기 — 입력→애니메이션→측정/레이아웃/그리기→GPU, 한 프레임의 일
- jank의 정의 — 프레임 예산 초과 = 드롭, 인지되는 끊김
- 측정 우선 원칙 — 추측하지 말고 Perfetto/Profiler로(performance-testing 공유)

### Chapter 2: 렌더링 성능과 jank (6개 문서)
- 메인 스레드 vs RenderThread — UI 준비(메인)와 GPU 제출(Render)의 분업
- 오버드로 — 같은 픽셀 여러 번 그리기, 시각화·줄이기(gpu 연결)
- 레이아웃 비용 — 깊은/복잡한 뷰 계층, 측정/레이아웃 패스 비용
- Compose jank — 과한 리컴포지션·불안정 파라미터(compose 연결)
- 리스트 성능 — RecyclerView/LazyColumn, 뷰 재활용·아이템 비용
- 애니메이션 — 메인 스레드 애니메이션의 위험, 프레임 정렬

### Chapter 3: ANR과 메인 스레드 (5개 문서)
- ANR이란 — 입력 5초·서비스 등 임계값, 메인 스레드 블로킹이 원인
- ANR 메커니즘 — 메시지 큐 적체→입력 처리 지연(framework 연결)
- 흔한 원인 — 메인 스레드 IO·DB·네트워크·과한 작업, 락 경합
- traces 분석 — ANR 덤프에서 메인 스레드 스택 읽기, 원인 특정
- 해결 — 무거운 작업 오프로딩(코루틴 Dispatcher), 메인 스레드 보호

### Chapter 4: 앱 시작 성능 (5개 문서)
- 시작 단계 — 프로세스 생성→Application→첫 프레임(runtime·framework 연결)
- 콜드/웜/핫 — 차이·측정, 무엇이 콜드를 지배하나
- 시작 최적화 — App Startup·초기화 지연·불필요 작업 제거
- Baseline Profile — 시작 경로 사전 컴파일(runtime 연결), 효과 측정
- 스플래시·인지 — 인지 시작 시간 단축, 빈 화면 방지

### Chapter 5: 메모리와 배터리 (6개 문서)
- 메모리 압박 — 힙 제한·Low Memory Killer, OOM
- GC와 jank — 할당 폭주가 GC를 부르고 프레임을 놓침(runtime 연결)
- 할당 줄이기 — 객체 풀·재사용·불필요 박싱 제거, 할당 추적
- 메모리 누수 — Context·리스너·핸들러 누수, LeakCanary로 추적
- 배터리 — wakelock·잦은 깨움·백그라운드 작업, Doze 대응
- 비트맵·리소스 — 이미지 메모리, 다운샘플링·캐시

### Chapter 6: 측정 도구 (5개 문서)
- Android Studio Profiler — CPU·메모리·에너지·네트워크 실시간
- Perfetto/systrace — 시스템 전반 트레이스, 프레임·메인 스레드·GC 트랙
- Macrobenchmark — 시작·jank를 재현 가능하게 측정(CI 통합)
- Microbenchmark — 코드 조각 벤치, JIT/캐시 함정 주의
- Frame metrics — JankStats·FrameTimingMetric으로 프레임 추적

### Chapter 7: 진단 방법론 (4개 문서)
- 병목 특정 — 증상→트레이스→원인, USE 방법론(performance-testing 공유)
- 우선순위 — 영향 큰 것부터, ROI 기반 최적화
- 회귀 방지 — 성능 예산·CI 벤치마크·알림
- 종합 — 끊기는 앱을 Perfetto로 진단→근본 수정→Macrobenchmark before/after

→ **총 36개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 Choreographer·프레임·GC 메커니즘, `💻 실전 실험`은 Perfetto·Profiler·Macrobenchmark. `📊`는 최적화 전후 프레임 시간·시작·메모리.

## 🎨 스타일 가이드

1. **측정으로만 말한다** — 모든 주장을 트레이스·벤치 숫자로(추측 금지)
2. **프레임 예산으로 환원** — jank를 "예산의 어느 단계 초과"로
3. **다른 안드로이드 레포로 착지** — GC→runtime, ANR→framework, 리컴포지션→compose
4. **백엔드 성능과 공유** — USE 방법론·병목 사고는 performance-testing과 동일
5. 프레임 생명주기·시작 단계는 다이어그램으로

## 🔬 검증 환경

> docker 아님. **Profiler + Perfetto + Macrobenchmark**가 검증 도구.

```bash
# 환경: Android Studio, 실기기 권장(에뮬은 GPU 다름)

# 1) Profiler: CPU·메모리·할당 실시간, 메서드 트레이스
# 2) Perfetto: ui.perfetto.dev — system trace 캡처
#    'Expected/Actual Timeline' 트랙으로 jank 프레임 식별
#    메인 스레드·RenderThread·GC 트랙 분석
adb shell perfetto -o /data/misc/perfetto-traces/trace -t 10s sched freq idle am wm gfx view

# 3) Macrobenchmark (벤치 모듈)
#    StartupBenchmark: cold/warm/hot
#    FrameTimingMetric: jank 프레임 비율
# 4) JankStats 라이브러리: 프로덕션 jank 수집
# 5) GPU 오버드로: 개발자 옵션 > "Debug GPU overdraw"
# 6) StrictMode: 메인 스레드 디스크/네트워크 위반 탐지
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🤖 Android 톤, "팁을 적용한다 vs 측정으로 특정한다" 포지셔닝, `🔗 레포 연결`(framework·runtime·compose)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- App 성능 가이드 — https://developer.android.com/topic/performance
- Perfetto 문서 — https://perfetto.dev/docs/
- Macrobenchmark — https://developer.android.com/topic/performance/benchmarking/macrobenchmark-overview
- "Android Performance Patterns" (구글 시리즈)
- JankStats·Frame metrics 문서

## 💡 핵심 분석 대상

```
프레임 예산 (60Hz = 16.6ms):
  vsync ──► Choreographer 콜백
            [입력] [애니메이션] [측정/레이아웃/그리기] (메인 스레드)
            → [RenderThread → GPU]
  이 합이 16.6ms 초과 → 프레임 드롭(jank)
  120Hz면 예산 8.3ms로 더 빡빡

ANR (메인 스레드 블로킹):
  메인 스레드에서 DB 쿼리 5초
  → 입력 메시지가 큐에서 처리 안 됨
  → 5초 경과 → ANR
  해결: withContext(Dispatchers.IO) { db }

GC와 jank:
  프레임마다 객체 대량 할당 → 힙 압박 → GC 발동
  → GC STW(짧지만) + 할당 비용 → 프레임 예산 초과
  → 할당 줄이기(풀·재사용)

콜드 스타트:
  프로세스 fork(Zygote) → Application.onCreate → 첫 Activity → 첫 프레임
  Application.onCreate에서 무거운 초기화 → 시작 지연
  → 지연 초기화 + Baseline Profile

측정 우선:
  "느린 것 같다" ✗ → Perfetto로 "이 프레임의 측정 패스가 12ms" ✓
  → 데이터로 병목 특정 후 그것만 고침
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 36개 확인 + Perfetto/Profiler/Macrobenchmark 검증 환경 + framework/runtime/compose/performance-testing 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
