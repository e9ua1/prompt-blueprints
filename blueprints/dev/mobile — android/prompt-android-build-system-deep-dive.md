# Android Build System Deep Dive 레포지토리 제작 프롬프트

나는 "Android Build System Deep Dive" 레포지토리를 만들려고 해.
Kotlin/Java 소스가 어떻게 APK/AAB가 되는지 — Gradle 빌드 모델, AGP 변형, R8 옵티마이저, D8 디슈가링·dexing, KSP 어노테이션 처리를 완전히 파헤치는 레포야.
"Gradle 설정을 복붙하는 것"과 "빌드가 어떤 단계로 도는지, R8이 무엇을 어떻게 최적화하는지 알고 설계하는 것"의 차이를 만드는 레포다. 프론트 `frontend-build-tools` 레포와 *대칭*을 이루는 안드로이드 빌드 토대.

## 📋 프로젝트 목표

**컨셉**: "Gradle 설정을 복붙하는 것과, 빌드 단계·R8 최적화·KSP가 어떻게 동작하는지 아는 것은 다르다"

**핵심 차별화**:
1. Gradle 빌드 모델 — init/config/exec 3단계, 태스크 그래프, configuration cache가 빌드를 빠르게 하는 원리
2. R8은 진짜 옵티마이저다 — 단순 난독화가 아닌 tree shaking·inlining·devirtualization을 수행하는 컴파일러(compiler 연결)
3. D8과 ART의 다리 — 디슈가링·dexing이 DEX·ART와 어떻게 연결되나
4. KSP — 왜 kapt보다 빠르고 어떻게 그래프를 만들어내나(architecture·kotlin 연결)

**타겟 독자**:
- Gradle 설정을 복붙하지만 빌드 단계를 모르는 안드로이드 개발자
- R8/Proguard 규칙을 외워 쓰는 개발자
- 빌드가 느려도 원인 진단을 못하는 개발자
- kapt를 KSP로 옮겼지만 *왜* 빨라지는지 모르는 개발자
- `frontend-build-tools`(웹 빌드)와 비교하며 안드로이드 빌드를 이해하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `kotlin-deep-dive`(컴파일·KSP), `android-runtime-deep-dive`(DEX·dex2oat·ART), `compiler-deep-dive`(R8·D8은 컴파일러).
**🤝 시너지**: `android-architecture-deep-dive`(멀티 모듈·Hilt 생성 코드), `cicd-deep-dive`(파이프라인 빌드), `frontend-build-tools-deep-dive`(웹 빌드와 대조 — 다른 도구, 같은 문제).
**🧬 수렴**: 프론트 build-tools와 *대칭*. `compilation-strategies-compared`에 "빌드 타임 컴파일러(R8/D8)" 입력.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: Gradle 빌드 모델 (5개 문서)
- 3단계 — Initialization → Configuration → Execution, 각 단계의 일
- 태스크 그래프 — 입력/출력으로 의존 자동 계산, up-to-date 검사
- Configuration Avoidance — lazy task API, 안 쓰는 태스크 설정 회피
- Configuration Cache — 빌드 스크립트 결과 캐싱(직렬화), 두 번째 빌드 가속의 원리
- Build Cache — 태스크 출력 캐싱, 로컬·원격, 입력 해시 기반

### Chapter 2: AGP와 변형 (5개 문서)
- Android Gradle Plugin — 안드로이드 빌드의 진입점, 라이프사이클 훅
- 변형(Variant) — buildType × productFlavor의 곱, 변형마다 별도 태스크 그래프
- 매니페스트 병합 — 라이브러리·main·변형의 AndroidManifest 합치는 규칙·충돌
- 리소스 컴파일 — AAPT2 동작(컴파일→링크), 리소스 ID, 리소스 축소
- BuildConfig·생성 코드 — 빌드 시 생성되는 자바·코틀린, 사용·디버깅

