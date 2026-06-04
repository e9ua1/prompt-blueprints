# Go Deep Dive 레포지토리 제작 프롬프트

나는 "Go Deep Dive" 레포지토리를 만들려고 해.
고루틴이 어떻게 수만 개씩 떠도 가벼운지(M:N 스케줄러), 채널이 동시성을 어떻게 안전하게 만드는지, GC가 어떻게 짧은 STW로 동작하는지를 완전히 파헤치는 레포야.
"Go로 동시성 코드를 짜는 것"과 "런타임이 고루틴·채널·GC를 어떻게 구현하는지 알고 설계하는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "고루틴을 띄우는 것과, 런타임 스케줄러가 그것을 OS 스레드 위에 어떻게 다중화하는지 아는 것은 다르다"

**핵심 차별화**:
1. M:N 스케줄러(GMP) — 고루틴이 OS 스레드가 아니라서 수만 개가 가능한 원리, 워크 스틸링
2. 채널의 실체 — "메모리를 공유하지 말고 통신하라"가 런타임에서 어떻게 구현되나
3. 동시성 GC — 삼색 표시·쓰기 배리어로 STW를 1ms 미만으로 줄인 방식
4. 단순함의 설계 — 인터페이스 동적 디스패치·escape 분석·고루틴 스택 증가의 트레이드오프

**타겟 독자**:
- 고루틴을 마구 띄우지만 스케줄러가 어떻게 돌리는지 모르는 개발자
- 채널 데드락을 만나지만 채널 내부 동작을 모르는 개발자
- "Go는 GC가 빠르다"고 들었지만 왜인지 설명 못하는 개발자
- 인터페이스를 쓰지만 nil 인터페이스 함정의 원리를 모르는 개발자
- Rust/Java 동시성과 Go가 어떻게 다른지 궁금한 개발자

## 🔗 레포 연결

**⬆️ 선행**: `computer-architecture-deep-dive`(스택·캐시·원자연산), `linux-for-backend-deep-dive`(스레드·스케줄링·epoll).
**🤝 시너지**: `rust-deep-dive`(GC 없는 동시성과 대조), `java-concurrency-deep-dive`(가상 스레드 vs 고루틴), `kubernetes-deep-dive`(Go로 쓰여진 시스템 읽기).
**🧬 수렴**: `concurrency-models-compared`(고루틴+채널 = CSP 모델의 대표), `memory-management-compared`(Go GC).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: Go의 설계 철학과 기본 (5개 문서)
- Go가 푸는 문제 — 단순함·빠른 컴파일·동시성, 언어가 의도적으로 *뺀* 것들(제네릭 이전 역사 포함)
- 값·포인터·메모리 — 값 타입 기본, 포인터, 스택 vs 힙, escape 분석이 결정하는 할당 위치
- 슬라이스 내부 — ptr/len/cap 3-워드 헤더, append의 재할당, 슬라이스 공유 함정
- map 내부 — 해시 버킷, 확장(grow), 순회 무작위성의 이유, map은 왜 동시성 안전이 아닌가
- 인터페이스 표현 — (type, value) 2-워드, nil 인터페이스 vs nil 포인터 함정

### Chapter 2: 고루틴과 스케줄러 (6개 문서)
- 고루틴이란 — OS 스레드가 아닌 런타임 관리 실행 단위, 2KB 시작 스택, 수만 개의 비결
- GMP 모델 — G(고루틴)/M(OS 스레드)/P(논리 프로세서), 셋의 관계와 GOMAXPROCS
- 스케줄링 — 협력적+선점적(비동기 선점), 스케줄 지점, P의 로컬 런큐
- 워크 스틸링 — 한가한 P가 바쁜 P의 큐에서 훔치기, 부하 분산
- 시스템콜과 블로킹 — 블로킹 시 M을 떼어내고 P를 다른 M에, 네트워크 폴러(netpoller)
- 스택 관리 — 분할 스택→연속 스택, 고루틴 스택 증가/축소, 함수 prologue의 스택 체크

### Chapter 3: 채널과 동시성 패턴 (6개 문서)
- 채널 내부 — hchan 구조(버퍼·sendq·recvq), 송수신이 큐와 락으로 동작하는 방식
- 버퍼드 vs 언버퍼드 — 동기/비동기 의미, 랑데부, 블로킹 조건
- select 동작 — 여러 채널 대기, 무작위 선택, default, 구현 원리
- 동시성 패턴 — 파이프라인·팬아웃/팬인·worker pool을 채널로
- 취소와 컨텍스트 — context.Context로 취소·타임아웃 전파, 고루틴 누수 방지
- 동기화 프리미티브 — sync.Mutex/WaitGroup/Once, 언제 채널 대신 락을 쓰나("Share by communicating" vs 현실)

### Chapter 4: 메모리 모델과 GC (6개 문서)
- Go 메모리 모델 — happens-before, 채널·락이 주는 동기화 보장, 데이터 레이스의 정의
- 레이스 디텍터 — `-race`가 런타임에 레이스를 잡는 원리, 한계
- GC 개요 — 동시 표시-쓸기(concurrent mark-sweep), 세대 없는 GC를 택한 이유
- 삼색 표시 — white/grey/black, 동시 표시 중 변이 처리
- 쓰기 배리어 — 동시 표시의 정확성을 위한 배리어, 짧은 STW의 비결
- GC 튜닝과 관찰 — GOGC, GOMEMLIMIT, gctrace로 GC 사이클·STW 측정

