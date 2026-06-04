# Linux for Backend Deep Dive 레포지토리 제작 프롬프트

나는 "Linux for Backend Deep Dive" 레포지토리를 만들려고 해.
리눅스 명령어를 쓰는 것과, 커널이 I/O 요청을 어떻게 처리하고, 메모리를 어떻게 관리하고, 프로세스를 어떻게 스케줄링하는지 아는 것의 차이를 만드는 레포야.

단순한 리눅스 명령어 튜토리얼이 아니야.
MySQL이 왜 Sequential Read가 빠른지, Redis가 왜 fork()로 스냅샷을 찍는지, Docker가 왜 namespace와 cgroups인지, Spring WebFlux가 왜 적은 스레드로 더 많은 요청을 처리하는지 — **이 모든 것의 기저에 있는 커널 동작 원리**를 파헤치는 레포야.

## 📋 프로젝트 목표

**컨셉**: "리눅스에 배포하는 것과, 커널이 I/O와 메모리를 어떻게 관리해 내 애플리케이션 성능을 결정하는지 아는 것은 다르다"

**핵심 차별화**:
1. I/O 모델 — Blocking에서 epoll까지, OS 레벨에서 어떻게 진화했는가
2. 메모리 관리 — Page Cache가 왜 MySQL/Redis/Kafka 성능의 열쇠인가
3. 프로세스 vs 스레드 — fork()의 Copy-On-Write가 Redis RDB 백업을 어떻게 가능하게 하는가
4. cgroups & namespace — Docker 컨테이너의 실제 구현 기반

**타겟 독자**:
- 리눅스 서버에 배포하지만 커널이 무슨 일을 하는지 모르는 백엔드 개발자
- MySQL `innodb_buffer_pool_size`를 설정하지만 Page Cache와의 관계를 모르는 개발자
- Redis가 `BGSAVE` 중에도 Write를 받는 이유를 설명 못하는 개발자
- `epoll`이라는 단어를 들었지만 Blocking I/O와 뭐가 다른지 모르는 개발자
- Docker를 쓰지만 namespace와 cgroups가 무엇인지 모르는 개발자
- Spring WebFlux나 Virtual Thread가 왜 등장했는지 OS 관점에서 설명 못하는 개발자
- `top`, `vmstat`, `iostat`을 보지만 수치가 무엇을 의미하는지 모르는 개발자

**선행 학습**:
- 없음 (백엔드 개발 경험만 있으면 충분)
- **이 레포가 선행**이 되어야 하는 레포: docker-deep-dive, network-deep-dive, redis-deep-dive, kafka-deep-dive

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 프로세스와 스레드 — 실행의 기본 단위 (6개 문서)
- 프로세스란 무엇인가 — 프로세스 주소 공간(코드/데이터/힙/스택), PCB, 컨텍스트 스위칭의 실체
- fork()와 exec() — 자식 프로세스 생성 원리, Copy-On-Write가 Redis BGSAVE를 가능하게 하는 방식
- 스레드 모델 — OS 스레드와 커널의 1:1 매핑, 스레드 생성 비용이 프로세스보다 작은 이유
- 컨텍스트 스위칭 비용 — CPU 레지스터 저장/복원, TLB 플러시, 캐시 무효화가 실제 성능에 미치는 영향
- 시그널과 IPC — SIGKILL vs SIGTERM 차이, Pipe/FIFO/Message Queue/Shared Memory 비교
- 프로세스 스케줄러 — CFS(Completely Fair Scheduler) 동작 원리, nice 값과 우선순위, CPU 어피니티

### Chapter 2: 메모리 관리 — 가상 주소 공간의 실체 (6개 문서)
- 가상 메모리 — 물리 메모리와 가상 주소 공간의 분리, Page Table과 TLB의 역할
- Page Fault — Minor vs Major Page Fault, 처음 접근 시 물리 메모리가 할당되는 과정
- Page Cache — 파일 읽기가 디스크가 아닌 메모리에서 되는 원리, MySQL/Kafka가 Page Cache를 활용하는 방식
- mmap과 Direct I/O — mmap이 Page Cache와 연결되는 방식, O_DIRECT로 Page Cache를 우회하는 이유
- 메모리 단편화와 할당 — ptmalloc/jemalloc/tcmalloc 비교, Redis와 JVM이 다른 할당기를 쓰는 이유
- OOM Killer — Out-Of-Memory 발생 조건, oom_score 계산 방식, OOM이 Redis/Java를 죽이는 시나리오

