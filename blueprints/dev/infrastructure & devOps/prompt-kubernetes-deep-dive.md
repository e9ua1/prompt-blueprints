# Kubernetes Deep Dive 레포지토리 제작 프롬프트

나는 "Kubernetes Deep Dive" 레포지토리를 만들려고 해.
`kubectl apply`로 파드를 배포하는 것과, Control Plane이 원하는 상태와 현재 상태를 어떻게 비교하고 수렴시키는지, 스케줄러가 파드를 어떤 노드에 어떤 기준으로 배치하는지, kube-proxy가 ClusterIP를 어떻게 실제 파드로 라우팅하는지를 완전히 파헤치는 레포를 만들 거야.

docker-deep-dive를 마친 독자가 "그래서 컨테이너 오케스트레이션은 내부에서 어떻게 동작하나?"를 배우는 레포야.

## 📋 프로젝트 목표

**컨셉**: "컨테이너를 배포하는 것과, Control Plane이 수천 개의 파드 상태를 어떻게 선언적으로 일치시키는지 아는 것은 다르다"

**핵심 차별화**:
1. Reconciliation Loop — etcd의 원하는 상태와 현재 상태를 어떻게 일치시키는가
2. 스케줄링 — Predicates/Scoring으로 파드가 어떤 노드에 배치되는가
3. 네트워킹 — ClusterIP/NodePort/LoadBalancer가 iptables/IPVS 레벨에서 어떻게 구현되는가
4. 스토리지 — PV/PVC/StorageClass가 실제 디스크와 어떻게 연결되는가

**타겟 독자**:
- `kubectl apply`를 쓰지만 Control Plane이 내부에서 무슨 일을 하는지 모르는 개발자
- Service를 만들면 로드밸런싱이 된다고 알지만 iptables 규칙이 어떻게 생성되는지 모르는 개발자
- Liveness/Readiness Probe가 왜 필요한지 kubelet 관점에서 설명 못하는 개발자
- HPA가 파드를 어떻게 스케일아웃하는지 메트릭 파이프라인을 모르는 개발자
- etcd가 무엇인지는 알지만 Raft 합의 알고리즘이 어떻게 동작하는지 모르는 개발자
- docker-deep-dive를 마쳤고 쿠버네티스의 내부까지 알고 싶은 개발자

**선행 학습**:
- docker-deep-dive (컨테이너, namespace, cgroups 개념 필수)
- linux-for-backend-deep-dive (namespace/cgroups/iptables 원리 이해하면 시너지 큼)
- network-deep-dive (iptables, L4/L7 로드밸런싱 이해 시 네트워킹 챕터 깊이 배가)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Kubernetes 아키텍처 — Control Plane과 Data Plane (6개 문서)
- 전체 아키텍처 개요 — Control Plane(API Server/etcd/Scheduler/Controller Manager) vs Data Plane(kubelet/kube-proxy/Container Runtime), 각 컴포넌트가 서로 통신하는 방식
- API Server — 모든 요청의 진입점, 인증(Authentication)/인가(Authorization)/어드미션 컨트롤러(Admission Controller) 파이프라인, Watch 메커니즘으로 컴포넌트들이 상태 변화를 감지하는 방법
- etcd와 Raft 합의 알고리즘 — etcd가 클러스터 상태의 단일 진실 공급원인 이유, Raft Leader Election과 Log Replication, Quorum이 가용성을 보장하는 방식
- Controller Manager — Reconciliation Loop의 실체, Desired State vs Actual State 비교, ReplicaSet Controller가 파드 수를 유지하는 정확한 메커니즘
- kubelet — 노드 에이전트의 역할, CRI(Container Runtime Interface)로 컨테이너를 실행하는 방식, Liveness/Readiness/Startup Probe가 컨테이너 상태를 판단하는 방법
- 클러스터 부트스트랩 — kubeadm이 클러스터를 초기화하는 전 과정, 인증서 생성, Static Pod로 Control Plane이 실행되는 방식

