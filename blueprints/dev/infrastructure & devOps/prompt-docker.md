# Docker Deep Dive 레포지토리 제작 프롬프트

나는 "Docker Deep Dive" 레포지토리를 만들려고 해.
JVM Deep Dive / Spring Boot Internals 등을 완성한 경험을 바탕으로, **Docker를 그냥 쓰는 것에서 완전히 이해하는 것으로 도약** 시키기 위해 Namespaces, Cgroups, Union Filesystem부터 OCI Runtime Spec, containerd, runc, GraalVM Native Image까지 끝까지 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "컨테이너를 실행하는 것을 넘어, Docker가 실제로 어떻게 동작하는지 완전히 이해하기"

**핵심 차별화**:
1. "어떻게 쓰나"가 아닌 **"왜 그렇게 동작하나"** (Linux Kernel Namespace / Cgroup v1 vs v2 / OverlayFS 동작 원리까지)
2. 모든 개념은 직접 실행 가능 + Before/After 비교 + 성능 벤치마크
3. `dockerd` → `containerd` → `runc` 의 3계층 분해와 각자의 책임 추적
4. OCI Image Spec / Runtime Spec 을 표준 문서 수준으로 분석
5. **Docker만 다루지 않음** — Kubernetes로의 전환까지 매끄럽게 연결 (마지막 섹션 K8s Bridge)
6. **실전 프로젝트 섹션** — 단순 토이 예제가 아닌 Full-stack / Database / Reverse Proxy / Monitoring Stack / ELK 까지 완전 구현

**타겟 독자**:
- `docker run` 은 능숙하게 하지만 컨테이너가 어떻게 격리되는지 모르는 개발자
- "컨테이너는 가벼운 VM" 이라는 비유 너머의 진짜 차이를 알고 싶은 개발자
- Cgroup v1과 v2의 차이, OverlayFS의 Copy-on-Write가 어떻게 동작하는지 궁금한 개발자
- Multi-stage Build, Distroless, Scratch 이미지를 적재적소에 활용하고 싶은 개발자
- 컨테이너 네트워킹 (veth, bridge, iptables) 패킷 흐름을 직접 추적하고 싶은 개발자
- containerd / runc 를 독립적으로 사용해보고 싶은 개발자
- Docker Swarm 을 거쳐 Kubernetes로 전환하려는 개발자
- 프로덕션 환경에서 컨테이너 보안 (Seccomp, AppArmor, Capabilities) 을 설정해야 하는 SRE/DevOps

**기존 레포와의 관계**:
- **Linux for Backend** : Namespace / Cgroup / iptables 같은 커널 레벨 메커니즘 — Docker 의 핵심 토대
- **JVM Deep Dive** : 컨테이너 환경에서 JVM 메모리 인식·CPU 인식 (`-XX:MaxRAMPercentage` 등)
- **Network** : TCP/IP 위에 컨테이너가 만드는 추가 레이어 (veth, bridge, VXLAN 등)
- **Kubernetes** (별도 레포) : 이 레포의 마지막 섹션이 자연스럽게 연결되는 다음 단계
- **Spring Boot Internals** : Layered JAR / Buildpacks 가 컨테이너 패키징과 만나는 지점

---

## 🎯 1단계: 전체 구조 (이미 확정 — 변경 금지)

이 레포는 **13개 섹션, 총 93개 챕터**로 구성된다. 섹션·챕터 제목·핵심 내용은 아래에 고정되어 있으며 추가/삭제/순서 변경 없이 이대로 작성한다.

### 🔹 Section 1: Fundamentals — 핵심 기초 (7개 챕터)
> Docker의 근본 원리를 완전히 이해하기

