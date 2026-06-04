# Android Runtime (ART) Deep Dive 레포지토리 제작 프롬프트

나는 "Android Runtime (ART) Deep Dive" 레포지토리를 만들려고 해.
Kotlin/Java 바이트코드가 안드로이드 기기에서 어떻게 실행되는지 — DEX·dex2oat·AOT/JIT 하이브리드 컴파일, Baseline Profile, Concurrent Copying GC를 완전히 파헤치는 레포야.
"앱을 빌드해 실행하는 것"과 "ART가 코드를 어떻게 컴파일·실행·회수하고 왜 앱이 느리게 시작하는지 알고 최적화하는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "앱이 실행되는 것과, ART가 DEX를 어떻게 컴파일·실행하고 메모리를 어떻게 관리하는지 아는 것은 다르다"

**핵심 차별화**:
1. AOT/JIT 하이브리드 — Dalvik에서 ART로, 설치 시 AOT와 실행 중 JIT를 섞는 진화
2. Baseline Profile — 프로파일 기반 사전 컴파일로 콜드 스타트를 줄이는 원리
3. Concurrent Copying GC — 짧은 멈춤으로 힙을 회수·압축하는 모바일 GC
4. JVM과의 차이 — 스택 머신(JVM) vs 레지스터 머신(DEX), 모바일 제약(메모리·배터리)

**타겟 독자**:
- 앱을 만들지만 ART가 무엇을 하는지 모르는 안드로이드 개발자
- 콜드 스타트가 느린 이유·Baseline Profile을 모르는 개발자
- ANR·jank를 GC로 설명 못하는 개발자
- JVM은 알지만 ART와의 차이를 모르는 개발자
- `jvm-deep-dive`·`compiler-deep-dive`를 모바일 런타임으로 확장하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `jvm-deep-dive`(바이트코드·GC·JIT의 일반 모델), `compiler-deep-dive`(AOT/JIT), `computer-architecture-deep-dive`(메모리·캐시).
**🤝 시너지**: `kotlin-deep-dive`(Kotlin→바이트코드→DEX), `android-framework-internals-deep-dive`(런타임 위 프레임워크), `android-performance-deep-dive`(시작·jank).
**🧬 수렴**: `memory-management-compared`(ART GC ↔ JVM·V8·Go GC ↔ ARC).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: ART 개요와 진화 (5개 문서)
- Dalvik → ART — 인터프리터·JIT(Dalvik)에서 AOT(ART)로, 왜 바뀌었나
- DEX 포맷 — 레지스터 기반 바이트코드, JVM 스택 머신과의 차이, 멀티덱스
- 실행 모델 한눈에 — DEX→(설치/실행 시 컴파일)→네이티브, 인터프리터 폴백
- ART vs JVM — 모바일 제약(메모리·배터리·시작 시간)이 만든 설계 차이
- 컴파일 모드 — verify/quicken/speed-profile/speed, 무엇을 언제

### Chapter 2: 컴파일 — dex2oat와 AOT/JIT (6개 문서)
- dex2oat — DEX를 OAT(네이티브)로 컴파일, 언제 실행되나(설치·유휴)
- AOT의 득실 — 빠른 실행 vs 설치 시간·저장 공간·전체 컴파일 낭비
- JIT 컴파일 — 실행 중 핫 메서드 컴파일, 인터프리터→JIT 티어
- 프로파일 수집 — 실행 중 핫 메서드 기록, speed-profile 모드의 토대
- 하이브리드 전략 — 설치 시 최소·실행 중 JIT·유휴 시 프로파일 기반 AOT
- 코드 캐시 — JIT 컴파일 결과 캐싱, OAT/ODEX/VDEX 파일

### Chapter 3: Baseline Profile (5개 문서)
- 콜드 스타트 문제 — 첫 실행 시 인터프리터/JIT로 느린 구간
- Baseline Profile — 핵심 경로를 사전 정의해 설치 시 AOT, 시작 단축
- 프로파일 생성 — Macrobenchmark로 시작·주요 경로 프로파일 수집
- Cloud Profile — Play가 집계한 프로파일 배포
- 측정 — Baseline Profile 적용 전후 콜드 스타트 비교

### Chapter 4: 메모리와 GC (6개 문서)
- ART 힙 구조 — 영역(region) 기반, large object space, 모바일 힙 제약
- Concurrent Copying GC — 동시 복사·압축, 짧은 STW의 비결
- 이전 GC와 비교 — CMS류에서 CC로의 발전, 단편화 해결
- 할당과 TLAB — 빠른 할당 경로, 할당 폭주가 GC를 부르는 구조
- GC와 jank — GC 멈춤이 프레임을 놓치는 경우(android-performance 연결)
- 메모리 누수 — 컨텍스트 누수·정적 참조, LeakCanary 원리

