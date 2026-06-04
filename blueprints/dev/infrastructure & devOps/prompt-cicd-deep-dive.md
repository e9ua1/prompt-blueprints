# CI/CD Pipeline Deep Dive 레포지토리 제작 프롬프트

나는 "CI/CD Pipeline Deep Dive" 레포지토리를 만들려고 해.
배포 버튼을 누르는 것과, GitHub Actions Runner가 어떻게 작업을 실행하고, Docker 레이어 캐시가 빌드를 어떻게 가속하고, ArgoCD가 Kubernetes 상태를 어떻게 수렴시키는지를 완전히 파헤치는 레포야.

"main에 push하면 배포되겠지"와 "파이프라인 단계별 실패 원인을 즉시 파악하고, 롤백 시점과 전략을 설계하며, 카나리 배포로 위험을 최소화하는 것"의 차이를 만드는 레포야.

## 📋 프로젝트 목표

**컨셉**: "배포 스크립트를 실행하는 것과, 파이프라인이 어떻게 코드를 신뢰할 수 있는 결과물로 만드는지 아는 것은 다르다"

**핵심 차별화**:
1. Pipeline 내부 동작 — GitHub Actions Runner가 Job을 어떻게 격리하고 실행하는가
2. 컨테이너 빌드 최적화 — Docker 레이어 캐시가 빌드 시간을 어떻게 단축하는가
3. GitOps 원칙 — ArgoCD가 Git을 Single Source of Truth로 삼아 Kubernetes를 수렴시키는 원리
4. 배포 전략 — Blue-Green/Canary/Rolling의 트레이드오프와 롤백 메커니즘

**타겟 독자**:
- GitHub Actions YAML을 복붙하지만 `uses:` 뒤의 Action이 무엇을 하는지 모르는 개발자
- Docker 이미지를 빌드하지만 레이어 순서가 캐시에 미치는 영향을 모르는 개발자
- `kubectl apply`로 배포하지만 Rolling Update가 중단되는 조건을 모르는 개발자
- 배포 실패 시 원인을 찾는 데 30분 이상 걸리는 개발자
- ArgoCD를 쓰지만 Sync와 Reconciliation이 어떻게 동작하는지 모르는 개발자
- 카나리 배포를 원하지만 트래픽을 어떻게 분할하는지 모르는 개발자

**선행 학습**:
- docker-deep-dive (컨테이너 이미지 레이어, Namespace/Cgroup 이해 필수)
- kubernetes-deep-dive (Pod, Deployment, Service 동작 원리, Reconciliation Loop)
- linux-for-backend-deep-dive (프로세스 격리, 환경변수, 파일시스템 — 시너지 큼)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: CI/CD 철학과 Pipeline 아키텍처 (5개 문서)
- CI/CD가 해결하는 문제 — 수동 배포의 실제 위험(휴먼 에러, 환경 불일치, 롤백 불가), Continuous Integration이 "항상 배포 가능한 상태"를 만드는 원리
- Pipeline 구성 요소 — Trigger/Job/Step/Artifact의 관계, Runner의 역할(GitHub-hosted vs Self-hosted), 작업 격리가 왜 필요한가
- GitHub Actions 내부 동작 — Webhook 수신 → Runner 선택 → Job 스케줄링 → 컨테이너/VM 프로비저닝, `GITHUB_TOKEN` 권한 범위
- Pipeline 설계 원칙 — Fast Fail(빠른 실패), 병렬 실행으로 피드백 루프 단축, 단계별 게이트(테스트 통과 없이 배포 차단)
- GitOps 원칙 — Git이 시스템 상태의 Single Source of Truth가 되는 원리, 명령형(Imperative) vs 선언형(Declarative) 배포의 차이