| 주제 | 핵심 내용 |
|------|----------|
| [01. Container vs VM](./fundamentals/01-Container-vs-VM.md) | 컨테이너와 VM의 근본적 차이, 성능 비교 실험 |
| [02. Docker Architecture](./fundamentals/02-Docker-Architecture.md) | dockerd, containerd, runc 구조, 컴포넌트 통신 흐름 |
| [03. Image Layers](./fundamentals/03-Image-Layers.md) | 레이어 시스템, Copy-on-Write, 캐싱 전략 |
| [04. Union Filesystem](./fundamentals/04-Union-Filesystem.md) | OverlayFS 동작 원리, 스토리지 드라이버 비교 |
| [05. Namespaces](./fundamentals/05-Namespaces.md) | 7가지 Namespace, 격리 메커니즘, 실전 활용 |
| [06. Cgroups](./fundamentals/06-Cgroups.md) | CPU/메모리/I/O 제한, OOM Killer, 리소스 관리 |
| [07. Docker Engine](./fundamentals/07-Docker-Engine.md) | 이벤트 시스템, 플러그인, Engine API |

### 🔹 Section 2: Images — 이미지 심화 (7개 챕터)
> 효율적이고 안전한 이미지 빌드

| 주제 | 핵심 내용 |
|------|----------|
| [01. Dockerfile Best Practices](./images/01-Dockerfile-Best-Practices.md) | 레이어 최적화, 빌드 컨텍스트, 캐시 활용 |
| [02. Multi-Stage Builds](./images/02-Multi-Stage-Builds.md) | 빌드/실행 분리, 이미지 크기 최소화 |
| [03. Image Optimization](./images/03-Image-Optimization.md) | Alpine vs Distroless, 불필요한 파일 제거 |
| [04. Cache Mechanism](./images/04-Cache-Mechanism.md) | 빌드 캐시 동작, 무효화 조건, 원격 캐시 |
| [05. BuildKit](./images/05-BuildKit.md) | 병렬 빌드, Secrets, SSH 마운트 |
| [06. Image Security](./images/06-Image-Security.md) | 취약점 스캔, 서명, 최소 권한 |
| [07. Custom Base Images](./images/07-Custom-Base-Images.md) | scratch부터 시작, 맞춤형 베이스 제작 |

### 🔹 Section 3: Networking — 네트워킹 완전 정복 (9개 챕터)
> 컨테이너 네트워킹의 모든 것

| 주제 | 핵심 내용 |
|------|----------|
| [01. Network Fundamentals](./networking/01-Network-Fundamentals.md) | veth pair, bridge, iptables, 패킷 흐름 |
| [02. Bridge Network](./networking/02-Bridge-Network.md) | 기본 네트워크, 사용자 정의 bridge, DNS |
| [03. Host Network](./networking/03-Host-Network.md) | Host 모드, 성능 특성, 사용 시나리오 |
| [04. Overlay Network](./networking/04-Overlay-Network.md) | 멀티 호스트 네트워킹, VXLAN, Swarm |
| [05. Macvlan Network](./networking/05-Macvlan-Network.md) | 물리 네트워크 통합, VLAN 태깅 |
| [06. Custom Networks](./networking/06-Custom-Networks.md) | CNI 플러그인, Calico, Weave |
| [07. DNS Resolution](./networking/07-DNS-Resolution.md) | 내부 DNS, 서비스 디스커버리 |
| [08. Load Balancing](./networking/08-Load-Balancing.md) | 내장 LB, 헬스 체크, 외부 연동 |
| [09. Network Security](./networking/09-Network-Security.md) | 네트워크 정책, 방화벽, 암호화 |

### 🔹 Section 4: Storage — 스토리지 & 데이터 관리 (7개 챕터)
> 영속적 데이터 관리 전략

