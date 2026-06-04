# Service Mesh Deep Dive 레포지토리 제작 프롬프트

나는 "Service Mesh Deep Dive" 레포지토리를 만들려고 해.
서비스 간 통신을 애플리케이션 밖으로 빼내는 방식 — Sidecar 프록시(Envoy), 데이터 플레인/컨트롤 플레인 분리, mTLS, 트래픽 정책·관측을 완전히 파헤치는 레포야.
"Istio를 설치하는 것"과 "왜 Sidecar가 트래픽을 가로채고 컨트롤 플레인이 어떻게 정책을 배포하는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "메시를 설치하는 것과, Sidecar가 트래픽을 가로채고 컨트롤 플레인이 데이터 플레인을 어떻게 제어하는지 아는 것은 다르다"

**핵심 차별화**:
1. 통신을 앱 밖으로 — 재시도·타임아웃·mTLS를 코드가 아닌 인프라 레이어로
2. 데이터/컨트롤 플레인 분리 — Envoy(트래픽 처리) vs 컨트롤 플레인(정책 배포)
3. 투명한 가로채기 — Sidecar가 iptables로 트래픽을 가로채는 원리
4. Zero Trust — mTLS로 서비스 간 상호 인증, 신원 기반 정책(crypto 연결)

**타겟 독자**:
- Istio를 설치했지만 Sidecar가 뭘 하는지 모르는 개발자
- 재시도·서킷브레이커를 코드로 짜던 개발자(spring-cloud)
- mTLS·데이터/컨트롤 플레인을 단어로만 아는 개발자
- 메시의 비용(지연·복잡도)을 모르는 개발자
- `kubernetes`·`network`·`grpc`를 메시로 연결하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `kubernetes-deep-dive`(Sidecar·Pod·CRD), `network-deep-dive`(L4/L7·TLS·HTTP/2).
**🤝 시너지**: `grpc-deep-dive`(Envoy의 gRPC·xDS 프로토콜), `spring-cloud-deep-dive`(라이브러리 vs 인프라 레질리언스 대조), `cryptography-deep-dive`(mTLS), `observability-deep-dive`(메시 텔레메트리).
**🧬 수렴**: `spring-cloud`(코드 레벨 분산 패턴)와 *같은 문제를 인프라 레벨로*. `iac`와 함께 클라우드 네이티브.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 서비스 메시의 개념 (5개 문서)
- 왜 메시인가 — MSA의 통신 횡단 관심사(재시도·보안·관측)를 어디서 처리하나
- 라이브러리 vs 메시 — Spring Cloud(코드) vs 메시(인프라), 트레이드오프(spring-cloud 연결)
- Sidecar 패턴 — 앱 옆 프록시가 모든 트래픽 처리, 앱은 통신을 모름
- 데이터 플레인 vs 컨트롤 플레인 — 트래픽 처리 vs 정책 관리 분리
- 메시 지형 — Istio·Linkerd·Cilium, 아키텍처 차이

### Chapter 2: Envoy — 데이터 플레인 (6개 문서)
- Envoy 개요 — L7 프록시, 메시의 일꾼, 왜 Envoy인가
- Envoy 구조 — Listener·Filter Chain·Cluster·Endpoint
- 트래픽 가로채기 — iptables로 Pod 트래픽을 Sidecar로 리다이렉트(투명)
- L7 처리 — HTTP/gRPC 라우팅·로드밸런싱(network·grpc 연결)
- 필터 — 재시도·타임아웃·헤더 조작을 필터 체인으로
- xDS API — 컨트롤 플레인이 Envoy를 동적 설정하는 프로토콜(gRPC 기반)

### Chapter 3: 컨트롤 플레인 (5개 문서)
- 역할 — 정책을 Envoy 설정으로 변환·배포, 서비스 디스커버리
- istiod — Istio 컨트롤 플레인, 설정 배포·인증서 발급
- 설정 전파 — CRD(VirtualService 등)→xDS→Envoy, eventual consistency(distributed 연결)
- 서비스 디스커버리 — K8s 서비스와 통합, 엔드포인트 추적
- 인증서 관리 — 워크로드 신원·인증서 발급·회전(crypto 연결)

### Chapter 4: 트래픽 관리 (6개 문서)
- 라우팅 — VirtualService, 가중치·헤더 기반 라우팅
- 카나리·블루그린 — 트래픽 분할로 점진 배포(cicd 연결)
- 로드밸런싱 — 알고리즘, 지역 인식, gRPC L7 분산(grpc 연결)
- 레질리언스 — 재시도·타임아웃·서킷브레이커를 메시에서(spring-cloud 대조)
- 결함 주입 — 지연·에러 주입으로 복원력 테스트(distributed 연결)
- 트래픽 미러링 — 프로덕션 트래픽 복제 테스트

### Chapter 5: 보안 — Zero Trust (5개 문서)
- mTLS — 서비스 간 상호 TLS, 자동 암호화(crypto·network 연결)
- 워크로드 신원 — SPIFFE/SVID, 인증서 기반 신원
- 인증 정책 — 누가 누구와 통신 가능, PeerAuthentication
- 인가 정책 — 세밀한 접근 제어(AuthorizationPolicy)
- Zero Trust 모델 — "신뢰하지 말고 검증", 네트워크 경계 너머

