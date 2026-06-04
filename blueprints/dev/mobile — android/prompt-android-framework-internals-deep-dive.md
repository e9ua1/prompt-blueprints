# Android Framework Internals Deep Dive 레포지토리 제작 프롬프트

나는 "Android Framework Internals Deep Dive" 레포지토리를 만들려고 해.
앱과 시스템이 어떻게 통신하는지 — Binder IPC, 시스템 서비스(AMS 등), Looper/Handler/MessageQueue, 액티비티 생명주기의 *실제 흐름*을 완전히 파헤치는 레포야.
"생명주기 콜백을 구현하는 것"과 "onCreate가 누가·어떻게 호출하고 앱이 시스템과 어떻게 IPC하는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "생명주기 콜백을 구현하는 것과, 시스템이 Binder로 앱과 통신하며 그 콜백을 호출하는 흐름을 아는 것은 다르다"

**핵심 차별화**:
1. Binder IPC — 모든 시스템 통신의 기반, 프로세스 경계를 넘는 메서드 호출의 원리
2. 시스템 서비스 — AMS·WMS·PMS가 앱을 관리하는 방식, 앱은 프록시를 호출
3. 메시지 루프 — Looper/Handler/MessageQueue가 메인 스레드를 돌리는 구조
4. 생명주기의 실체 — onCreate/onResume가 시스템에서 Binder를 거쳐 호출되는 전체 경로

**타겟 독자**:
- 생명주기 콜백을 외워 쓰지만 누가 호출하는지 모르는 개발자
- Binder를 단어로만 아는 개발자
- Handler·Looper를 쓰지만 메인 스레드가 어떻게 도는지 모르는 개발자
- "context"가 정확히 뭔지·왜 누수가 위험한지 모르는 개발자
- `linux-for-backend`의 IPC를 안드로이드로 확장하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `linux-for-backend-deep-dive`(프로세스·IPC·스케줄링), `android-runtime-deep-dive`(런타임 위 프레임워크).
**🤝 시너지**: `android-architecture-deep-dive`(생명주기 위 아키텍처), `jetpack-compose-internals-deep-dive`(Choreographer·메인 스레드), `android-performance-deep-dive`(ANR).
**🧬 수렴**: `concurrency-models-compared`(메시지 루프 모델), `distributed-systems-theory-deep-dive`(IPC = 작은 분산).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 안드로이드 프레임워크 구조 (5개 문서)
- 계층 구조 — 앱→프레임워크→네이티브→커널, 각 계층의 역할
- 앱과 시스템의 경계 — 앱은 별도 프로세스, 시스템 서비스와 IPC로 대화
- 시스템 서버 — system_server 프로세스, 주요 서비스들의 거처
- 앱 프로세스 — UID 격리·샌드박스, 앱이 가진 것과 못 가진 것
- 전체 그림 — 탭 한 번이 시스템을 거쳐 앱 콜백으로 오는 여정 미리보기

### Chapter 2: Binder IPC (6개 문서)
- Binder가 푸는 문제 — 프로세스 격리 + 효율적 통신, 왜 소켓/파이프가 아닌가
- Binder 아키텍처 — 드라이버(커널)·서비스·프록시, 클라-서버 모델
- AIDL — 인터페이스 정의→스텁/프록시 생성, 마샬링/언마샬링
- Binder 트랜잭션 — Parcel, 스레드 풀, 동기/비동기 호출
- 토큰과 권한 — Binder 토큰으로 신원·권한, 보안 경계
- 한계와 함정 — 트랜잭션 크기 제한(TransactionTooLarge), 데드락

### Chapter 3: 시스템 서비스와 AMS (6개 문서)
- ServiceManager — 서비스 등록·조회, Binder 디렉토리
- ActivityManagerService(AMS) — 컴포넌트·프로세스·태스크 관리의 중심
- 컴포넌트 시작 흐름 — startActivity가 AMS를 거쳐 대상 앱으로 가는 경로
- PackageManager·WindowManager — 앱 정보·윈도우 관리 개요
- 프로세스 우선순위 — oom_adj, 백그라운드 프로세스 종료 결정
- 권한 시스템 — 런타임 권한이 시스템 서비스로 검증되는 방식

### Chapter 4: Looper / Handler / MessageQueue (6개 문서)
- 메인 스레드 루프 — main()→Looper.loop(), 앱이 종료 안 되고 도는 이유
- MessageQueue — 메시지 큐·지연 메시지·동기 배리어
- Handler — 메시지 송신·콜백, 스레드 간 통신
- 메시지 처리 — 한 번에 하나, 큐가 비면 대기(epoll 기반)
- 비동기 패턴 — HandlerThread·post·메인 스레드 마샬링
- ANR의 메커니즘 — 메인 스레드 블로킹 → 메시지 처리 지연 → ANR(performance 연결)