| 주제 | 핵심 내용 |
|------|----------|
| [01. Volume Types](./storage/01-Volume-Types.md) | Named Volume, Anonymous Volume 비교 |
| [02. Bind Mounts](./storage/02-Bind-Mounts.md) | 호스트 디렉토리 마운트, 개발 환경 |
| [03. Tmpfs Mounts](./storage/03-Tmpfs-Mounts.md) | 메모리 기반 스토리지, 임시 데이터 |
| [04. Volume Drivers](./storage/04-Volume-Drivers.md) | NFS, GlusterFS, Ceph 통합 |
| [05. Storage Drivers](./storage/05-Storage-Drivers.md) | overlay2, btrfs, zfs 상세 비교 |
| [06. Data Persistence](./storage/06-Data-Persistence.md) | 데이터베이스 영속성 전략 |
| [07. Backup & Restore](./storage/07-Backup-Restore.md) | 백업 자동화, 재해 복구 |

### 🔹 Section 5: Orchestration — 오케스트레이션 (7개 챕터)
> 컨테이너 편성과 관리

| 주제 | 핵심 내용 |
|------|----------|
| [01. Docker Compose](./orchestration/01-Docker-Compose.md) | 멀티 컨테이너 앱, YAML 작성법 |
| [02. Compose Advanced](./orchestration/02-Compose-Advanced.md) | extends, profiles, 환경 분리 |
| [03. Docker Swarm](./orchestration/03-Docker-Swarm.md) | 클러스터 구성, 매니저/워커 노드 |
| [04. Swarm Services](./orchestration/04-Swarm-Services.md) | 서비스 배포, 레플리카, 제약 조건 |
| [05. Swarm Networking](./orchestration/05-Swarm-Networking.md) | Ingress 네트워크, 서비스 메시 |
| [06. Rolling Updates](./orchestration/06-Rolling-Updates.md) | 무중단 배포, 롤백 전략 |
| [07. High Availability](./orchestration/07-High-Availability.md) | 고가용성 아키텍처, 장애 복구 |

### 🔹 Section 6: Security — 보안 강화 (8개 챕터)
> 프로덕션 보안 베스트 프랙티스

| 주제 | 핵심 내용 |
|------|----------|
| [01. Security Principles](./security/01-Security-Principles.md) | 최소 권한, 심층 방어, 공격 표면 |
| [02. Image Scanning](./security/02-Image-Scanning.md) | Trivy, Clair, Anchore 활용 |
| [03. Runtime Security](./security/03-Runtime-Security.md) | Seccomp, AppArmor, Capabilities |
| [04. Secrets Management](./security/04-Secrets-Management.md) | Docker Secrets, Vault 통합 |
| [05. AppArmor & SELinux](./security/05-AppArmor-SELinux.md) | MAC 시스템, 프로파일 작성 |
| [06. User Namespaces](./security/06-User-Namespaces.md) | UID 재매핑, rootless 컨테이너 |
| [07. Security Scanning Tools](./security/07-Security-Scanning-Tools.md) | 자동화된 보안 스캔 파이프라인 |
| [08. Compliance](./security/08-Compliance.md) | CIS 벤치마크, PCI-DSS, HIPAA |

### 🔹 Section 7: Performance — 성능 최적화 (8개 챕터)
> 컨테이너 성능 극대화

| 주제 | 핵심 내용 |
|------|----------|
| [01. Resource Limits](./performance/01-Resource-Limits.md) | CPU/메모리 제한 전략 |
| [02. CPU Management](./performance/02-CPU-Management.md) | CPU pinning, NUMA 인식 |
| [03. Memory Management](./performance/03-Memory-Management.md) | 메모리 누수 탐지, OOM 대응 |
| [04. I/O Performance](./performance/04-IO-Performance.md) | 디스크 I/O 최적화, 벤치마킹 |
| [05. Monitoring](./performance/05-Monitoring.md) | Prometheus, cAdvisor 통합 |
| [06. Logging](./performance/06-Logging.md) | 중앙화 로깅, ELK 스택 |
| [07. Profiling](./performance/07-Profiling.md) | 애플리케이션 프로파일링 |
| [08. Benchmarking](./performance/08-Benchmarking.md) | 성능 테스트 방법론 |

