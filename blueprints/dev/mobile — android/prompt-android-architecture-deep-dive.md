# Android Architecture Deep Dive 레포지토리 제작 프롬프트

나는 "Android Architecture Deep Dive" 레포지토리를 만들려고 해.
MVVM/MVI 단방향 흐름, Clean Architecture 레이어, 멀티 모듈화, 컴파일타임 DI(Hilt)가 *왜 그렇게 설계*되고 무엇을 해결하는지를 완전히 파헤치는 레포야.
"아키텍처 패턴을 따라 하는 것"과 "각 레이어·DI·모듈 경계가 무엇을 보장하고 어떤 트레이드오프를 갖는지 알고 설계하는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "MVVM 템플릿을 따르는 것과, 단방향 흐름·의존성 규칙·모듈 경계가 무엇을 보장하는지 알고 설계하는 것은 다르다"

**핵심 차별화**:
1. 단방향 데이터 흐름 — MVI/MVVM이 상태 예측가능성을 만드는 메커니즘(생명주기·구성변경 대응)
2. 의존성 규칙 — Clean Architecture의 안쪽으로만 향하는 의존, DIP의 실전
3. 멀티 모듈 — 빌드 속도·경계·캡슐화, 모듈 의존 그래프 설계
4. 컴파일타임 DI — Hilt가 KSP로 의존 그래프를 *컴파일 때* 검증·생성하는 원리

**타겟 독자**:
- MVVM을 쓰지만 왜 ViewModel이 구성 변경에 살아남는지 모르는 개발자
- MVI·단방향을 단어로만 아는 개발자
- Hilt를 어노테이션 복붙으로 쓰고 그래프 생성 원리를 모르는 개발자
- 멀티 모듈을 "그냥 나눈" 개발자
- `architecture-patterns`·`ddd` 백엔드 지식을 안드로이드로 옮기려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `android-framework-internals-deep-dive`(생명주기·구성변경), `kotlin-deep-dive`(코루틴·Flow·KSP).
**🤝 시너지**: `architecture-patterns-deep-dive`/`ddd-deep-dive`(백엔드 아키텍처 원리 공유), `jetpack-compose-internals-deep-dive`(상태 호이스팅), `spring-core-deep-dive`(DI 대조).
**🧬 수렴**: `reactivity-state-compared`(단방향 상태), 백엔드 architecture 레포와 *같은 원리, 다른 플랫폼*.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 아키텍처 원칙 (5개 문서)
- 왜 아키텍처인가 — 생명주기·구성변경·프로세스 사망이라는 안드로이드 고유 제약
- 관심사 분리 — UI/로직/데이터 분리, 테스트 가능성
- 단방향 데이터 흐름 — 상태가 한 방향으로, 예측가능성의 근원
- 권장 아키텍처 — Google의 UI/Domain/Data 레이어 가이드
- 안티패턴 — God Activity·로직이 UI에·생명주기 무시

### Chapter 2: MVVM과 MVI (6개 문서)
- MVVM — View/ViewModel/Model, 데이터 바인딩·관찰
- ViewModel의 생존 — 구성 변경에 살아남는 이유(framework·ViewModelStore 연결)
- 상태 노출 — StateFlow/LiveData로 UI 상태, collectAsState(compose)
- MVI — 단일 상태·인텐트·환원, Redux 유사(state-management 대조)
- 이벤트 vs 상태 — 일회성 이벤트(SnackBar) 처리의 어려움·패턴
- MVVM vs MVI — 트레이드오프, 언제 무엇을

### Chapter 3: Clean Architecture와 레이어 (6개 문서)
- 레이어 — Presentation/Domain/Data, 각 책임
- 의존성 규칙 — 안쪽(도메인)으로만 의존, 바깥은 추상에 의존(DIP)
- UseCase — 도메인 로직 캡슐화, 언제 필요·언제 과한가
- Repository 패턴 — 데이터 소스 추상화, 도메인이 구현을 모르게
- 모델 분리 — DTO/Entity/Domain Model, 매핑의 비용과 이득
- 백엔드 아키텍처와 동형 — architecture-patterns·ddd의 원리가 그대로 적용됨

### Chapter 4: 멀티 모듈화 (6개 문서)
- 왜 모듈인가 — 빌드 속도(병렬·증분)·경계·재사용
- 모듈 종류 — app/feature/core/data 모듈, 의존 그래프 설계
- 모듈 경계 — public API vs internal, 캡슐화 강제
- 모듈 간 통신 — 인터페이스·네비게이션·공유 모듈
- 빌드 성능 — 모듈화가 빌드에 주는 영향, 측정
- 함정 — 과한 모듈화·순환 의존·잘못된 경계

### Chapter 5: 의존성 주입 — Hilt (6개 문서)
- DI가 푸는 문제 — 결합도·테스트, 수동 DI의 한계
- Dagger/Hilt 개요 — 컴파일타임 DI, 런타임 리플렉션 없는 이유
- 컴파일타임 그래프 — KSP/annotation processing이 그래프를 *컴파일 때* 생성·검증
- 컴포넌트·스코프 — Singleton/ActivityRetained 등, 생명주기와 결합
- 바인딩 — @Provides/@Binds, 한정자(qualifier), 다중 바인딩
- Spring DI와 대조 — 컴파일타임(Hilt) vs 런타임(Spring) DI의 트레이드오프