### Chapter 2: 파드의 생명주기 — 생성부터 종료까지 (5개 문서)
- 파드 생성 전 과정 — `kubectl apply` → API Server → etcd → Scheduler → kubelet → Container Runtime까지 각 단계의 실제 동작
- 파드 스케줄링 알고리즘 — Filtering(Predicates) 단계로 부적합 노드 제거, Scoring(Priorities) 단계로 최적 노드 선택, NodeAffinity/Taint/Toleration이 스케줄링에 미치는 영향
- 컨테이너 런타임 — containerd/CRI-O가 CRI를 통해 컨테이너를 생성하는 방식, OCI 스펙(runc), containerd Shim 역할
- 파드 종료 — SIGTERM 전달 후 gracePeriod 대기, preStop Hook, 연결 드레인이 무중단 배포에 왜 중요한가
- Init Container와 Sidecar — Init Container 실행 순서 보장 원리, Sidecar 패턴(Envoy Proxy, Fluentd)이 Pod 네트워크 네임스페이스를 공유하는 방식

### Chapter 3: Kubernetes 네트워킹 완전 분해 (7개 문서)
- 파드 네트워킹 기초 — 모든 파드가 고유한 IP를 갖는 이유, 파드 간 통신이 NAT 없이 이루어지는 원리
- CNI(Container Network Interface) — CNI 플러그인이 파드 IP를 할당하는 방식, Flannel(VXLAN 오버레이) vs Calico(BGP 라우팅) 내부 동작 차이
- Service와 kube-proxy — ClusterIP가 실제 IP 없이 동작하는 방법, kube-proxy가 iptables 규칙을 생성/관리하는 방식, IPVS 모드의 차이
- Service 종류 완전 분해 — ClusterIP/NodePort/LoadBalancer/ExternalName 각각의 구현 방식, NodePort가 모든 노드에서 동일하게 동작하는 이유
- Ingress — Ingress Controller(Nginx/Traefik)가 L7 라우팅을 구현하는 방식, Ingress 리소스와 실제 Nginx 설정의 관계
- DNS — CoreDNS가 Service 이름을 ClusterIP로 변환하는 방식, ndots 설정이 DNS 조회 성능에 미치는 영향
- Network Policy — iptables/eBPF로 파드 간 트래픽을 필터링하는 원리, Default Deny 정책이 동작하는 방식

### Chapter 4: 스토리지 — 데이터 영속성의 원리 (5개 문서)
- Volume과 파드 — emptyDir/hostPath/configMap/secret이 각각 어떻게 마운트되는가, tmpfs와 실제 파일시스템의 차이
- PersistentVolume과 PersistentVolumeClaim — 동적 프로비저닝이 StorageClass를 통해 실제 디스크를 생성하는 과정, Binding 알고리즘
- CSI(Container Storage Interface) — 플러그인 없이 스토리지 드라이버를 교체할 수 있는 이유, CSI Driver가 볼륨을 노드에 Attach/Mount하는 과정
- StatefulSet — 파드 순서 보장, 안정적인 네트워크 ID, 볼륨이 파드 재생성 후에도 유지되는 원리
- 스토리지 성능 고려사항 — ReadWriteOnce vs ReadWriteMany, 로컬 PV와 원격 스토리지의 성능 차이, 데이터베이스를 쿠버네티스에서 운영할 때의 트레이드오프

### Chapter 5: 자원 관리와 스케일링 (5개 문서)
- Request와 Limit — CPU/메모리 Request와 Limit의 차이, Limit 없는 파드가 노드에 미치는 영향, cgroups로 Limit이 구현되는 방식
- QoS 클래스 — Guaranteed/Burstable/BestEffort 분류 기준, OOM 발생 시 어떤 파드가 먼저 종료되는가
- HPA(Horizontal Pod Autoscaler) — Metrics Server가 CPU/메모리 메트릭을 수집하는 방법, HPA 컨트롤러가 파드 수를 계산하는 알고리즘
- VPA(Vertical Pod Autoscaler) — Request/Limit을 자동 조정하는 원리, Admission Webhook으로 파드 스펙을 변경하는 방식
- 클러스터 오토스케일러 — Pending 파드 감지 → 노드 추가 결정 → 클라우드 API 호출 과정, 스케일 다운 시 파드 이동 원리

