# Computer Architecture Deep Dive 레포지토리 제작 프롬프트

나는 "Computer Architecture Deep Dive" 레포지토리를 만들려고 해.
고수준 코드가 결국 CPU 위에서 어떻게 실행되는지 — 파이프라인, 캐시 계층, 메모리 모델, 분기 예측이 성능을 어떻게 결정하는지를 완전히 파헤치는 레포야.
"코드가 동작한다"와 "이 코드가 왜 이 하드웨어에서 이 속도인지 안다"의 차이를 만드는 레포다. 이 연구소의 **모든 성능 논의가 결국 수렴하는 바닥**이다.

## 📋 프로젝트 목표

**컨셉**: "코드를 짜는 것과, 그 코드가 CPU·캐시·메모리에서 실제로 무슨 일을 하는지 아는 것은 다르다"

**핵심 차별화**:
1. 캐시가 성능을 지배한다 — 캐시라인·지역성·False Sharing이 같은 알고리즘의 속도를 10배 가르는 원리
2. CPU는 코드를 순서대로 실행하지 않는다 — 파이프라이닝·비순차 실행·분기 예측·투기 실행
3. 메모리 모델은 약속이다 — 하드웨어 재정렬과 메모리 배리어, 동시성 버그의 하드웨어 근원
4. 추상화의 비용 — 가상 함수·포인터 추적·캐시 미스가 만드는 보이지 않는 비용을 수치로 증명

**타겟 독자**:
- 알고리즘 복잡도는 같은데 왜 한 코드가 더 빠른지 설명 못하는 개발자
- JVM Concurrency에서 `volatile`·메모리 배리어를 외웠지만 왜 필요한지 모르는 개발자
- 캐시 친화적 코드가 뭔지 들어봤지만 실제로 측정해본 적 없는 개발자
- 프로파일러가 "캐시 미스"를 가리킬 때 그게 무슨 의미인지 모르는 개발자
- false sharing·NUMA를 단어로만 아는 백엔드/시스템 개발자

## 🔗 레포 연결

**⬆️ 선행**: 없음 — 스택의 최하단. 단, C/어셈블리 읽을 수 있으면 좋다.
**🤝 시너지**: `linux-for-backend-deep-dive`(스케줄러·메모리가 OS 레벨에서 어떻게 노출되나), `jvm-deep-dive`(JIT가 생성하는 기계어), `java-concurrency-deep-dive`(메모리 모델의 하드웨어 근원).
**🧬 수렴**: `memory-management-compared`(메모리 계층이 모든 GC/ARC 비용의 바닥), `performance-testing-deep-dive`(USE 방법론의 "U=Utilization"이 결국 CPU 자원).

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 설계해줘:

### Chapter 1: 실행의 실체 — 명령어와 파이프라인 (6개 문서)
- C 한 줄이 기계어가 되기까지 — 컴파일→어셈블리→기계어, godbolt로 추적, x86-64 기본 명령어 해부
- CPU 파이프라인 — Fetch/Decode/Execute/Memory/Writeback 5단계, 파이프라이닝이 IPC를 올리는 원리
- 파이프라인 해저드 — 데이터/제어/구조 해저드, 스톨·버블, 포워딩으로 해결
- 슈퍼스칼라와 비순차 실행(OoO) — 한 사이클에 여러 명령, 명령어 재정렬, ROB(Reorder Buffer)
- 분기 예측 — 파이프라인을 비우지 않으려는 투기 실행, 2-bit 예측기, 분기 미스 비용 측정
- 명령어 수준 병렬성(ILP)의 한계 — 의존성 사슬, 왜 단일 스레드 성능이 정체되었나

### Chapter 2: 메모리 계층 — 성능을 지배하는 캐시 (6개 문서)
- 메모리 계층 전체 — 레지스터→L1/L2/L3→DRAM→디스크, 각 층의 지연시간(사이클) 체감
- 캐시 동작 원리 — 캐시라인(64B), 태그·인덱스·오프셋, 직접/집합 연관 매핑, 교체 정책(LRU)
- 지역성(Locality) — 시간/공간 지역성, 배열 순회 vs 연결 리스트의 캐시 미스 차이 측정
- 캐시 미스의 종류 — Compulsory/Capacity/Conflict, cachegrind로 미스율 측정
- 데이터 구조와 캐시 — AoS vs SoA, 패딩·정렬, 캐시라인에 맞춘 자료구조 설계
- TLB와 가상 메모리 — 페이지 테이블 워크, TLB 미스, Huge Page가 성능에 미치는 영향