### 🔹 Section 8: Advanced — 고급 주제 (8개 챕터)
> Docker 내부 깊숙이

| 주제 | 핵심 내용 |
|------|----------|
| [01. Container Runtime](./advanced/01-Container-Runtime.md) | OCI Runtime Spec 상세 |
| [02. OCI Specification](./advanced/02-OCI-Specification.md) | Image/Runtime Spec 분석 |
| [03. containerd](./advanced/03-containerd.md) | containerd 독립 사용 |
| [04. runc](./advanced/04-runc.md) | runc 직접 제어 |
| [05. Docker API](./advanced/05-Docker-API.md) | REST API 완전 정복 |
| [06. Docker SDK](./advanced/06-Docker-SDK.md) | Python/Go SDK 활용 |
| [07. Custom Plugins](./advanced/07-Custom-Plugins.md) | 플러그인 개발 가이드 |
| [08. Docker Extensions](./advanced/08-Docker-Extensions.md) | Docker Desktop 확장 |

### 🔹 Section 9: Patterns — 실전 패턴 (8개 챕터)
> 프로덕션 검증된 디자인 패턴

| 주제 | 핵심 내용 |
|------|----------|
| [01. Microservices](./patterns/01-Microservices.md) | 마이크로서비스 아키텍처 |
| [02. Sidecar Pattern](./patterns/02-Sidecar-Pattern.md) | 사이드카 컨테이너 활용 |
| [03. Ambassador Pattern](./patterns/03-Ambassador-Pattern.md) | 프록시 패턴 구현 |
| [04. Adapter Pattern](./patterns/04-Adapter-Pattern.md) | 레거시 시스템 통합 |
| [05. Init Containers](./patterns/05-Init-Containers.md) | 초기화 컨테이너 패턴 |
| [06. Health Checks](./patterns/06-Health-Checks.md) | 헬스 체크 전략 |
| [07. Graceful Shutdown](./patterns/07-Graceful-Shutdown.md) | 우아한 종료 처리 |
| [08. Configuration Management](./patterns/08-Configuration-Management.md) | 설정 관리 베스트 프랙티스 |

### 🔹 Section 10: CI/CD — 지속적 통합/배포 (7개 챕터)
> Docker 기반 CI/CD 파이프라인

| 주제 | 핵심 내용 |
|------|----------|
| [01. Docker in CI](./cicd/01-Docker-in-CI.md) | GitHub Actions, GitLab CI |
| [02. Image Tagging](./cicd/02-Image-Tagging.md) | 태깅 전략, 버저닝 |
| [03. Registry Setup](./cicd/03-Registry-Setup.md) | Private Registry 구축 |
| [04. Automated Testing](./cicd/04-Automated-Testing.md) | 컨테이너 기반 테스트 |
| [05. Security Scanning](./cicd/05-Security-Scanning.md) | 파이프라인 보안 스캔 |
| [06. GitOps](./cicd/06-GitOps.md) | Git 기반 배포 자동화 |
| [07. Deployment Strategies](./cicd/07-Deployment-Strategies.md) | Blue/Green, Canary |

### 🔹 Section 11: Debugging — 디버깅 & 트러블슈팅 (6개 챕터)
> 실전 문제 해결 가이드

| 주제 | 핵심 내용 |
|------|----------|
| [01. Debugging Techniques](./debugging/01-Debugging-Techniques.md) | strace, nsenter, 컨테이너 진입 |
| [02. Log Analysis](./debugging/02-Log-Analysis.md) | 로그 분석 기법 |
| [03. Network Debugging](./debugging/03-Network-Debugging.md) | tcpdump, 네트워크 추적 |
| [04. Performance Issues](./debugging/04-Performance-Issues.md) | 성능 병목 찾기 |
| [05. Common Problems](./debugging/05-Common-Problems.md) | 자주 발생하는 문제들 |
| [06. Diagnostic Tools](./debugging/06-Diagnostic-Tools.md) | 진단 도구 모음 |