### Chapter 2: GitHub Actions 완전 분해 (6개 문서)
- Workflow YAML 구조 분해 — `on:`, `jobs:`, `steps:`, `env:`, `secrets:` 각 키워드의 실제 처리 방식, 표현식 문법(`${{ }}`)의 평가 시점
- Actions 내부 동작 — `uses:` Action이 Docker 컨테이너 또는 JavaScript로 실행되는 방식, Composite Action vs Reusable Workflow 차이
- Job 의존성과 병렬화 — `needs:` 키워드로 DAG(방향 비순환 그래프) 구성, 병렬 Job의 Runner 할당 방식, Matrix Strategy로 다중 환경 테스트
- Secrets와 환경 변수 관리 — Repository/Environment/Organization Secret 범위 차이, Secrets가 로그에 마스킹되는 원리, OIDC로 AWS/GCP 자격증명 없이 인증
- 캐시 전략 — `actions/cache`의 key/restore-keys 계층 구조, Gradle/Maven/npm 의존성 캐시 적중률 최적화, 캐시 충돌 방지
- 아티팩트와 리포트 — 빌드 결과물을 Job 간 공유하는 방법, 테스트 결과 리포트 업로드, 바이너리 아티팩트 보존 정책

### Chapter 3: Docker 이미지 빌드 최적화 (6개 문서)
- Dockerfile 레이어 캐시 원리 — 레이어 불변성, 명령어 순서가 캐시 히트율에 미치는 영향, `COPY` 명령어 순서를 왜 의존성 먼저 해야 하는가
- 멀티 스테이지 빌드 — Builder 스테이지에서 컴파일 후 Runtime 스테이지에만 결과물 복사, Java 애플리케이션 이미지를 800MB → 80MB로 줄이는 실전 예시
- BuildKit 내부 동작 — 병렬 스테이지 빌드, 빌드 캐시 내보내기/가져오기(`--cache-to`, `--cache-from`), 원격 캐시 레지스트리 활용
- 이미지 보안 — 루트가 아닌 사용자 실행, 불필요한 패키지 제거, `distroless` 베이스 이미지, Trivy로 취약점 스캐닝
- Spring Boot 최적화 이미지 — Layered JAR(레이어드 JAR) 구조, Spring Boot Buildpacks(`./gradlew bootBuildImage`), JVM 컨테이너 메모리 설정(`-XX:MaxRAMPercentage`)
- 레지스트리 관리 — Docker Hub vs GitHub Container Registry vs ECR, 이미지 태그 전략(Git SHA vs 버전 태그 vs latest의 위험성), 이미지 정리 정책

### Chapter 4: 배포 전략과 Kubernetes 통합 (7개 문서)
- Rolling Update 완전 분해 — `maxSurge`와 `maxUnavailable` 설정이 배포 속도와 가용성에 미치는 영향, Readiness Probe 실패 시 롤아웃 중단 메커니즘
- Blue-Green 배포 — 두 환경을 동시에 유지하는 비용, Service 트래픽 전환 방식, 롤백이 즉각적인 이유, DB 스키마 변경과의 충돌 문제
- Canary 배포 — 트래픽 비율을 단계적으로 증가시키는 방법, Kubernetes Ingress 가중치 기반 라우팅, Argo Rollouts로 자동화된 카나리 분석
- Argo Rollouts — AnalysisTemplate으로 메트릭 기반 자동 승급/롤백, Prometheus 쿼리로 에러율 임계값 설정, 수동 개입 시점
- 롤백 전략 — `kubectl rollout undo`의 내부 동작, 이전 ReplicaSet이 보존되는 방식, DB 마이그레이션이 있을 때 롤백 불가능한 이유와 Forward-Only 전략
- Zero-Downtime 배포 조건 — Graceful Shutdown(SIGTERM 처리), Connection Draining, Readiness/Liveness Probe 올바른 설정, preStop Hook
- 배포 파이프라인 전체 흐름 — 코드 Push → 테스트 → 이미지 빌드 → 레지스트리 Push → Kubernetes 배포 → 헬스체크 → 알림

### Chapter 5: GitOps와 ArgoCD (6개 문서)
- GitOps 원칙 완전 분해 — Declarative, Versioned, Automated, Continuously Reconciled의 4원칙, 명령형 배포와 비교했을 때의 감사(Audit) 추적성
- ArgoCD 아키텍처 — Application Controller, Repo Server, API Server 역할 분리, Reconciliation Loop가 Git과 Cluster 상태를 비교하는 주기
- Sync 전략 — Manual vs Automated Sync, Self-Heal(클러스터 직접 수정을 자동으로 되돌리는 원리), Pruning(Git에서 삭제된 리소스 자동 제거)
- Kustomize와 Helm 통합 — ArgoCD에서 Kustomize overlay로 환경별 설정 분리, Helm values.yaml을 Git으로 관리하는 방법, 템플릿 렌더링 시점
- 멀티 클러스터 관리 — ArgoCD ApplicationSet으로 여러 클러스터에 동일 앱 배포, 클러스터별 설정 오버라이드, 환경별 프로모션 파이프라인
- Secret 관리 — Git에 Secret을 저장하면 안 되는 이유, Sealed Secrets / External Secrets Operator / Vault Agent 비교, SOPS(Mozilla Secret OPerationS)