### Chapter 3: I/O 모델의 진화 — Blocking에서 epoll까지 (6개 문서)
- 파일 디스크립터와 시스템 콜 — fd가 무엇인가, read()/write() 시스템 콜이 커널에서 하는 일
- Blocking I/O — read() 호출 시 프로세스가 대기하는 원리, 스레드 1개 = 연결 1개의 한계
- Non-Blocking I/O와 Polling — O_NONBLOCK 설정, busy-wait의 CPU 낭비 문제
- I/O Multiplexing(select/poll) — 하나의 스레드로 여러 fd를 감시하는 방식, O(N) 스캔의 한계
- epoll — 이벤트 기반 통지, 레벨 트리거 vs 엣지 트리거, O(1) 이벤트 처리가 가능한 이유
- I/O 모델이 백엔드에 미치는 영향 — Tomcat(스레드 모델) vs Netty/WebFlux(이벤트 루프), Redis 단일 스레드의 비밀

### Chapter 4: 파일 시스템과 디스크 I/O (5개 문서)
- VFS(Virtual File System) — 리눅스가 ext4/XFS/tmpfs를 동일 인터페이스로 다루는 방식
- 디스크 I/O 스택 — 애플리케이션 → VFS → 파일시스템 → 블록 레이어 → 디바이스 드라이버 전 과정
- Sequential vs Random I/O — HDD/SSD에서 각각 성능 차이가 나는 원리, MySQL/Kafka가 Sequential Write를 선호하는 이유
- fsync와 내구성 — 커널 버퍼에 쓴 데이터가 디스크에 반영되는 시점, MySQL `innodb_flush_method` 설정의 의미
- inode와 파일 시스템 구조 — inode가 파일 메타데이터를 저장하는 방식, 파일 삭제가 즉시 공간을 해제하지 않는 이유

### Chapter 5: 네트워크 스택 — 커널에서 소켓까지 (5개 문서)
- 소켓과 커널 버퍼 — send()/recv()가 실제로 하는 일, 커널의 송신/수신 버퍼(sk_buff)
- TCP 소켓 상태와 커널 — Listen Backlog, Accept Queue, SYN Backlog가 고트래픽 서버에서 중요한 이유
- 소켓 옵션 — TCP_NODELAY(Nagle 알고리즘 비활성), SO_KEEPALIVE, SO_REUSEADDR, SO_REUSEPORT
- 커널 네트워크 파라미터 튜닝 — net.core.somaxconn, tcp_max_syn_backlog, net.ipv4.tcp_tw_reuse
- 제로 카피(Zero-copy) — sendfile() 시스템 콜로 커널 버퍼→유저 버퍼 복사 없이 데이터 전송, Kafka/Nginx가 활용하는 원리

### Chapter 6: cgroups와 namespace — 컨테이너의 실체 (5개 문서)
- Linux namespace — PID/Network/Mount/UTS/IPC namespace가 컨테이너 격리를 만드는 방식
- cgroups v2 — CPU/메모리/I/O 자원 제한 구현, Docker `--memory`/`--cpus` 옵션의 실제 동작
- 컨테이너 네트워크 — veth pair, bridge 인터페이스, iptables NAT 규칙이 Docker 네트워킹을 만드는 과정
- OOM in Containers — cgroups 메모리 한도 초과 시 OOM Killer 동작, JVM Heap과 컨테이너 메모리 한도의 관계
- seccomp와 capabilities — 컨테이너의 시스템 콜 제한, root-less 컨테이너가 가능한 이유