### 🔹 Section 12: Real World — 실전 프로젝트 (7개 챕터)
> 실무 프로젝트 완전 구현

| 주제 | 핵심 내용 |
|------|----------|
| [01. Web Application](./real-world/01-Web-Application.md) | Full-stack 앱 Docker화 |
| [02. Database Setup](./real-world/02-Database-Setup.md) | 데이터베이스 컨테이너화 |
| [03. Reverse Proxy](./real-world/03-Reverse-Proxy.md) | Nginx/Traefik 설정 |
| [04. Monitoring Stack](./real-world/04-Monitoring-Stack.md) | Prometheus + Grafana |
| [05. Log Aggregation](./real-world/05-Log-Aggregation.md) | ELK/EFK 스택 구축 |
| [06. Backup System](./real-world/06-Backup-System.md) | 자동 백업 시스템 |
| [07. Multi-Tier App](./real-world/07-Multi-Tier-App.md) | 다층 아키텍처 구현 |

### 🔹 Section 13: Kubernetes Bridge — K8s로의 전환 (4개 챕터)
> Docker에서 Kubernetes로

| 주제 | 핵심 내용 |
|------|----------|
| [01. Docker to K8s](./kubernetes-bridge/01-Docker-to-K8s.md) | 개념 매핑, 차이점 이해 |
| [02. Pod Concepts](./kubernetes-bridge/02-Pod-Concepts.md) | Pod vs 컨테이너 |
| [03. Deployment Patterns](./kubernetes-bridge/03-Deployment-Patterns.md) | Deployment, StatefulSet |
| [04. Migration Guide](./kubernetes-bridge/04-Migration-Guide.md) | 마이그레이션 전략 |

**총 챕터 수: 7+7+9+7+7+8+8+8+8+7+6+7+4 = 93개 (확정, README에 "80+" 로 표기)**

---

## 📐 챕터 문서 구조 (모든 챕터 동일 적용)

```markdown
## 🎯 학습 목표
이 챕터에서 배울 핵심 내용 (3~5개의 구체적 항목).

## 📌 왜 중요한가
실무에서의 중요성과 적용 사례. "이 메커니즘을 모르면 어떤 실수를 하게 되는가" 를 구체적으로.

## 🔬 Deep Dive — 내부 동작 원리
커널 / 컨테이너 런타임 / 표준 명세 (OCI / CNI / CRI) 레벨까지 내려가는 상세 설명.
- 시각화: 패킷 흐름 / 파일 시스템 레이어 / 프로세스 트리 등 mermaid / ASCII art
- 가능한 모든 곳에서 `strace`, `nsenter`, `unshare`, `lsns`, `ip netns`, `mount` 등 도구 출력으로 증명

## 💻 실습
복사-붙여넣기 즉시 실행 가능한 명령어와 코드.
- 단계별 실행 → 결과 확인 → 변형 실험 흐름
- Linux Ubuntu 22.04 기준으로 통일

## 🔥 실전 적용
프로덕션 시나리오에서 어떻게 쓰이는가.
- 실제 서비스에서 마주치는 문제 + 해결책
- Spring Boot / Node.js / Python 앱과의 통합 예시

## ⚡ 최적화 (Before/After)
성능 / 보안 / 효율성 개선 팁.
- 측정 가능한 지표로 비교: 이미지 크기, 시작 시간, 메모리 사용량, 보안 스캔 결과 등
- "10MB 이미지 → 2MB", "시작 5초 → 0.5초" 같은 구체적 수치

## 🚫 안티패턴
피해야 할 실수들.
- "왜 root 로 컨테이너를 실행하면 안 되는가"
- "왜 latest 태그를 쓰면 안 되는가"
- "왜 ENV 로 비밀번호를 넘기면 안 되는가"
각 항목에 그 결과 (보안 사고, 디버깅 지옥, 성능 저하 등) 명시

## 🎓 핵심 정리
한 화면 요약. 5~7줄.

## 🤔 생각해볼 문제 (+ 해설)
이 챕터를 진정으로 이해했는지 검증하는 응용 문제 2~3개와 그 해설.
```