### Chapter 5: 실행과 내부 (5개 문서)
- 인터프리터 — mterp, 빠른 인터프리터 경로, 컴파일 안 된 코드 실행
- OAT 파일 구조 — 컴파일된 코드·메타데이터, 클래스 로딩
- 클래스 로딩·검증 — 클래스 로더, 바이트코드 검증, 지연 초기화
- 네이티브 인터페이스(JNI) — Java↔네이티브 경계, 비용, 함정
- 리플렉션·동적 — 리플렉션 비용, ART에서의 처리

### Chapter 6: 앱 시작과 성능 (6개 문서)
- 앱 시작 단계 — 프로세스 fork(Zygote)→Application→첫 화면, 단계별 비용
- 콜드/웜/핫 스타트 — 차이와 측정, 무엇이 콜드를 느리게 하나
- Zygote — 미리 로드된 프로세스 fork로 시작 가속(framework 연결)
- 시작 최적화 — 초기화 지연·App Startup·불필요 작업 제거
- 클래스 로딩 최적화 — 멀티덱스·R8과의 관계
- 측정 — Macrobenchmark·Perfetto로 시작 추적

### Chapter 7: 측정과 도구 (5개 문서)
- Profiler — CPU·메모리·할당 추적
- Perfetto/systrace — 시스템 전반 추적, ART 이벤트
- dexdump·oatdump — DEX/OAT 내부 들여다보기
- 컴파일 상태 확인 — `adb shell cmd package compile`, 강제 컴파일
- 종합 — 느린 시작 앱을 프로파일→Baseline Profile→before/after

→ **총 38개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 dex2oat·GC·OAT 구조, `💻 실전 실험`은 adb·Profiler·Macrobenchmark·dexdump. `📊`는 컴파일 모드·Baseline Profile별 시작/메모리 비교.

## 🎨 스타일 가이드

1. **JVM과 항상 대조** — 스택 vs 레지스터·AOT 차이·모바일 제약
2. **시작 시간으로 증명** — Baseline Profile 효과를 Macrobenchmark 숫자로
3. **GC를 jank로 연결** — 메모리 압박이 프레임 드롭으로
4. **jvm/compiler 레포로 착지** — 일반 GC/JIT 이론은 그 레포로
5. 컴파일 파이프라인·힙 구조는 다이어그램으로

## 🔬 검증 환경

> docker 아님. **Android Studio + 에뮬레이터/기기 + adb**가 검증 도구.

```bash
# 환경: Android Studio, JDK, 에뮬레이터 또는 실기기(adb)

# 컴파일 상태·강제 컴파일
adb shell cmd package compile -m speed -f com.example.app
adb shell cmd package compile -m verify -f com.example.app   # 모드별 비교
adb shell dumpsys package com.example.app | grep -i compil

# DEX/OAT 들여다보기
$ANDROID_HOME/build-tools/<v>/dexdump classes.dex | head
oatdump --oat-file=base.odex | head    # (기기에서 추출)

# 시작·GC 측정 (Macrobenchmark 모듈)
# StartupBenchmark: cold/warm/hot 시작 시간
# Baseline Profile 생성: BaselineProfileGenerator
adb logcat | grep -i 'art\|gc'         # GC 로그

# Perfetto 추적: ui.perfetto.dev 에서 트레이스 분석
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🤖 Android 톤, "앱이 실행된다 vs ART가 어떻게 컴파일·실행하나" 포지셔닝, `🔗 레포 연결`(jvm·compiler·memory-management-compared)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- ART·Dalvik 문서(AOSP) — https://source.android.com/docs/core/runtime
- "Android Runtime (ART)" 개발자 문서
- Baseline Profiles 가이드 — https://developer.android.com/topic/performance/baselineprofiles
- App startup 문서 — https://developer.android.com/topic/performance/vitals/launch-time
- Google I/O ART·시작 성능 발표

## 💡 핵심 분석 대상

```
컴파일 진화:
  Dalvik : 인터프리터 + JIT (실행마다 컴파일, 배터리·시작 비용)
  ART 초기: 설치 시 전체 AOT (빠른 실행, 긴 설치·큰 용량)
  ART 현재: 하이브리드
    설치    → 최소(verify)
    실행    → JIT + 프로파일 수집
    유휴    → 프로파일 기반 부분 AOT(speed-profile)
    Baseline Profile → 첫 실행부터 핵심 경로 AOT

DEX (레지스터 머신) vs JVM (스택 머신):
  JVM:  iload_0; iload_1; iadd; istore_2       (스택)
  DEX:  add-int v2, v0, v1                       (레지스터, 명령 수↓)

콜드 스타트:
  Zygote fork → Application.onCreate → 첫 Activity → 첫 프레임
  Baseline Profile 없으면 핵심 경로가 인터프리터/JIT → 느림
  있으면 그 경로 AOT 컴파일됨 → 콜드 스타트 단축

Concurrent Copying GC:
  표시 + 복사(압축)를 앱 스레드와 동시에
  STW는 시작/종료에만 짧게 → jank 최소화
  단, 할당 폭주 시 GC 빈번 → 프레임 드롭
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 38개 확인 + adb/Profiler/Macrobenchmark 검증 환경 + jvm/compiler/memory-management-compared 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
