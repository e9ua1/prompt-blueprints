# IaC Deep Dive 레포지토리 제작 프롬프트

나는 "IaC (Infrastructure as Code) Deep Dive" 레포지토리를 만들려고 해.
인프라를 코드로 선언하면 어떻게 실제 자원이 되는지 — Terraform의 상태(state)·plan/apply 그래프, Provider 프로토콜, 드리프트 감지, 멱등성을 완전히 파헤치는 레포야.
"Terraform을 작성하는 것"과 "state가 왜 필요하고 plan이 어떻게 차이를 계산하며 무엇이 멱등성을 보장하는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "Terraform 코드를 작성하는 것과, state·의존성 그래프·Provider가 선언을 실제 자원으로 수렴시키는 원리를 아는 것은 다르다"

**핵심 차별화**:
1. 선언적 수렴 — "원하는 상태"를 선언하면 현재→목표로 수렴, 명령형 스크립트와의 차이
2. State의 역할 — 코드·실제 자원 사이 매핑, 왜 state 없이는 안 되는가
3. plan = diff 계산 — 의존성 그래프 + state + 실제로 변경 계획을 만드는 원리
4. 멱등성과 드리프트 — 여러 번 적용해도 같은 결과, 수동 변경(드리프트) 감지

**타겟 독자**:
- Terraform을 쓰지만 state가 왜 필요한지 모르는 개발자
- plan이 어떻게 변경을 계산하는지 모르는 개발자
- 드리프트·state 충돌을 만나지만 원인을 모르는 개발자
- 멱등성을 단어로만 아는 개발자
- `kubernetes`(선언적 수렴)·`cicd`를 인프라 코드로 연결하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `docker-deep-dive`·`kubernetes-deep-dive`(관리 대상 인프라), `cicd-deep-dive`(파이프라인에서 apply).
**🤝 시너지**: `kubernetes-deep-dive`(둘 다 선언적 수렴·reconciliation), `service-mesh-deep-dive`(메시 설정도 선언적), `distributed-systems-theory`(state 일관성·락).
**🧬 수렴**: K8s reconciliation과 *같은 선언적 수렴 패턴*. 인프라 자동화의 토대.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: IaC의 개념 (5개 문서)
- IaC란 — 인프라를 코드로, 수동 클릭(ClickOps)의 문제
- 선언적 vs 명령형 — "원하는 상태 선언"(Terraform) vs "단계 나열"(스크립트)
- 멱등성 — 여러 번 적용해도 같은 결과, 왜 중요한가(distributed 연결)
- 수렴(convergence) — 현재 상태→목표 상태로 맞추기, K8s와 공통 철학
- 도구 지형 — Terraform/OpenTofu·Pulumi·CloudFormation·Ansible 위치

### Chapter 2: Terraform 동작 모델 (6개 문서)
- 구성 언어(HCL) — 선언적 리소스 정의, 표현식·변수
- 워크플로 — init→plan→apply→destroy, 각 단계의 일
- Resource·Data Source — 관리 자원 vs 조회 자원
- 의존성 그래프 — 리소스 참조로 그래프 구성, 생성 순서 결정
- 그래프 순회 — 병렬 생성·순서 보장, 의존 기반 실행
- 변수·출력·모듈 — 재사용·파라미터화, 모듈 경계

### Chapter 3: State 완전 분해 (6개 문서)
- State가 필요한 이유 — 코드↔실제 자원 매핑, ID 추적, 없으면 안 되는 이유
- State 구조 — 리소스·속성·메타데이터, JSON 내부
- plan과 state — 코드(목표) vs state(기록) vs 실제(refresh) 3자 비교
- Remote State — 팀 공유, 백엔드(S3 등), 왜 로컬은 위험한가
- State Locking — 동시 apply 충돌 방지(distributed 락 연결)
- State 조작 — import·mv·rm, state와 실제 불일치 복구

### Chapter 4: plan과 apply (6개 문서)
- plan = diff — 목표(코드) - 현재(state+refresh) = 변경 계획
- refresh — 실제 자원 상태를 읽어 state 갱신, 드리프트 발견
- 변경 종류 — create/update/replace(destroy+create)/destroy
- replace의 위험 — 어떤 속성 변경이 자원 재생성을 유발(force new)
- apply 실행 — 그래프 순서대로 Provider 호출, 부분 실패 처리
- plan 읽기 — `+/-/~/-+` 기호, 변경 검토의 중요성

### Chapter 5: Provider와 확장 (5개 문서)
- Provider 프로토콜 — Terraform Core ↔ Provider(gRPC, grpc 연결!)
- Provider 동작 — CRUD를 클라우드 API로 번역, 스키마
- Provider 개발 개요 — 리소스 정의·CRUD 구현
- 버전·제약 — Provider/모듈 버전 고정, 호환성
- 레지스트리 — 모듈·Provider 배포·재사용