### Chapter 5: 타입 시스템과 인터페이스 (5개 문서)
- 인터페이스 동적 디스패치 — itab(인터페이스 테이블), 메서드 조회, 정적 타입 vs 동적 타입
- 타입 단언·타입 스위치 — 런타임 타입 검사 비용, comma-ok
- 임베딩 — 상속 아닌 구성, 메서드 승격, 인터페이스 임베딩
- 제네릭(1.18+) — 타입 파라미터, 제약(constraints), GC shape stenciling 구현 방식
- 에러 처리 — error 인터페이스, errors.Is/As, panic/recover의 동작과 비용

### Chapter 6: 런타임과 성능 (5개 문서)
- escape 분석 — 컴파일러가 스택/힙을 결정하는 규칙, `-gcflags=-m`으로 확인, 할당 줄이기
- 인라이닝 — 인라인 조건, 비용 모델, 인라인이 막히는 패턴
- pprof 프로파일링 — CPU·힙·고루틴·블록 프로파일, 플레임 그래프로 병목 특정
- 어셈블리 관찰 — `go tool compile -S`로 생성 코드 보기, 바운드 체크 제거
- 흔한 성능 함정 — 불필요한 인터페이스 박싱, 슬라이스 재할당, 고루틴 누수 측정

### Chapter 7: 표준 라이브러리 내부 (5개 문서)
- net/http 서버 내부 — 연결당 고루틴 모델, netpoller와 epoll, 동시성 처리
- sync.Pool — 객체 재사용으로 할당 줄이기, GC와의 상호작용, P별 풀
- time과 타이머 — 타이머 힙, 런타임 타이머 관리
- io 인터페이스 설계 — Reader/Writer 조합, 작은 인터페이스 철학
- 종합 사례 — 간단한 동시성 서버를 짜고 pprof·gctrace로 전 구간 관찰

→ **총 38개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 런타임 소스(runtime 패키지)·구조체, `💻 실전 실험`은 gctrace·schedtrace·pprof·escape 분석. `📊`는 고루틴 수·할당·STW의 정량 측정.

## 🎨 스타일 가이드

1. **런타임 관찰로 증명** — "스케줄러가 옮긴다"가 아니라 GODEBUG=schedtrace 출력으로
2. **할당을 escape 분석으로** — `-gcflags=-m`으로 스택/힙 결정을 직접 확인
3. **Rust/Java와 대조** — 고루틴 vs Rust async vs Java 가상 스레드, GC vs 소유권
4. **함정 중심** — nil 인터페이스·고루틴 누수·슬라이스 공유 등 *런타임 동작이 만든 함정*
5. GMP·채널·삼색 표시는 항상 다이어그램으로

## 🔬 검증 환경

> docker 불필요. **go 툴체인 + GODEBUG + pprof**가 검증 도구.

```dockerfile
FROM golang:1-bookworm
RUN apt-get update && apt-get install -y graphviz   # pprof 시각화
```

```bash
# 스케줄러 관찰
GODEBUG=schedtrace=1000 ./app      # 1초마다 P/M/G 상태
GODEBUG=scheddetail=1,schedtrace=1000 ./app

# GC 관찰
GODEBUG=gctrace=1 ./app            # 사이클·STW·힙 크기
GOGC=200 ./app                     # GC 빈도 조절

# escape 분석 / 인라이닝
go build -gcflags='-m -m' ./...

# 생성 어셈블리
go tool compile -S main.go

# 레이스 검출
go run -race main.go

# 프로파일링
go test -cpuprofile cpu.prof -memprofile mem.prof -bench .
go tool pprof -http=:8080 cpu.prof
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🧱 Languages 톤, "고루틴을 띄운다 vs 스케줄러가 어떻게 돌리나" 포지셔닝, `🔗 레포 연결`
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- Go 공식 문서·메모리 모델 — https://go.dev/ref/mem
- *The Go Programming Language* (Donovan & Kernighan)
- Go runtime 소스 — https://github.com/golang/go/tree/master/src/runtime
- Dave Cheney 블로그 — https://dave.cheney.net/
- "Scheduling in Go" (Bill Kennedy, Ardan Labs)
- GopherCon 스케줄러·GC 발표들

## 💡 핵심 분석 대상

```
GMP 스케줄러:
  G(고루틴) 수만 개  ──┐
  G  G  G  G ...       │ P의 로컬 런큐
                       ▼
  P(논리프로세서) ── M(OS스레드) ── CPU 코어
  P  P  P  (= GOMAXPROCS)
  - M이 시스템콜로 블로킹 → P를 떼어 다른 M에 붙임(고루틴은 계속 진행)
  - 한가한 P → 바쁜 P의 큐에서 G를 훔침(work stealing)
  → 고루틴 2KB 스택 + M:N 다중화 = 수만 고루틴 가능

채널 (hchan):
  ch <- v   : 수신 대기자(recvq) 있으면 직접 전달, 없으면 버퍼/블록
  <- ch     : 송신 대기자(sendq) 있으면 받기, 없으면 버퍼/블록
  언버퍼드  : 송수신이 만나야 진행(랑데부)

동시 GC (삼색):
  white(미방문) / grey(방문, 자식 미처리) / black(완료)
  STW: 표시 시작/종료에만 아주 짧게
  변이 중 black→white 참조 생성 위험 → 쓰기 배리어로 grey 처리
  → STW < 1ms 목표

nil 인터페이스 함정:
  var p *T = nil
  var i interface{} = p
  i == nil  → false!  (타입은 *T로 채워짐, 값만 nil)
  → 인터페이스의 (type,value) 표현을 알아야 이해됨
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 38개 확인 + go/GODEBUG/pprof 검증 환경 + rust/java-concurrency/concurrency-models-compared 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