### Chapter 5: 생명주기 실제 흐름 (6개 문서)
- 액티비티 시작 전체 경로 — Intent→AMS→프로세스 생성→ActivityThread→onCreate
- ActivityThread — 앱 프로세스의 메인, 시스템 콜백을 메시지로 받아 생명주기 호출
- 생명주기 콜백의 출처 — onCreate/onStart/onResume를 누가 어떤 순서로
- 구성 변경 — 회전 시 재생성, ViewModel이 살아남는 이유(architecture 연결)
- 프로세스 사망·복원 — 백그라운드 종료, savedInstanceState 복원
- Context의 실체 — Application/Activity Context 차이, 누수가 위험한 이유

### Chapter 6: Zygote와 프로세스 (5개 문서)
- Zygote — 미리 로드된 템플릿 프로세스, fork로 앱 시작 가속(runtime 연결)
- 프로세스 생성 — fork→specialize, 공유 메모리(copy-on-write)
- 앱 생명 — Application 객체, 프로세스와 앱의 관계
- 멀티 프로세스 앱 — 별도 프로세스 컴포넌트, IPC 비용
- 백그라운드 제한 — Doze·백그라운드 실행 제한, WorkManager 위임

### Chapter 7: 측정과 디버깅 (6개 문서)
- dumpsys — activity·meminfo 등 시스템 상태 덤프
- Binder 추적 — 트랜잭션 관찰, 과도한 IPC 진단
- Perfetto/systrace — 메인 스레드·메시지·생명주기 추적
- ANR 분석 — traces.txt·ANR 덤프 읽기, 원인 특정
- 메모리·누수 — Context 누수 추적
- 종합 — 탭→화면 전환 전 과정을 Perfetto로 추적·해부

→ **총 40개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 Binder·AMS·ActivityThread 흐름(AOSP 수준), `💻 실전 실험`은 dumpsys·Perfetto·adb. `📊`는 IPC 비용·시작 단계 시간.

## 🎨 스타일 가이드

1. **"누가 호출하나"에 답한다** — 모든 콜백의 출처를 시스템→Binder→앱 경로로
2. **Perfetto로 추적** — 추상 흐름을 실제 트레이스로 *보여준다*
3. **linux/distributed와 연결** — Binder는 linux IPC + 작은 분산
4. **함정 중심** — Context 누수·TransactionTooLarge·ANR을 메커니즘으로
5. 시작 경로·메시지 루프는 시퀀스 다이어그램으로

## 🔬 검증 환경

> docker 아님. **Android Studio + 기기/에뮬 + adb + AOSP 소스 읽기**.

```bash
# 시스템 상태 덤프
adb shell dumpsys activity activities | head     # 액티비티·태스크
adb shell dumpsys activity processes | head      # 프로세스·oom_adj
adb shell dumpsys meminfo com.example.app

# 생명주기·ANR 관찰
adb logcat | grep -i 'ActivityThread\|lifecycle'
adb shell cat /data/anr/traces.txt               # ANR 덤프(권한 필요)

# Binder 트랜잭션
adb shell cat /sys/kernel/debug/binder/stats      # (루팅/에뮬)

# Perfetto: ui.perfetto.dev 에서 system trace 캡처·분석
#  - 'system_server', 앱 메인 스레드, Binder 트랜잭션 트랙

# AOSP 소스: cs.android.com 에서 AMS·ActivityThread·Looper 읽기
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🤖 Android 톤, "콜백을 구현한다 vs 누가 호출하나" 포지셔닝, `🔗 레포 연결`(linux·runtime·architecture)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- AOSP 소스 검색 — https://cs.android.com/
- "Android Binder" (Thorsten Schreiber 등) 분석 글
- *Android Internals* (Jonathan Levin)
- 개발자 문서: 프로세스·스레드, 액티비티 생명주기
- Gityuan 블로그(중국어, AMS·Binder 심층)

## 💡 핵심 분석 대상

```
앱 ↔ 시스템 (Binder):
  앱 프로세스          system_server
  ───────────         ─────────────
  startActivity() ──Binder──► AMS.startActivity()
                              프로세스 없으면 Zygote fork
       ◄──Binder── ActivityThread.scheduleLaunchActivity()
  Looper가 메시지 처리 → onCreate() 호출!
  → onCreate는 "내가" 부른 게 아니라 시스템이 Binder로 스케줄

메인 스레드 루프:
  main() {
    Looper.prepareMainLooper();
    ActivityThread.attach();   // AMS에 등록
    Looper.loop();             // 무한 메시지 처리 (앱이 안 끝나는 이유)
  }
  메시지 = 생명주기 콜백·입력·draw 요청
  한 메시지가 길면(>5s 입력) → ANR

Binder 트랜잭션:
  클라이언트 → 프록시(스텁) → Binder 드라이버(커널) → 서버 스레드풀 → 실제 메서드
  Parcel로 인자 마샬링, 크기 제한(~1MB) → TransactionTooLarge

Context 누수:
  static Activity / 긴 생명 객체가 Activity Context 잡음
  → Activity 파괴돼도 GC 안 됨 → 메모리 누수
  → Application Context를 써야 할 곳 구분
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 40개 확인 + adb/dumpsys/Perfetto/AOSP 검증 환경 + linux/runtime/architecture 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