### Chapter 3: 메모리 모델과 동시성의 하드웨어 (6개 문서)
- 멀티코어 캐시 일관성 — MESI 프로토콜, 코어 간 캐시라인 공유와 무효화
- False Sharing — 서로 다른 변수가 같은 캐시라인에 있을 때 생기는 성능 붕괴, 측정·해결(패딩)
- 메모리 재정렬 — 컴파일러·CPU가 명령을 재정렬하는 이유, Store Buffer, x86 vs ARM 메모리 모델
- 메모리 배리어 — Load/Store 배리어, `volatile`·`atomic`이 하드웨어로 번역되는 방식
- 원자적 연산과 CAS — `lock` 프리픽스, Compare-And-Swap의 하드웨어 구현, ABA 문제의 근원
- JMM과의 연결 — Java 메모리 모델의 happens-before가 이 하드웨어 위에 어떻게 세워졌나

### Chapter 4: 추상화의 숨은 비용 (5개 문서)
- 함수 호출 비용 — 스택 프레임, 호출 규약(calling convention), 인라이닝이 없애는 비용
- 가상 함수·동적 디스패치 — vtable 추적, 분기 예측 실패, 다형성의 캐시 비용
- 포인터 추적(Pointer Chasing) — 간접 참조가 캐시 미스를 부르는 구조, 객체 그래프 순회의 비용
- 분기 vs 분기 없는 코드 — branchless 기법, 조건 이동(cmov), 데이터 의존 분기의 비용
- 측정으로 증명 — 같은 로직의 두 구현을 perf로 비교, "추상화 비용"을 사이클로 환산

### Chapter 5: SIMD와 데이터 병렬성 (5개 문서)
- SIMD 기본 — SSE/AVX 레지스터, 한 명령으로 여러 데이터, 벡터화의 원리
- 자동 벡터화 — 컴파일러가 루프를 벡터화하는 조건, 벡터화를 막는 패턴
- 수동 SIMD — 인트린식으로 합/내적 가속, 정렬 요구사항, 측정
- 데이터 레이아웃과 벡터화 — SoA가 벡터화에 유리한 이유, 분기가 벡터화를 깨는 이유
- 처리량 vs 지연시간 관점 — SIMD가 빛나는 워크로드와 그렇지 않은 워크로드

### Chapter 6: 멀티코어와 NUMA (5개 문서)
- 멀티코어 토폴로지 — 코어·L3 공유·소켓 구조, lstopo로 실제 머신 토폴로지 확인
- NUMA — 원격 메모리 접근 비용, NUMA 노드 바인딩, `numactl`로 측정
- 스케일링의 벽 — 암달의 법칙·구스타프슨, 공유 자원 경합이 만드는 비선형 스케일
- 락 경합의 하드웨어 비용 — 캐시라인 핑퐁, 스핀락 vs 뮤텍스의 하드웨어 관점
- 코어 친화도(Affinity) — 스레드 핀닝이 캐시 지역성에 주는 효과

### Chapter 7: 측정과 최적화 방법론 (5개 문서)
- perf 완전 활용 — `perf stat`(IPC·캐시미스·분기미스), `perf record/report`, 하드웨어 카운터
- 마이크로벤치마킹의 함정 — 데드코드 제거·상수 폴딩·워밍업, 신뢰할 수 있는 측정 설계
- 루프라인 모델 — 연산 강도 vs 메모리 대역폭, 내 코드가 compute-bound인가 memory-bound인가
- 최적화 우선순위 — 알고리즘 → 메모리 접근 → 벡터화 → 마이크로 순서가 맞는 이유
- 케이스 스터디 — 느린 코드 하나를 perf로 진단→캐시 친화 개선→사이클 단위 before/after

→ 각 챕터 5~6개 문서. **총 38개 문서** 목표.

## 📄 문서 구조

각 문서는 아래 10섹션. (표준 템플릿 v2 준수)
`🎯 핵심 질문 / 🔍 왜 중요한가 / 😱 Before(추상적으로만 생각) / ✨ After(하드웨어를 알고 설계) / 🔬 내부 동작 원리(어셈블리·캐시 동작) / 💻 실전 실험(perf·cachegrind·godbolt) / 📊 성능 비교(사이클·미스율) / ⚖️ 트레이드오프 / 📌 핵심 정리 / 🤔 생각해볼 문제(+해설)`