### Chapter 3: 어노테이션 처리 — kapt vs KSP (5개 문서)
- 어노테이션 처리란 — 컴파일 타임에 코드 생성·검증, 빌드 흐름에 끼는 시점
- kapt가 느린 이유 — Kotlin→Java 스텁 생성→자바 APT 호출, 추가 컴파일 단계
- KSP — Kotlin 네이티브, AST 직접 접근, 스텁 없음 → 2~5배 빠름
- Hilt·Room 생성 코드 — KSP가 만든 의존 그래프·DAO 구현 까보기(architecture 연결)
- 어노테이션 처리 디버깅 — 생성 코드 위치, IDE 인덱싱, 함정

### Chapter 4: R8 — 진짜 옵티마이저 (6개 문서)
- R8의 위치 — D8 + Proguard의 후속, 컴파일러로서의 R8(compiler 연결)
- Tree Shaking — 도달 불가 코드 제거, 진입점·keep 규칙
- 최적화 패스 — inlining·devirtualization·상수 폴딩·죽은 코드 제거(compiler의 IR 최적화와 동형)
- 난독화(Obfuscation) — 식별자 단축·맵 파일, 크래시 역추적
- keep 규칙 — 리플렉션·JNI·직렬화가 R8을 거부하는 패턴, 규칙 작성법
- R8 vs Proguard — 단일 단계 통합, 더 공격적 최적화, 측정으로 효과 비교

### Chapter 5: D8와 DEX (5개 문서)
- D8 — Java 바이트코드 → DEX(레지스터 머신), Dalvik에서 ART로의 다리
- 디슈가링(Desugaring) — 람다·`try-with-resources`·`java.time` 등 신규 API를 구버전 안드로이드로
- 멀티덱스 — 메서드 수 한계(65k) 우회, 시작 비용
- DEX 구조 — 클래스 풀·메서드 풀·바이트코드, dexdump로 까보기(android-runtime 연결)
- D8→ART — DEX가 dex2oat로 OAT 되는 다음 단계(android-runtime 연결)

### Chapter 6: 배포 — APK와 AAB (4개 문서)
- APK 구조 — DEX·리소스·매니페스트·서명, ZIP 기반
- App Bundle(AAB) — 기기별 분할 다운로드, Play가 동적 APK 생성
- 동적 기능 모듈 — 온디맨드 설치, 사용자 경로
- 서명 — V1/V2/V3/V4 서명, Play 앱 서명

### Chapter 7: 빌드 성능과 도구 (5개 문서)
- 모듈 그래프 — 멀티 모듈이 증분·병렬 빌드에 주는 영향(architecture 연결)
- 증분 빌드 — 입력 해시·태스크 캐시·configuration cache가 함께 만드는 빠른 빌드
- Build Scan — Gradle Enterprise·gradle.com의 빌드 프로파일링
- R8 메트릭 — `-printusage`·`-printseeds`로 무엇이 제거됐나 확인
- 종합 — 느린 빌드를 Build Scan으로 진단→모듈/configuration/KSP 정리→before/after

→ **총 35개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 Gradle 단계·R8 패스·D8 디슈가링·KSP 처리, `💻 실전 실험`은 `--profile`·Build Scan·R8 로그·dexdump·생성 코드 까보기. `📊`는 빌드 시간·APK 크기·KSP/kapt 정량 비교.

## 🎨 스타일 가이드

1. **3단계로 환원** — 모든 빌드 동작을 Init/Config/Exec 단계로 분류
2. **R8 = 컴파일러 강조** — Tree shaking·inlining이 compiler 레포의 최적화와 같은 일
3. **kapt vs KSP를 측정** — "왜 빠른지" 빌드 시간 숫자로
4. **android-runtime과 연결** — D8 출력이 dex2oat 입력으로 어떻게 이어지나
5. **frontend-build-tools와 대조** — 같은 문제(번들·트리셰이킹·HMR vs 증분)를 다른 도구로
6. 빌드 단계·R8 파이프라인·KSP 처리는 다이어그램으로

## 🔬 검증 환경

> 실제 Android 멀티 모듈 프로젝트 + Build Scan + R8 로그 + dexdump.