### Chapter 7: 성능 진단 — 도구와 방법론 (5개 문서)
- CPU 분석 — `top`/`htop`/`mpstat`으로 CPU 사용률 해석, us/sy/wa/id 각 항목의 의미
- 메모리 분석 — `free`/`vmstat`으로 메모리 사용량 해석, cache/buffer의 실체, `/proc/meminfo` 읽기
- I/O 분석 — `iostat`/`iotop`으로 디스크 I/O 병목 찾기, await와 %util의 의미
- 네트워크 분석 — `ss`/`netstat`으로 소켓 상태 확인, `sar`로 네트워크 트래픽 측정, TIME_WAIT 누적 진단
- strace와 perf — 시스템 콜 추적으로 애플리케이션 I/O 패턴 분석, CPU Flame Graph로 핫스팟 찾기

---

각 챕터는 **5~6개 문서**로 구성해줘. 총 **38개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 실무에서 중요한가
## 😱 흔한 실수 (Before — 커널을 모를 때의 접근)
## ✨ 올바른 접근 (After — 커널을 알고 난 설계/운영)
## 🔬 내부 동작 원리 (커널 레벨 분석, /proc 파일 시스템, strace 결과)
## 💻 실전 실험 (재현 가능한 시나리오, 성능 진단 명령어)
## 📊 성능/비용 비교 (Blocking vs Non-Blocking, Page Cache 유무 등)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 추상적 설명 금지 — 모든 원리는 "커널이 실제로 무엇을 하는가"로 연결
2. `/proc` 파일 시스템으로 커널 상태를 직접 확인하는 실험 포함 (`/proc/meminfo`, `/proc/net/tcp` 등)
3. `strace`로 시스템 콜을 추적해 애플리케이션이 커널과 어떻게 상호작용하는지 확인
4. 기존 레포와 연결 — MySQL Page Cache, Redis fork(), Kafka Zero-copy, Docker cgroups와의 연결을 명시
5. 운영 장애 시나리오 → 진단 명령어 세트 (어떤 증상에 어떤 도구를 쓰는가)

**실험 환경**:
```yaml
# docker-compose.yml
services:
  lab:
    image: ubuntu:22.04
    privileged: true          # 커널 기능 실험용
    pid: host                 # 호스트 PID namespace 공유
    volumes:
      - /proc:/host-proc:ro   # 호스트 /proc 마운트
      - /sys:/host-sys:ro
    command: /bin/bash
    stdin_open: true
    tty: true

  resource-monitor:
    image: nicolargo/glances
    pid: host
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - "61208:61208"
```

```bash
# 실험용 공통 명령어 세트

# 프로세스/스레드
strace -c -p <pid>           # 시스템 콜 통계
cat /proc/<pid>/status       # 프로세스 상태
cat /proc/<pid>/maps         # 가상 주소 공간 매핑
ls -la /proc/<pid>/fd/       # 열린 파일 디스크립터

# 메모리
cat /proc/meminfo            # 시스템 메모리 전체 현황
cat /proc/<pid>/smaps        # 프로세스 메모리 세부 사용
vmstat 1 10                  # 메모리/swap/I/O 1초 간격 10회

# I/O
iostat -xz 1                 # 디스크 I/O 통계 (await, %util)
iotop -bo                    # 프로세스별 실시간 I/O
strace -e trace=read,write,open,close -p <pid>  # I/O 시스템 콜 추적

# 네트워크 소켓
ss -tanp                     # 모든 TCP 소켓 상태
ss -s                        # 소켓 통계 요약
cat /proc/net/tcp            # TCP 연결 상태 raw 데이터

# cgroups (컨테이너 실험)
cat /sys/fs/cgroup/memory/docker/<id>/memory.usage_in_bytes
cat /sys/fs/cgroup/cpu/docker/<id>/cpuacct.usage
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**:
   - mysql-deep-dive README 스타일 참고
   - "모든 백엔드 레포의 기저가 되는 커널 원리"라는 포지셔닝 강조
   - 기존 레포(MySQL, Redis, Kafka, Docker)와의 연결 지점 명시
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료 섹션

README 작성 시 다음을 Reference로 포함해줘:
- Linux Kernel Development, 3rd Edition (Robert Love)
- The Linux Programming Interface (Michael Kerrisk)
- Systems Performance: Enterprise and the Cloud, 2nd Edition (Brendan Gregg) — https://www.brendangregg.com/
- Understanding the Linux Kernel (Bovet & Cesati)
- Linux man pages — https://man7.org/linux/man-pages/
- Brendan Gregg's Blog — http://www.brendangregg.com/blog/

---

## 💡 핵심 분석 대상

```
I/O 처리 흐름 (백엔드 개발자 관점):