### Chapter 6: 배포 전략과 운영 (5개 문서)
- Deployment 롤링 업데이트 — maxSurge/maxUnavailable 설정으로 제어되는 교체 알고리즘, 롤백이 ReplicaSet으로 구현되는 방식
- 무중단 배포 — Readiness Probe + terminationGracePeriodSeconds + preStop Hook의 삼각편대, 연결이 끊기지 않는 파드 교체 절차
- RBAC — Role/ClusterRole/RoleBinding/ClusterRoleBinding 계층 구조, ServiceAccount가 API Server에 인증하는 방식
- ConfigMap과 Secret — 환경변수/파일 마운트 방식 비교, Secret이 Base64이지 암호화가 아닌 이유, Sealed Secret/External Secrets Operator
- 모니터링과 로깅 — Prometheus Operator로 메트릭 수집, Loki로 중앙 로그 수집, kubectl top이 데이터를 가져오는 경로

### Chapter 7: 고급 패턴과 운영 심화 (5개 문서)
- Operator 패턴 — CRD(Custom Resource Definition)로 쿠버네티스를 도메인 특화 플랫폼으로 확장하는 방법, Reconciliation Loop 직접 구현
- Admission Webhook — Mutating/Validating Webhook이 리소스 생성/수정 시 호출되는 방식, Istio Sidecar Injection이 동작하는 원리
- Service Mesh(Istio) — Envoy Sidecar가 모든 트래픽을 가로채는 방법, mTLS 자동 적용, 트래픽 분산(Canary/Blue-Green) 구현
- etcd 운영 — etcd 백업과 복구, etcd 클러스터 확장, 디스크 I/O가 Control Plane 성능을 결정하는 이유
- 멀티 클러스터 운영 — 클러스터 Federation, KubeFed, 멀티 클러스터 네트워킹(Submariner), 지역(Region) 간 트래픽 라우팅

---

각 챕터는 **5~7개 문서**로 구성해줘. 총 **38개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 실무에서 중요한가
## 😱 흔한 실수 (Before — 내부를 모를 때의 접근)
## ✨ 올바른 접근 (After — 내부를 알고 난 설계/운영)
## 🔬 내부 동작 원리 (컴포넌트 소스 레벨 분석, kubectl 이벤트 추적)
## 💻 실전 실험 (kubectl, etcdctl, iptables-save 등으로 재현)
## 📊 성능/비용 비교 (iptables vs IPVS, Flannel vs Calico 등)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. "kubectl apply가 내부에서 어떤 API 호출을 하는가" 수준으로 구체적으로
2. `kubectl describe`, `kubectl get events`, `kubectl logs` 결과를 직접 분석하는 실험
3. `iptables-save`로 kube-proxy가 생성한 실제 iptables 규칙 확인
4. 기존 레포 연결 명시 — docker-deep-dive(cgroups/namespace), linux-deep-dive(iptables), network-deep-dive(CNI)
5. 운영 장애 시나리오 → 진단 절차 (CrashLoopBackOff 원인 찾기, OOMKilled 분석 등)

**실험 환경**:
```yaml
# docker-compose.yml (Kind 기반 로컬 클러스터)
services:
  kind-cluster-setup:
    image: alpine:3.18
    privileged: true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./kind-config.yaml:/kind-config.yaml
    command: >
      sh -c "
        apk add --no-cache curl &&
        curl -Lo /usr/local/bin/kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64 &&
        chmod +x /usr/local/bin/kind &&
        kind create cluster --config /kind-config.yaml --name deep-dive
      "
```

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
networking:
  disableDefaultCNI: false   # 기본 CNI 사용 (kindnetd)
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"
```

```bash
# 실험용 공통 명령어 세트

# etcd 직접 조회 (클러스터 상태 원본 데이터)
kubectl exec -n kube-system etcd-<node> -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  get /registry/pods/default --prefix --keys-only

# iptables 규칙 확인 (kube-proxy가 생성한 규칙)
iptables-save | grep <service-name>
iptables -t nat -L KUBE-SERVICES -n --line-numbers