### Chapter 6: 관측성 (5개 문서)
- 자동 텔레메트리 — Sidecar가 모든 트래픽 메트릭 수집(코드 변경 0)
- 메트릭 — 요청률·에러율·지연(RED), Prometheus(observability 연결)
- 분산 추적 — 트레이스 헤더 전파, 메시의 추적 자동화
- 서비스 그래프 — 의존성 시각화(Kiali)
- 한계 — 메시 텔레메트리가 못 보는 것, 앱 레벨 관측 필요성

### Chapter 7: 비용과 실전 (4개 문서)
- 메시의 비용 — Sidecar 지연·리소스·복잡도, 언제 메시가 과한가
- Sidecar-less — Ambient Mesh·eBPF(Cilium), Sidecar 비용 줄이기
- 도입 전략 — 점진 도입, 메시가 필요한 규모
- 종합 — K8s에 Istio 설치, mTLS·카나리·결함주입 실습, 텔레메트리 관찰

→ **총 34개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 Envoy·xDS·iptables 가로채기·mTLS, `💻 실전 실험`은 K8s+Istio·Envoy config dump·트래픽 정책. `📊`는 메시 유무 지연·라이브러리 vs 메시 비교.

## 🎨 스타일 가이드

1. **"앱 밖으로"로 환원** — 모든 기능을 "코드가 아닌 인프라가" 관점으로
2. **spring-cloud와 대조** — 같은 패턴(서킷브레이커·재시도)을 코드 vs 메시로
3. **Envoy config를 까본다** — config_dump로 동적 설정을 *직접 본다*
4. **crypto/network/grpc 레포로 착지** — mTLS·L7·xDS의 토대
5. 데이터/컨트롤 플레인·트래픽 가로채기는 다이어그램으로

## 🔬 검증 환경

> docker + kind(로컬 K8s) + Istio.

```bash
# 로컬 K8s + Istio
kind create cluster
istioctl install --set profile=demo -y
kubectl label namespace default istio-injection=enabled   # Sidecar 자동 주입

# 검증 방법
# 1) Sidecar 주입 확인: kubectl get pod → 컨테이너 2개(앱 + istio-proxy)
# 2) iptables 가로채기: istio-init이 설정한 redirect 규칙 확인
kubectl exec <pod> -c istio-proxy -- iptables -t nat -L
# 3) Envoy 설정 까보기:
istioctl proxy-config listeners <pod>      # Listener
istioctl proxy-config cluster <pod>        # Cluster/Endpoint
kubectl exec <pod> -c istio-proxy -- curl localhost:15000/config_dump
# 4) mTLS 확인: PeerAuthentication STRICT → 평문 거부 확인
# 5) 카나리: VirtualService 가중치 90/10 → 트래픽 분할 관찰
# 6) 결함 주입: 지연/에러 주입 → 클라이언트 영향 관찰
# 7) Kiali로 서비스 그래프·텔레메트리 시각화
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🖥️ Infrastructure 톤, "메시를 설치한다 vs 어떻게 트래픽을 제어하나" 포지셔닝, `🔗 레포 연결`(kubernetes·network·grpc·spring-cloud)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- Istio 공식 문서 — https://istio.io/latest/docs/
- Envoy 문서 — https://www.envoyproxy.io/docs
- "Istio in Action" (Christian Posta)
- SPIFFE/SPIRE 문서(워크로드 신원)
- Linkerd·Cilium 문서(대조)

## 💡 핵심 분석 대상

```
통신을 앱 밖으로:
  기존(Spring Cloud): 앱 코드에 재시도·서킷브레이커·mTLS 라이브러리
  메시: 같은 기능을 Sidecar(Envoy)가 처리 → 앱은 평범한 HTTP만
  → 언어 무관·코드 변경 0, 단 인프라 복잡도·지연 추가

Sidecar 가로채기 (투명):
  앱 컨테이너 ──┐
                │ iptables가 모든 in/out 트래픽을
                ▼ Sidecar(Envoy)로 리다이렉트
  Envoy: mTLS·재시도·메트릭 처리 → 목적지로
  → 앱은 자기 트래픽이 가로채진 줄 모름

데이터 vs 컨트롤 플레인:
  컨트롤(istiod): VirtualService(CRD) → Envoy 설정으로 변환
       │ xDS(gRPC)
       ▼
  데이터(Envoy): 실제 트래픽 처리(라우팅·mTLS·재시도)
  → 설정 변경은 컨트롤, 트래픽은 데이터 (분리)

mTLS (Zero Trust):
  서비스 A ↔ 서비스 B: 양쪽 인증서로 상호 인증 + 암호화
  istiod가 워크로드별 인증서 발급·회전(crypto 레포)
  → 네트워크 안이라도 신뢰 안 함, 검증

라이브러리 vs 메시 (같은 문제):
  서킷브레이커: Spring Cloud(코드) vs Istio(설정)
  → 폴리글랏·운영 일관성은 메시, 단순함·저지연은 라이브러리
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 34개 확인 + kind+Istio/Envoy config 검증 환경 + kubernetes/network/grpc/spring-cloud 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