### Chapter 6: 드리프트와 운영 (5개 문서)
- 드리프트 — 수동 변경으로 코드와 실제 불일치, 감지
- 드리프트 감지 — plan/refresh로 차이 발견, 조정
- 멱등성 검증 — 재apply 시 no-op이어야, 멱등 깨지는 패턴
- 협업 — 워크스페이스·환경 분리, state 분리 전략
- 비밀·보안 — state의 민감 정보, 시크릿 관리

### Chapter 7: 실전과 CI/CD (4개 문서)
- 모듈 설계 — 재사용 모듈, 합성, 안티패턴
- CI/CD 통합 — plan을 PR에·apply를 머지에(cicd 연결), 자동화
- 테스트 — terraform validate·plan 검토·정책(OPA)
- 종합 — 작은 인프라를 모듈화·remote state·CI로 관리, 드리프트 복구 실습

→ **총 35개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 state·그래프·plan diff·Provider 프로토콜, `💻 실전 실험`은 plan/apply·state 조작·드리프트 재현(LocalStack). `📊`는 선언적 vs 명령형·remote state 비교.

## 🎨 스타일 가이드

1. **선언적 수렴으로 환원** — 모든 동작을 "현재→목표 수렴"으로(K8s와 공통)
2. **3자 비교 강조** — 코드(목표) vs state(기록) vs 실제(refresh)
3. **드리프트를 재현** — 수동 변경→plan이 잡아내는 것 *직접 본다*
4. **k8s/cicd 레포로 착지** — reconciliation은 k8s, 자동화는 cicd
5. **grpc 연결** — Provider 프로토콜이 gRPC임을 명시
6. 의존성 그래프·plan diff는 다이어그램으로

## 🔬 검증 환경

> docker(LocalStack으로 실제 클라우드 없이 실습).

```yaml
# docker-compose.yml — LocalStack(AWS 에뮬)
services:
  localstack:
    image: localstack/localstack:3
    ports: ["4566:4566"]
    environment: { SERVICES: "s3,ec2,dynamodb,iam" }
```

```bash
# 검증 방법 (LocalStack 대상, 비용 0)
terraform init
terraform plan      # diff 계산 — 무엇을 만들/바꿀지
terraform apply

# 1) state 까보기
terraform show              # 현재 state
cat terraform.tfstate | jq  # JSON 구조·리소스 매핑

# 2) plan = diff 검증
#    코드 수정 → plan → "~"(수정)·"-/+"(재생성) 기호 해석
terraform plan

# 3) 드리프트 재현:
#    awslocal s3 rb ... (Terraform 밖에서 수동 삭제)
#    terraform plan → 드리프트 감지(다시 만들려 함) 확인

# 4) replace 실험: force-new 속성 변경 → 재생성 계획 관찰
# 5) state lock: 동시 apply → 락 충돌 확인
# 6) 의존성 그래프: terraform graph | dot -Tsvg > graph.svg
# 7) 멱등성: 재apply → "No changes" 확인
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🖥️ Infrastructure 톤, "코드를 쓴다 vs 어떻게 자원으로 수렴하나" 포지셔닝, `🔗 레포 연결`(kubernetes·cicd·grpc)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- Terraform 공식 문서 — https://developer.hashicorp.com/terraform
- "Terraform: Up & Running" (Yevgeniy Brikman)
- Terraform Provider 프로토콜 — https://developer.hashicorp.com/terraform/plugin
- OpenTofu 문서
- LocalStack 문서 — https://docs.localstack.cloud/

## 💡 핵심 분석 대상

```
선언적 수렴 (K8s와 공통):
  코드: "S3 버킷 1개, EC2 2대 원함"
  Terraform: 현재 상태 확인 → 목표와 diff → 부족분만 생성
  → "어떻게 만들지" 안 적고 "무엇을 원하는지"만 (명령형과 차이)

3자 비교 (plan의 핵심):
  코드(목표)   : main.tf — 원하는 상태
  state(기록)  : terraform.tfstate — Terraform이 아는 상태
  실제(refresh): 클라우드 API로 읽은 진짜 상태
  plan = 목표 - (state ∪ refresh) = 변경 계획

드리프트:
  Terraform으로 만든 자원을 콘솔에서 수동 변경
  → state는 옛날 그대로, 실제는 바뀜 → 불일치(드리프트)
  → plan/refresh가 감지 → 코드 상태로 되돌리려 함

멱등성:
  apply 후 또 apply → "No changes"(이미 목표 상태)
  → 몇 번을 돌려도 같은 결과 (distributed 이론의 멱등)

Provider 프로토콜:
  Terraform Core ──gRPC──► Provider(aws/gcp) ──API──► 클라우드
  → Core는 클라우드를 모름, Provider가 번역 (grpc 레포 연결)
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 35개 확인 + LocalStack/plan-apply/드리프트 검증 환경 + kubernetes/cicd/grpc 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