# API Server 감사 로그 추적
kubectl get events --sort-by='.metadata.creationTimestamp' -A

# 파드 스케줄링 추적
kubectl describe pod <pod-name> | grep -A 10 Events

# 네트워크 정책 확인
kubectl exec -it <pod> -- curl <other-pod-ip>:<port>
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**:
   - mysql-deep-dive README 스타일 참고
   - "kubectl을 쓰는 것을 넘어 Control Plane 내부를 이해한다"는 차별화 강조
   - docker-deep-dive, linux-deep-dive, network-deep-dive와의 연결 지점 명시
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료 섹션

README 작성 시 다음을 Reference로 포함해줘:
- Kubernetes 공식 문서 — https://kubernetes.io/docs/
- Kubernetes in Action, 2nd Edition (Marko Luksa)
- Programming Kubernetes (Michael Hausenblas & Stefan Schimanski)
- Kubernetes 소스 코드 — https://github.com/kubernetes/kubernetes
- Learnk8s Blog — https://learnk8s.io/blog
- Ivan Velichko's Blog — https://iximiuz.com/en/

---

## 💡 핵심 분석 대상

```
파드 생성 전 과정:

kubectl apply -f pod.yaml
  │
  ▼ 1. API Server 수신
  │   ├── 인증 (Bearer Token / Client Certificate)
  │   ├── 인가 (RBAC 체크)
  │   └── Admission Controller (Mutating → Validating)
  │
  ▼ 2. etcd에 Desired State 저장
  │   Pod 오브젝트: spec.nodeName = "" (미배정 상태)
  │
  ▼ 3. Scheduler 감지 (Watch)
  │   ├── Filtering: 리소스 부족한 노드 제거
  │   │   (Insufficient CPU/Memory, Taint, NodeAffinity)
  │   ├── Scoring: 남은 노드 점수 계산
  │   │   (LeastAllocated, InterPodAffinity 등)
  │   └── API Server에 Bind 요청 (spec.nodeName = "worker-1")
  │
  ▼ 4. kubelet 감지 (Watch)
  │   ├── Pod 스펙 확인
  │   ├── 이미지 Pull (containerd에 요청)
  │   ├── 네트워크 설정 (CNI 플러그인 호출)
  │   └── 컨테이너 시작 (containerd → runc)
  │
  ▼ 5. 상태 업데이트
      ├── kubelet → API Server: Pod status = Running
      └── kube-proxy: Service iptables 규칙 갱신

Service 네트워킹 (ClusterIP):
  kubectl expose deployment myapp --port=80
    │
    ├── API Server가 Service 오브젝트 생성
    │   ClusterIP: 10.96.43.125 (가상 IP, 실제 바인딩 없음)
    │
    ├── kube-proxy가 iptables 규칙 생성:
    │   PREROUTING → KUBE-SERVICES → KUBE-SVC-<hash>
    │   → KUBE-SEP-<hash> (실제 Pod IP:Port로 DNAT)
    │
    └── 클라이언트가 10.96.43.125:80 접근
        → iptables DNAT → 실제 Pod 10.244.1.5:8080

Reconciliation Loop (ReplicaSet):
  목표: 3개 파드 실행
  현재: 2개 파드 실행 (1개 노드 다운으로 종료)
    │
    ▼ Controller Manager의 ReplicaSet Controller
    ├── etcd에서 Desired (3) vs Actual (2) 비교
    ├── 차이(1) 감지
    ├── 새 Pod 오브젝트 생성 요청
    └── 다시 etcd Watch → Desired == Actual 확인

  이것이 쿠버네티스의 핵심:
  "명령형(imperative)"이 아닌 "선언형(declarative)"
  상태를 선언하면 컨트롤러가 알아서 맞춤
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (38개 목표)
- Kind 기반 로컬 클러스터 실험 환경
- docker-deep-dive, linux-deep-dive, network-deep-dive와 연결되는 지점 명시 (어느 문서에서 어느 레포와 연결하는지)

**준비됐으면 1단계 구조 설계부터 시작해줘!**