## 🎨 스타일 가이드

1. **항상 측정으로 증명** — "캐시 미스가 느리다"가 아니라 perf 출력으로 사이클·미스율을 보여준다
2. **godbolt 링크/스니펫** — 고수준 코드 옆에 생성된 어셈블리를 항상 병기
3. **두 구현 비교** — 캐시 친화 vs 비친화, 분기 vs branchless를 같은 입력으로 before/after
4. **JVM·동시성과 연결** — 이 하드웨어 사실이 `volatile`·`false sharing`·`@Contended`로 어떻게 나타나는지
5. 숫자는 항상 "내 머신 기준"임을 명시하고 측정 환경(CPU 모델·캐시 크기)을 표기

## 🔬 검증 환경

> 이 레포는 docker-compose가 아니라 **로컬 측정 도구**가 핵심이다. 재현성을 위해 Dockerfile 제공.

```dockerfile
# Dockerfile — 측정 도구 일괄 설치 (Linux x86-64 권장)
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y \
    gcc g++ clang \
    linux-tools-generic linux-tools-common \
    valgrind \
    likwid hwloc numactl \
    python3 python3-pip
# perf는 호스트 커널 권한 필요: docker run --privileged 또는 호스트에서 직접 실행 권장
```

```bash
# 핵심 측정 명령어 세트

# IPC·캐시미스·분기미스 한눈에
perf stat -e cycles,instructions,cache-references,cache-misses,branches,branch-misses ./a.out

# 캐시 미스 상세 (라인 단위)
valgrind --tool=cachegrind ./a.out
cg_annotate cachegrind.out.<pid>

# 핫스팟 프로파일
perf record -g ./a.out && perf report

# 머신 토폴로지 / NUMA
lstopo --of console
numactl --hardware
numactl --cpunodebind=0 --membind=0 ./a.out   # vs --membind=1 (원격) 비교

# 어셈블리 확인 (또는 godbolt.org)
gcc -O2 -S -masm=intel demo.c -o demo.s
```

핵심 실험 예: `int[1024][1024]` 행 우선 vs 열 우선 순회의 cache-misses 비교 / `false sharing` 카운터를 패딩 전후로 perf 측정 / 분기 예측 가능 데이터 vs 랜덤 데이터 정렬 후 분기미스 비교.

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더 bash로 생성
2. **README.md**: org README의 🪨 Foundations 톤, "코드가 동작한다 vs 왜 이 속도인지 안다" 포지셔닝, `🔗 레포 연결` 노출
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- *Computer Systems: A Programmer's Perspective* (Bryant & O'Hallaron)
- *What Every Programmer Should Know About Memory* (Ulrich Drepper)
- *Computer Architecture: A Quantitative Approach* (Hennessy & Patterson)
- Agner Fog 최적화 매뉴얼 — https://www.agner.org/optimize/
- Intel 64 and IA-32 Architectures Optimization Reference Manual
- Compiler Explorer — https://godbolt.org/

## 💡 핵심 분석 대상

```
메모리 계층 지연 (체감용, 대략적 사이클):
  레지스터      : 0
  L1 캐시       : ~4 cycles
  L2 캐시       : ~12 cycles
  L3 캐시       : ~40 cycles
  DRAM          : ~200+ cycles   ← L1 대비 50배
  → "캐시 미스 한 번 = 명령 수십 개 실행할 시간"

행 우선 vs 열 우선 (캐시라인 64B = int 16개):
  for i: for j: sum += a[i][j]   → 순차 접근, 캐시라인 재사용 ✓
  for j: for i: sum += a[i][j]   → 매번 새 캐시라인, 미스 폭발 ✗
  같은 O(N²)인데 수 배 차이

False Sharing:
  struct { long a; long b; }  // a, b가 같은 64B 라인
  코어1이 a 쓰기, 코어2가 b 쓰기
    → 서로의 캐시라인을 계속 무효화(MESI) → 캐시라인 핑퐁
  해결: a와 b를 각각 다른 라인으로 패딩(@Contended)

분기 예측:
  if (data[i] >= 128) sum += data[i];
  정렬된 데이터  → 예측 적중 → 빠름
  랜덤 데이터    → 예측 실패 → 파이프라인 플러시 → 느림
  (정렬만 했는데 빨라지는 유명한 현상)
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목(5~6개씩) + 각 문서 핵심 내용(3~4줄) + 총 38개 확인 + 측정 환경(Dockerfile/perf) + 시너지 레포 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