### Chapter 6: 테스트 자동화와 품질 게이트 (5개 문서)
- 테스트 피라미드와 Pipeline — 단위/통합/E2E 테스트의 Pipeline 배치 전략, 빠른 피드백을 위한 테스트 병렬화, 테스트 격리를 위한 Docker 컨테이너 활용
- Spring Boot 테스트 최적화 — `@SpringBootTest` 컨텍스트 재사용, Testcontainers로 실제 DB/Redis 사용, `@MockBean` 남용이 테스트를 느리게 만드는 이유
- 코드 품질 게이트 — SonarQube Quality Gate가 파이프라인을 중단시키는 조건, JaCoCo 코드 커버리지 임계값, 정적 분석(SpotBugs, Checkstyle) 자동화
- 성능 회귀 감지 — k6 smoke test를 Pipeline에 통합, 응답시간 임계값 초과 시 배포 차단, Baseline 비교로 성능 저하 자동 감지
- 보안 스캐닝 자동화 — Trivy 이미지 취약점 스캔, OWASP Dependency-Check로 라이브러리 CVE 감지, Secret 노출 방지(TruffleHog, git-secrets)

### Chapter 7: 모니터링, 알림과 운영 (5개 문서)
- 배포 추적과 알림 — Slack/Discord 배포 성공/실패 알림, GitHub Deployment API로 배포 이력 추적, Datadog/Grafana 배포 마커 연동
- 파이프라인 장애 진단 — GitHub Actions 디버그 로깅(`ACTIONS_STEP_DEBUG`), Runner 환경 확인, 간헐적 실패의 원인 분석 방법론
- Pipeline 성능 최적화 — Job 실행 시간 분석, 병목 단계 식별, 의존성 캐시 적중률 향상, 불필요한 재빌드 제거
- 환경 관리 전략 — Dev/Staging/Prod 환경 분리, 환경별 변수 관리, Production-like Staging 환경 구성 방법
- 장애 시나리오와 복구 — 배포 중 장애 발생 시 자동 롤백 트리거, 데이터베이스 롤백이 없는 이유, 포스트모텀(Postmortem) 작성 방법

---

각 챕터는 **5~7개 문서**로 구성해줘. 총 **40개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 실무에서 중요한가
## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)
## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)
## 🔬 내부 동작 원리 (GitHub Actions/ArgoCD/Docker 내부 분석)
## 💻 실전 실험 (GitHub Actions YAML, kubectl, ArgoCD CLI 재현 시나리오)
## 📊 성능/비용 비교 (빌드 시간, 배포 시간, 가용성 비교)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 모든 개념은 실제 GitHub Actions YAML 또는 `kubectl` 명령어로 재현 가능하게
2. 잘못된 Dockerfile/Workflow → 최적화된 버전 Before/After 비교 필수
3. 각 설계 결정의 이유 명시 ("왜 레이어 순서가 중요한가", "왜 latest 태그가 위험한가")
4. kubernetes-deep-dive와 연결 — Deployment/Service/Ingress 리소스가 배포 파이프라인에서 어떻게 사용되는지
5. 운영 장애 시나리오 → 진단 명령어 세트 (kubectl rollout status, argocd app get)

**실험 환경**:
```yaml
# docker-compose.yml (로컬 GitOps 환경)
services:
  gitea:
    image: gitea/gitea:latest
    ports:
      - "3000:3000"
    volumes:
      - gitea-data:/data

  act-runner:
    image: gitea/act_runner:latest
    environment:
      GITEA_INSTANCE_URL: http://gitea:3000
      GITEA_RUNNER_REGISTRATION_TOKEN: ${RUNNER_TOKEN}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock

  argocd:
    image: quay.io/argoproj/argocd:latest
    ports:
      - "8080:8080"

volumes:
  gitea-data:
```