---

## 🎨 스타일 가이드

1. **모든 주장은 명령어 출력으로 증명** — 글로 설명하지 말고 `docker inspect`, `nsenter`, `strace`, `lsns`, `ip netns`, `tcpdump`, `iptables -L`, `mount`, `cat /proc/.../status` 등으로 보여주기
2. **Linux Kernel 메커니즘이 핵심** — Namespace 7가지(PID, NET, MNT, IPC, UTS, USER, CGROUP) / Cgroup v1 vs v2 / OverlayFS / iptables 는 Docker 의 토대이므로 절대 추상화하지 않음
3. **OCI 표준 인용** — Container Runtime / Image Spec 챕터는 OCI 공식 명세 (github.com/opencontainers) 를 직접 인용
4. **Before/After + 측정 강제** — 이미지 크기, 시작 시간, 메모리 사용량, 빌드 시간을 모든 최적화 챕터에 측정값 첨부
5. **Ubuntu 22.04 + Docker 24.x + Compose v2** 기준 통일 (Docker Desktop 사용자도 따라할 수 있게 호스트별 차이 명시)
6. **Container Runtime 3계층 (`dockerd` → `containerd` → `runc`) 분해** — 어떤 작업이 어느 계층에서 처리되는지 모든 관련 챕터에 명시
7. **K8s Bridge 섹션은 "K8s 입문" 이 아니라 "Docker 와의 차이" 중심** — 이미 Docker 를 깊이 안 사람이 K8s 로 넘어갈 때 인지부조화를 줄여주는 다리 역할
8. **단순 명령어 나열 금지** — 모든 명령어는 그것이 무엇을 호출하는지 (`docker run` → `dockerd` REST API → containerd gRPC → runc OCI Bundle 생성) 추적

---

## 🛠️ 실습 환경

```bash
# OS: Ubuntu 22.04 LTS (권장)
# Docker: 24.x
# Compose: v2 (docker compose 명령)
# 추가 도구: strace, nsenter, util-linux (unshare, lsns), tcpdump, iptables, jq, dive, hadolint, trivy

# Docker 설치
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER && newgrp docker

# 진단/분석 도구
sudo apt install -y strace tcpdump util-linux jq
brew install dive hadolint trivy   # 또는 각 GitHub 릴리즈에서 받기

# 모니터링/벤치마킹 (Section 7, 12에서 사용)
# - Prometheus, Grafana, cAdvisor, ELK
# - sysbench, fio (I/O 벤치마크)
```

권장 사양: 4 cores / 8GB RAM / 50GB Disk (Real World, Monitoring Stack 섹션에서 필요).

---

## 🎯 2단계: 작업 순서

전체 구조는 위에 이미 확정되어 있으므로:

1. **디렉토리 생성**: bash 명령어로 13개 섹션 폴더 생성
   ```
   fundamentals/
   images/
   networking/
   storage/
   orchestration/
   security/
   performance/
   advanced/
   patterns/
   cicd/
   debugging/
   real-world/
   kubernetes-bridge/
   ```
2. **README.md 작성**:
   - 빠른 시작 배지 13개 (각 섹션의 첫 챕터)
   - "이 프로젝트에 대하여" 박스 + 6개 특징 체크리스트 (원리 / 실습 / 실전 / 아키텍처 + 80+ / 실행 가능 / 원리 기반 등)
   - 13개 섹션 details 토글 + 표 형식 챕터 목록
   - 학습 로드맵 (초급 6주 / 중급 3개월 / 고급 아키텍트 / 빠른 복습 3일)
   - 학습 방법 + 실습 환경 구성 + 추천 학습 순서 (1~4단계)
   - 추천 도구 (Dive, Hadolint, Trivy, Portainer)