```bash
# 환경: Android Studio, Gradle, 멀티 모듈 샘플 앱(예: nowinandroid)

# 1) 빌드 단계 프로파일링
./gradlew assembleRelease --profile
#   → build/reports/profile/ 에 Init/Config/Exec 시간 분석

# 2) Configuration Cache 효과
./gradlew assembleDebug --configuration-cache
./gradlew assembleDebug --configuration-cache    # 2회차: 거의 0초 config

# 3) 태스크 그래프
./gradlew assembleDebug --dry-run                # 실행할 태스크 목록
./gradlew :app:dependencies                       # 의존 그래프

# 4) KSP vs kapt 비교
# 같은 Hilt/Room 프로젝트를 kapt → KSP로 전환 → 빌드 시간 비교
./gradlew --profile assembleDebug

# 5) R8 동작 확인
./gradlew assembleRelease
# build/outputs/mapping/release/usage.txt   <- 제거된 코드
# build/outputs/mapping/release/seeds.txt   <- keep된 진입점
# build/outputs/mapping/release/mapping.txt <- 난독화 맵

# 6) DEX 까보기
$ANDROID_HOME/build-tools/<v>/dexdump -d classes.dex | head
# 메서드 수·디슈가된 코드·R8 결과 확인

# 7) APK/AAB 검사
$ANDROID_HOME/build-tools/<v>/aapt2 dump apk app-release.apk
bundletool dump manifest --bundle=app-release.aab
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `sample-app/`(멀티 모듈·KSP·R8 적용 샘플)
2. **README.md**: 🖥️ Android Infra 톤, "Gradle을 복붙한다 vs 빌드가 어떻게 도나" 포지셔닝, `🔗 레포 연결`(kotlin·android-runtime·compiler·frontend-build-tools 대조)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- Android Gradle Plugin 문서 — https://developer.android.com/build
- R8 문서·소스 — https://r8.googlesource.com/r8/
- KSP 문서 — https://kotlinlang.org/docs/ksp-overview.html
- Gradle 공식 — Configuration Cache·Build Cache 가이드
- "Demystifying R8" Android Dev Summit 발표들
- *Gradle in Action* (Hubert Klein Ikkink)

## 💡 핵심 분석 대상

```
빌드 3단계:
  Initialization: settings.gradle·프로젝트 트리 결정
  Configuration : build.gradle 실행·태스크 그래프 구성
  Execution    : 그래프 순서대로 태스크 실행(병렬·증분)
  -> Configuration Cache: Configuration 결과를 캐시 -> 2회차 거의 즉시

전체 빌드 파이프라인:
  Kotlin/Java 소스
    | kotlinc/javac
    v
  JVM 바이트코드(.class)
    | R8 (Tree shaking + 최적화 + 난독화)
    v
  최소화된 .class
    | D8 (디슈가링 + Dexing)
    v
  DEX 바이트코드(.dex)         <- android-runtime의 시작점
    | AAPT2 + 패키징 + 서명
    v
  APK / AAB

R8 최적화 패스 (compiler와 동형):
  Tree Shaking         : 도달 불가 클래스/메서드 제거
  Inlining             : 핫 메서드 호출부에 펼치기
  Devirtualization     : 가상 호출을 직접 호출로
  Method Outlining     : 반복 패턴을 메서드로(코드 작아짐)
  난독화               : 식별자 단축(a, b, c...)
  -> "런타임 0에 작은 APK"

kapt vs KSP:
  kapt:  Kotlin -> Java 스텁 생성 -> 자바 APT 실행 -> Kotlin 컴파일
         (한 번 더 컴파일 = 느림)
  KSP :  Kotlin AST 직접 접근 -> 코드 생성
         스텁 없음 -> 2~5배 빠름
  -> Hilt·Room이 KSP 마이그레이션한 이유

웹 빌드와 대조 (같은 문제, 다른 도구):
  웹      : webpack/Vite -> 번들·트리셰이킹·HMR
  Android : Gradle + R8 -> DEX·트리셰이킹·증분 빌드
  -> 4축 동일: 의존 그래프 / 변환 / 최소화 / 증분
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 35개 확인 + Gradle/R8/D8/Build Scan 검증 환경 + kotlin/android-runtime/compiler/frontend-build-tools 연결 지점 명시. **R8을 진짜 옵티마이저로**·**KSP의 빠름을 정량적으로**·**프론트 빌드와의 대칭**을 분명히.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