Java 애플리케이션 (Spring MVC)
  │
  ▼ ServerSocket.accept() — Blocking 대기
  │   ↳ 커널: 새 연결이 Accept Queue에서 dequeue
  │
  ▼ InputStream.read() — Blocking 대기
  │   ↳ 커널: 수신 버퍼에 데이터 올 때까지 프로세스 Sleep
  │   ↳ 데이터 도착 → 프로세스 Wake-up → 사용자 공간으로 복사
  │
  ▼ 처리 로직 실행
  │
  ▼ OutputStream.write() — 커널 송신 버퍼에 복사
      ↳ 커널: 나중에 비동기로 네트워크 전송

Spring WebFlux (Netty 기반):
  이벤트 루프 스레드 (소수)
    │
    ▼ epoll_wait() — 이벤트 감시
    │   ↳ 커널: 이벤트 발생 시 해당 fd만 통지 (O(1))
    │
    ▼ 콜백 실행 (데이터 준비된 연결만)
    │
    ▼ 처리 완료 후 즉시 다음 이벤트 처리

Tomcat vs Netty 비교:
  Tomcat: 연결 1개 = 스레드 1개
    → 1000 동시 연결 = 1000 스레드
    → 컨텍스트 스위칭 비용 × 1000
  Netty: 이벤트 루프 스레드 8개
    → 1000 동시 연결 = 8 스레드
    → CPU 코어 수에 맞춰 최적화

Redis BGSAVE 동작:
  BGSAVE 명령 → fork() 호출
    │
    ├── 부모 프로세스: 계속 클라이언트 요청 처리
    │   └── 쓰기 발생 시 → Copy-On-Write로 해당 페이지만 복사
    │
    └── 자식 프로세스: 포크 시점의 스냅샷 직렬화 → .rdb 파일 저장
        └── 자식은 부모의 가상 주소 공간을 공유(읽기 전용)
            → 쓰기가 없으면 추가 메모리 0 사용!

Page Cache와 MySQL:
  InnoDB Buffer Pool: 자체 메모리 캐시 (innodb_buffer_pool_size)
  OS Page Cache: 파일 시스템 레벨 캐시
  
  innodb_flush_method = O_DIRECT:
    → Page Cache 우회, InnoDB가 직접 디스크 관리
    → 이중 캐싱 방지 (Buffer Pool ≠ Page Cache)
  
  innodb_flush_method = fsync (기본):
    → Page Cache 거쳐서 쓰기
    → Buffer Pool 크기 + Page Cache 크기만큼 메모리 사용

cgroups가 Docker를 만드는 방식:
  docker run --memory=512m --cpus=1.5 myapp
    │
    ├── /sys/fs/cgroup/memory/docker/<id>/memory.limit_in_bytes = 536870912
    ├── /sys/fs/cgroup/cpu/docker/<id>/cpu.cfs_quota_us = 150000
    └── 프로세스를 해당 cgroup에 추가
        → 커널이 자동으로 자원 제한 적용
        → 초과 시: OOM Kill (메모리) 또는 Throttling (CPU)

OOM 발생과 JVM:
  컨테이너 메모리 한도: 512MB
  JVM Heap (-Xmx): 1024MB (잘못된 설정)
    → JVM이 1GB Heap 요청
    → cgroups 메모리 한도 초과
    → OOM Killer가 JVM 프로세스 종료
    → "Killed" 메시지만 남고 스택 트레이스 없음
  
  올바른 설정:
    -XX:+UseContainerSupport (Java 10+, cgroups 인식)
    -XX:MaxRAMPercentage=75.0 (컨테이너 메모리의 75%를 Heap으로)
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~6개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (38개 목표)
- 실험 환경 (Docker + privileged 컨테이너 기반)
- 기존 레포(MySQL/Redis/Kafka/Docker)와 연결되는 지점 명시 (어느 문서에서 어느 레포와 연결하는지)

**준비됐으면 1단계 구조 설계부터 시작해줘!**