3. **섹션별 챕터 작성**:
   - Section 1 부터 순서대로
   - 한 섹션 완성 후 다음으로
   - 각 챕터는 3000~4500 단어 분량 (도구 출력·코드·벤치마크 분량 포함)
   - **모든 챕터는 위 9섹션 구조 (학습 목표 → 생각해볼 문제) 를 준수**
   - **Before/After + 도구 출력 증거 + 안티패턴 모두 강제**

---

## 📚 참고 자료

### 공식 문서 / 표준
- **Docker Documentation** — docs.docker.com
- **OCI Specifications** — github.com/opencontainers (Image Spec, Runtime Spec, Distribution Spec)
- **containerd** — containerd.io
- **runc** — github.com/opencontainers/runc
- **Linux man-pages** — namespaces(7), cgroups(7), unshare(1), nsenter(1)

### 커널 메커니즘
- **Linux Programming Interface** (Michael Kerrisk) — Namespace, Cgroup 챕터
- **Linux Kernel Documentation** — Documentation/admin-guide/cgroup-v2.rst

### 도구
- **Dive** — 이미지 레이어 분석
- **Hadolint** — Dockerfile 린터
- **Trivy** — 취약점 스캐너
- **Portainer** — Docker GUI

<JVM Deep Dive README를 여기에 붙여넣기>
<Linux for Backend README를 여기에 붙여넣기 (있다면)>

위 README들의 비주얼 톤 (배지, details 토글, 학습 경로 / 학습 방법 시각화) 유지.

---

## 💡 핵심 분석 대상

### Namespace 직접 만들기 (Section 1)

```bash
# Docker 없이 Linux 커널만으로 컨테이너의 핵심을 재현
$ sudo unshare --pid --uts --net --mount --fork bash

(root@host)# hostname mycontainer    # UTS 격리 — 호스트는 영향 없음
(root@host)# ps -ef                  # PID 격리 — bash 가 PID 1
UID  PID  PPID
root   1     0  bash

(root@host)# ip link                 # NET 격리 — lo 만 존재 (down)
1: lo: <LOOPBACK> ...

# 다른 터미널에서:
$ lsns -t pid                        # 새 namespace 가 정말로 만들어졌는지 확인
        NS TYPE  NPROCS  PID USER COMMAND
4026531836 pid       N   ... 
4026532170 pid       1   ... bash   ← 우리가 만든 것
```

→ "Docker 컨테이너 = Namespace + Cgroup + OverlayFS" 라는 추상적 표현을 직접 손으로 재현해서 진짜 의미 깨닫게 하기

### `docker run` 한 줄의 실제 호출 흐름 (Section 1, 8)

```
$ docker run -d nginx

  ↓ Docker CLI (REST API 호출)
dockerd (Docker Daemon)
  ↓ gRPC
containerd (Container Lifecycle 관리)
  ↓ exec
containerd-shim-runc-v2 (컨테이너 종료 후에도 살아있어 stdout 캡처)
  ↓ exec
runc (OCI Runtime — 실제 컨테이너 생성)
  ↓ 시스템 콜
clone() with CLONE_NEWPID, CLONE_NEWNET, ...
mount() with MS_PRIVATE, OverlayFS
setresuid(), setgroups() 등
execve("/docker-entrypoint.sh", ...)
```

→ 책임이 어떻게 분산되어 있는지, 왜 Kubernetes 에서 dockerd 를 제거하고 containerd 를 직접 쓰는지 이해

### OverlayFS Copy-on-Write (Section 1, 4)

```bash
$ docker run --name test -d nginx
$ docker exec test sh -c 'echo "modified" > /etc/nginx/nginx.conf'
$ docker diff test
C /etc
C /etc/nginx
C /etc/nginx/nginx.conf

# 실제 파일 시스템 확인
$ sudo ls /var/lib/docker/overlay2/<container-id>/diff/etc/nginx/
nginx.conf   ← upper layer 에만 존재. lower layer 는 손대지 않음

# 원본 nginx 이미지의 nginx.conf 는 그대로
$ docker run --rm nginx cat /etc/nginx/nginx.conf
# (원본 내용 출력)
```