```yaml
# .github/workflows/ci-cd.yml — 완전한 파이프라인 예시
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          cache: 'gradle'
      - run: ./gradlew test

  build:
    needs: test
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix={{branch}}-
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: production
    steps:
      - run: |
          # GitOps: Git 저장소의 이미지 태그 업데이트
          # ArgoCD가 감지하여 자동 Sync
          yq e '.spec.template.spec.containers[0].image = "${{ needs.build.outputs.image-tag }}"' \
            -i k8s/deployment.yaml
```

```bash
# 배포 상태 확인 명령어 세트
# Rolling Update 진행 상황
kubectl rollout status deployment/myapp -n production

# 롤백 실행
kubectl rollout undo deployment/myapp -n production

# ArgoCD Sync 상태 확인
argocd app get myapp
argocd app sync myapp

# 카나리 트래픽 비율 조정 (Argo Rollouts)
kubectl argo rollouts set weight myapp 20
kubectl argo rollouts promote myapp

# Docker 레이어 캐시 분석
docker history myapp:latest --no-trunc
docker buildx imagetools inspect myapp:latest
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**:
   - kafka-deep-dive README 스타일 참고
   - "배포 버튼을 누르는 것과 파이프라인 내부를 아는 것의 차이"라는 포지셔닝 강조
   - docker-deep-dive, kubernetes-deep-dive, observability-deep-dive와의 연결 지점 명시
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료

- GitHub Actions 공식 문서 — https://docs.github.com/en/actions
- ArgoCD 공식 문서 — https://argo-cd.readthedocs.io/
- Argo Rollouts — https://argoproj.github.io/rollouts/
- Google SRE Book (배포 안정성 챕터) — https://sre.google/sre-book/
- Docker BuildKit 문서 — https://docs.docker.com/build/buildkit/
- Continuous Delivery (Jez Humble, David Farley)
- GitOps 원칙 — https://opengitops.dev/

---

## 💡 핵심 분석 대상

```
CI/CD 전체 흐름:

개발자 git push
  │
  ▼ GitHub Webhook → Actions Runner 선택
  │
  ▼ [CI 단계 — 병렬 실행]
  ├── 단위 테스트 (./gradlew test)
  ├── 정적 분석 (./gradlew sonarqube)
  └── 보안 스캔 (trivy filesystem .)
  │
  ▼ [빌드 단계]
  Docker Build (레이어 캐시 활용)
    FROM eclipse-temurin:21-jre        ← 자주 변경 X → 캐시 히트
    COPY build/libs/*.jar app.jar      ← 자주 변경 O → 최하단 배치
  │
  ▼ 이미지 Push → ghcr.io (SHA 태그)
  │
  ▼ [CD 단계 — GitOps]
  Git 저장소 k8s/deployment.yaml 이미지 태그 업데이트
    ArgoCD 감지 (3분 간격 Polling 또는 Webhook)
    ArgoCD Sync → kubectl apply
  │
  ▼ [배포 전략]
  Rolling Update:
    maxSurge=1, maxUnavailable=0
    새 Pod Readiness 확인 후 구 Pod 종료
    → Readiness Probe 실패 시 자동 중단
  │
  ▼ 배포 검증
  Smoke Test → Prometheus 에러율 확인
  에러율 > 1% → 자동 롤백

Docker 레이어 캐시 원리:
  ✗ 잘못된 순서 (매번 의존성 재다운로드):
    COPY . .                    ← 소스 변경 → 캐시 무효
    RUN ./gradlew dependencies  ← 항상 재실행 (비용 큼)

  ✓ 올바른 순서 (의존성 캐시 유지):
    COPY build.gradle settings.gradle ./
    RUN ./gradlew dependencies --no-daemon  ← 의존성만: 캐시 유지
    COPY . .                               ← 소스: 이후만 무효
    RUN ./gradlew bootJar

배포 전략 가용성 비교:
  Rolling:     가용성 99.9% / 롤백 느림 (구 Pod 이미 종료)
  Blue-Green:  가용성 100%  / 비용 2배 (두 환경 유지)
  Canary:      가용성 100%  / 위험 최소화 (일부 트래픽만 노출)
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (40개 목표)
- GitHub Actions 실험 환경 구성
- docker-deep-dive, kubernetes-deep-dive, observability-deep-dive와 연결되는 지점 명시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
