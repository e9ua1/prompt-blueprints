# Jetpack Compose Internals Deep Dive 레포지토리 제작 프롬프트

나는 "Jetpack Compose Internals Deep Dive" 레포지토리를 만들려고 해.
선언적 UI가 어떻게 효율적인 화면 갱신이 되는지 — Composer·Slot Table(Gap Buffer)·Recomposition·Snapshot State(MVCC 유사)·측정/배치/그리기를 완전히 파헤치는 레포야.
"@Composable을 작성하는 것"과 "Recomposition이 왜·어디서 일어나고 Snapshot State가 어떻게 일관성을 보장하는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "@Composable을 짜는 것과, Composer가 Slot Table에 무엇을 기록하고 Recomposition을 어떻게 좁히는지 아는 것은 다르다"

**핵심 차별화**:
1. Composer & Slot Table — Composable 호출이 Gap Buffer에 기록되고 위치로 기억(Positional Memoization)되는 원리
2. Recomposition — 무엇이 변하면 어디까지 다시 실행되나, 스킵의 조건
3. Snapshot State — 가변 상태가 MVCC 유사 스냅샷으로 일관성·동시성을 보장(InnoDB MVCC 동형성!)
4. Compose 컴파일러 플러그인 — @Composable 함수에 주입되는 것들($composer·키·skip)

**타겟 독자**:
- Compose를 쓰지만 "왜 리컴포지션되는지" 정확히 모르는 개발자
- remember·key·derivedStateOf를 외워 쓰는 개발자
- Snapshot State가 어떻게 동작하는지 모르는 개발자
- 불필요한 리컴포지션으로 jank를 겪는 개발자
- `react-internals`(Fiber)와 비교하며 선언적 UI를 깊이 보려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `kotlin-deep-dive`(코루틴·컴파일러 플러그인·람다), `android-framework-internals-deep-dive`(Choreographer·메인 스레드).
**🤝 시너지**: `react-internals-deep-dive`(Fiber와 대조), `gpu-graphics-deep-dive`(Skia로 그리기), `android-performance-deep-dive`(jank).
**🧬 수렴**: `reactivity-state-compared`(Snapshot State ↔ Signal·SwiftUI AttributeGraph ↔ **InnoDB MVCC**), `rendering-pipelines-compared`.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 선언적 UI와 Composable (5개 문서)
- 명령형 View → 선언적 Compose — findViewById의 문제, UI = f(state)
- @Composable의 의미 — 함수가 UI를 *방출(emit)*, 반환이 아닌 부수효과 등록
- Composition — Composable 트리의 표현, View 계층과의 차이
- 컴포지션 vs 레이아웃 vs 드로우 — 3단계 분리(react·browser와 대조)
- 첫 컴포지션 vs 리컴포지션 — 최초 구성과 갱신의 차이

### Chapter 2: Composer와 Slot Table (7개 문서)
- Composer 역할 — Composable 실행 중 상태를 기록하는 "받아쓰는 자"
- Slot Table — Gap Buffer 자료구조, 왜 선형 구조로 트리를 표현하나
- Positional Memoization — 호출 *위치*로 상태를 기억, 같은 위치=같은 슬롯
- group과 키 — 그룹으로 구조 추적, 조건/반복 시 키의 역할
- remember의 실체 — Slot Table에 값 저장, 위치 기반 재사용
- 구조적 변경 — if/for로 구조가 바뀔 때 Slot Table 업데이트
- movableContentOf·key — 위치가 바뀌어도 상태 유지(react의 key와 비교)

### Chapter 3: Recomposition (7개 문서)
- Recomposition이란 — 상태 변경 시 영향받은 Composable만 재실행
- 무효화(Invalidation) — 어떤 State를 읽은 Composable이 그 State 변경 시 무효화
- 스킵(Skip) — 입력이 안 변한 Composable 건너뛰기, 안정성(stability) 조건
- 안정성(Stable/Unstable) — 안정 타입의 조건, 불안정이 스킵을 막는 이유
- 리컴포지션 범위 — 어디서 시작해 어디까지, 좁히는 설계
- 자주 하는 실수 — 람다·불안정 파라미터·읽기 위치가 만드는 과한 리컴포지션
- derivedStateOf·remember 키 — 파생 상태로 리컴포지션 줄이기

### Chapter 4: Snapshot State (6개 문서)
- mutableStateOf — 읽기 추적 가능한 상태, 일반 변수와의 차이
- Snapshot 시스템 — 상태의 스냅샷, 격리된 변경, MVCC 유사 구조
- 읽기 추적 — 누가 어떤 State를 읽었나 기록 → 정확한 무효화
- 쓰기와 적용 — 스냅샷에 쓰고 apply, 충돌 처리
- 동시성 — 여러 스냅샷 동시 존재, 일관된 읽기(InnoDB MVCC와 동형 — synthesis 핵심)
- State 종류 — derivedStateOf·produceState·collectAsState(Flow 연동)

### Chapter 5: Layout — Measure / Place / Draw (6개 문서)
- 단일 패스 측정 — Compose 레이아웃이 한 번 측정하는 원리(중첩 제약)
- 제약(Constraints) — 부모→자식 제약 전달, 측정 정책
- 커스텀 Layout — Layout 컴포저블, MeasurePolicy
- 그리기 — Canvas·DrawScope, Skia로 GPU 그리기(gpu 연결)
- Modifier 체인 — Modifier가 측정/그리기에 관여하는 순서, 비용
- 화면 갱신 — Recomposition→Layout→Draw가 프레임에 배치(framework Choreographer 연결)