→ Copy-on-Write 가 "파일을 수정하면 일어나는 일" 을 직접 보고, 이미지 레이어가 왜 immutable 한지 이해

### 이미지 크기 최적화 Before/After (Section 2)

| 빌드 방식 | 이미지 크기 | 빌드 시간 |
|:----------|:----------:|:--------:|
| 단순 `node:20` 베이스 + 모든 빌드 도구 포함 | 1.2GB | 95s |
| Multi-stage + `node:20-alpine` | 280MB | 80s |
| Multi-stage + Distroless (`gcr.io/distroless/nodejs20`) | 180MB | 78s |
| Static binary + `scratch` 베이스 | 22MB | 60s |

→ 각 단계에서 무엇이 빠졌는지 + 보안 측면 trade-off (Distroless 는 shell 도 없어서 디버깅 어려움 등)

### Cgroup v2 로 컨테이너 리소스 제한 직접 보기 (Section 1, 7)

```bash
$ docker run -d --name limited --cpus=0.5 --memory=256m nginx

# Cgroup v2 인터페이스에서 직접 확인
$ CONTAINER_ID=$(docker inspect -f '{{.Id}}' limited)
$ cat /sys/fs/cgroup/system.slice/docker-$CONTAINER_ID.scope/cpu.max
50000 100000   ← 100ms 당 50ms = 0.5 코어

$ cat /sys/fs/cgroup/system.slice/docker-$CONTAINER_ID.scope/memory.max
268435456      ← 256MB

# 메모리 한계 초과 시
$ docker exec limited stress-ng --vm 1 --vm-bytes 300M
# OOM Killer 발동 → dmesg 에서 추적 가능
$ dmesg | grep -i 'killed process'
```

→ Docker 가 추상화하는 것 (`--cpus`, `--memory`) 의 실체가 Linux Cgroup v2 파일 인터페이스라는 것을 직접 확인

### Kubernetes Bridge — Docker Compose vs K8s Manifest (Section 13)

```yaml
# docker-compose.yml
services:
  web:
    image: nginx:1.25
    ports: ["80:80"]
    deploy:
      replicas: 3
```

```yaml
# kubernetes manifest (등가)
apiVersion: apps/v1
kind: Deployment
metadata: { name: web }
spec:
  replicas: 3
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports: [{ containerPort: 80 }]
---
apiVersion: v1
kind: Service
metadata: { name: web }
spec:
  selector: { app: web }
  ports: [{ port: 80, targetPort: 80 }]
  type: LoadBalancer
```

→ Compose의 한 줄이 K8s 에서 Deployment + Service + Selector 라는 명시적 분리로 펼쳐지는 것 보여주기 + Pod 라는 추가 추상화의 의미

---

## 🎯 진행 방식

**구조는 이미 확정되어 있으므로 1단계는 생략하고 바로 2단계(디렉토리 생성)부터 시작.**

진행 순서:
1. 디렉토리 13개 생성
2. README.md 작성 (빠른 시작 배지 13개 + 학습 로드맵 + 실습 환경 구성)
3. Section 1부터 순서대로 7 → 7 → 9 → 7 → 7 → 8 → 8 → 8 → 8 → 7 → 6 → 7 → 4 = 93개 챕터 작성
4. 각 챕터마다 9섹션 구조 강제 준수
5. Before/After + 도구 출력 증거 + 안티패턴 + 측정값 모두 강제
6. Linux Kernel 메커니즘은 절대 추상화하지 않음 (Namespace / Cgroup / OverlayFS / iptables)

**시작해줘! 디렉토리 생성 후 README, 그다음 Section 1의 1번 챕터 (Container vs VM) 부터.**