### Chapter 6: 데이터 레이어 (5개 문서)
- Repository 구현 — 네트워크+로컬 조합, 단일 진실 원천
- 로컬 저장 — Room 개요(local-first 레포로 심화 위임)
- 오프라인 우선 — 캐시·동기화 전략(local-first 연결)
- 비동기 데이터 — Flow로 데이터 스트림, 코루틴 통합
- 에러·상태 — Result 래핑, 로딩/에러/성공 상태 모델링

### Chapter 7: 테스트와 측정 (5개 문서)
- 테스트 가능성 — 아키텍처가 테스트를 쉽게 만드는 방식
- 레이어별 테스트 — ViewModel·UseCase·Repository 단위 테스트
- 가짜 구현 — Fake/Test Double, Hilt 테스트 모듈
- 빌드·아키텍처 측정 — 모듈 의존 그래프 시각화, 빌드 시간
- 종합 — 작은 앱을 Clean + MVI + 멀티모듈 + Hilt로 설계·구현

→ **총 39개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 ViewModel 생존·Hilt 그래프 생성·모듈 의존, `💻 실전 실험`은 생성된 Hilt 코드·모듈 그래프·테스트. `📊`는 아키텍처 선택·모듈화의 빌드/테스트 영향.

## 🎨 스타일 가이드

1. **"무엇을 보장하나"에 답한다** — 패턴을 따라 하지 말고 보장·트레이드오프로
2. **백엔드 아키텍처와 동형 강조** — architecture-patterns·ddd 원리가 그대로
3. **Hilt 생성 코드 까보기** — 컴파일타임 그래프를 *직접 본다*
4. **framework 레포로 착지** — ViewModel 생존·생명주기는 그 레포로
5. 레이어·의존 그래프·단방향 흐름은 다이어그램으로

## 🔬 검증 환경

> docker 아님. **Android Studio + Gradle + 생성 코드 확인**.

```bash
# 환경: Android Studio, 멀티모듈 Gradle 프로젝트

# 1) Hilt 생성 코드 확인
#    build/generated/.../Hilt_*.java, *_HiltModules 등
#    → 컴파일타임에 만들어진 의존 그래프·팩토리 직접 열람
#    의존 누락 시 "컴파일 에러"로 잡힘(런타임 아님) → 핵심 이점

# 2) 모듈 의존 그래프 시각화
./gradlew :app:dependencies
# 또는 module-graph 플러그인으로 시각화

# 3) 빌드 측정
./gradlew assembleDebug --profile     # 빌드 리포트
# 모듈화 전후 증분 빌드 시간 비교

# 4) ViewModel 생존 실험: 회전 시 상태 유지 vs Activity 재생성 관찰
# 5) 테스트: ViewModel/UseCase 단위 테스트, Hilt 테스트 모듈로 Fake 주입
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🤖 Android 톤, "패턴을 따른다 vs 무엇을 보장하나" 포지셔닝, `🔗 레포 연결`(framework·architecture-patterns·ddd)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- Android 앱 아키텍처 가이드 — https://developer.android.com/topic/architecture
- Now in Android 샘플 — https://github.com/android/nowinandroid
- Hilt 문서 — https://developer.android.com/training/dependency-injection/hilt-android
- *Clean Architecture* (Robert C. Martin) — 원리(백엔드와 공유)
- "Guide to app modularization" (Android)

## 💡 핵심 분석 대상

```
단방향 데이터 흐름 (MVI):
  Intent(사용자 동작) → ViewModel
                         → reduce(state, intent) → 새 State
                         → UI는 State를 관찰·렌더 (한 방향)
  → 상태가 예측 가능, 디버깅 쉬움(state-management의 Redux와 동형)

의존성 규칙 (Clean):
  Presentation → Domain ← Data
  (바깥)         (안)      (바깥)
  - Domain은 아무것도 의존 안 함(순수)
  - Data는 Domain의 인터페이스를 구현(DIP)
  → 백엔드 ddd·architecture-patterns와 같은 원리!

Hilt 컴파일타임 그래프:
  @Inject constructor(repo: Repo)    // 의존 선언
  @Binds RepoImpl → Repo             // 바인딩
  ↓ KSP가 컴파일 때
  팩토리·컴포넌트 코드 생성 + 그래프 검증
  → 의존 빠지면 "컴파일 에러"(Spring은 런타임 에러) ← 핵심 차이

ViewModel 생존:
  Activity 회전 → Activity 재생성
  ViewModel은 ViewModelStore에 보관 → 재생성 후 같은 인스턴스 복원
  → 데이터 재로딩 불필요 (framework 레포: ViewModelStore 메커니즘)
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 39개 확인 + Hilt생성코드/모듈그래프/빌드프로파일 검증 환경 + framework/architecture-patterns/ddd 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