### Chapter 6: Compose 컴파일러 플러그인 (6개 문서)
- 컴파일러 플러그인 — @Composable 함수 변환, $composer 주입
- 주입되는 것 — composer 인자·그룹 키·changed 비트(스킵 판정)
- 안정성 추론 — 컴파일러가 타입 안정성을 추론·표시
- 컴파일러 리포트 — stability·recomposition 메트릭 생성·해석
- @Stable/@Immutable — 컴파일러에 안정성 약속, 효과
- 디버깅 — 생성 코드 관점에서 리컴포지션 원인 추적

### Chapter 7: 성능과 측정 (5개 문서)
- 리컴포지션 측정 — Layout Inspector 리컴포지션 카운트
- 컴파일러 메트릭 — restartable/skippable/stable 비율로 진단
- 최적화 — 불안정 제거·람다 안정화·상태 호이스팅·지연 읽기
- 리스트 성능 — LazyColumn·key, 대량 항목
- 종합 — 과한 리컴포지션 화면을 메트릭→수정→Macrobenchmark before/after

→ **총 42개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 Slot Table·Snapshot·컴파일러 변환, `💻 실전 실험`은 Layout Inspector·컴파일러 메트릭·Macrobenchmark. `📊`는 리컴포지션 횟수·프레임 시간 비교.

## 🎨 스타일 가이드

1. **Slot Table/Snapshot로 환원** — 모든 동작을 위치 기억·스냅샷으로 설명
2. **메트릭으로 증명** — 리컴포지션 카운트·skippable 비율 숫자로
3. **React Fiber와 대조** — 위치 기억 vs Fiber, 무효화 vs 재조정
4. **MVCC 동형성 강조** — Snapshot State ↔ InnoDB MVCC를 reactivity-state-compared로 예고
5. Slot Table·Snapshot·리컴포지션 범위는 다이어그램으로

## 🔬 검증 환경

> docker 아님. **Android Studio + Layout Inspector + Compose 컴파일러 메트릭**.

```bash
# 환경: Android Studio, Compose 프로젝트

# 1) Layout Inspector: 리컴포지션 카운트·스킵 카운트(컴포저블별)
# 2) Compose 컴파일러 메트릭/리포트
#    build.gradle.kts:
#    composeCompiler {
#      reportsDestination = layout.buildDirectory.dir("compose_reports")
#      metricsDestination = layout.buildDirectory.dir("compose_metrics")
#    }
#    → *-classes.txt: restartable/skippable/stable 표시 확인
#    → *-composables.txt: 각 컴포저블의 안정성

# 3) Macrobenchmark: FrameTimingMetric으로 jank 측정
# 4) 실험: 불안정 파라미터 vs @Immutable → skippable 변화 관찰
# 5) Snapshot 실험: 동일 State를 여러 곳에서 읽기 → 무효화 범위 추적
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🤖 Android 톤, "Composable을 짠다 vs 내부가 어떻게 도나" 포지셔닝, `🔗 레포 연결`(kotlin·react-internals·reactivity-state-compared)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- *Jetpack Compose Internals* (Jorge Castillo) — 핵심 책
- Compose 공식 문서(Mental model·Lifecycle·Performance) — https://developer.android.com/jetpack/compose
- "Compose under the hood" Android Dev Summit 발표들
- Compose 컴파일러·Snapshot 소스(AOSP)
- Leland Richardson 발표(Compose 컴파일러)

## 💡 핵심 분석 대상

```
Slot Table (Positional Memoization):
  @Composable fun A() {
    Text("hi")       // 위치 0 → 슬롯에 기록
    val x = remember { ... }  // 위치 1 → 슬롯에 저장
    if (cond) Text("y")       // 위치 2 (그룹)
  }
  → 호출 "위치"로 상태를 기억 → 같은 위치=같은 상태 재사용
  → React가 key/순서로 하는 걸 Compose는 위치로

Recomposition + Skip:
  count(State) 읽은 Composable만 → count 변경 시 무효화 → 재실행
  입력 안 변하고 안정(stable) → skip
  불안정 파라미터(예: List) → skip 불가 → 항상 재실행(함정)

Snapshot State ≈ MVCC:
  여러 스냅샷이 동시 존재(읽는 쪽은 일관된 버전)
  쓰기는 자기 스냅샷에 → apply 시 반영, 충돌 감지
  → InnoDB MVCC(Undo Log로 버전 관리)와 같은 문제·같은 해법
  → reactivity-state-compared의 핵심 동형성

컴파일러 주입:
  @Composable fun Greet(name: String)
  ↓
  fun Greet(name: String, $composer: Composer, $changed: Int) {
    $composer.startRestartGroup(key)
    if (스킵 가능 && 입력 안 변함) $composer.skipToGroupEnd()
    else { ...본문... }
    $composer.endRestartGroup()
  }
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 42개 확인 + Layout Inspector/컴파일러 메트릭 검증 환경 + kotlin/react-internals/reactivity-state-compared 연결 지점 명시(특히 Snapshot↔MVCC 동형성).

**준비됐으면 1단계 구조 설계부터 시작해줘!**
