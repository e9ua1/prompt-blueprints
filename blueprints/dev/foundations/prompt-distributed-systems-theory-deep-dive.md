# Distributed Systems Theory Deep Dive 레포지토리 제작 프롬프트

나는 "Distributed Systems Theory Deep Dive" 레포지토리를 만들려고 해.
Kafka의 ISR, K8s의 etcd, Saga 보상 트랜잭션이 *전부 같은 이론의 다른 얼굴*이라는 걸 — 합의(Consensus)·시간·일관성·장애 모델을 통해 완전히 파헤치는 레포야.
"분산 도구를 쓰는 것"과 "왜 분산은 근본적으로 어려운지 알고 설계하는 것"의 차이를 만드는 레포다. 이 연구소의 흩어진 분산 레포들(Kafka·Kubernetes·MSA·Spring Cloud)을 **하나의 이론으로 묶는 수렴 레포**다.

## 📋 프로젝트 목표

**컨셉**: "분산 도구를 사용하는 것과, 분산 시스템이 왜 어렵고 무엇이 보장 가능한지 아는 것은 다르다"

**핵심 차별화**:
1. 합의는 모든 것의 중심 — Raft 하나를 제대로 이해하면 etcd·ZooKeeper·Kafka 컨트롤러가 보인다
2. 분산엔 "지금"이 없다 — 시계 동기화 불가, 논리 시계(Lamport/Vector)로 순서를 정의하는 법
3. 트레이드오프는 법칙이다 — CAP·PACELC, FLP 불가능성, "공짜 점심은 없다"의 수학적 근거
4. 충돌 없는 동시 편집 — CRDT가 락 없이 수렴하는 원리(local-first의 이론적 토대)

**타겟 독자**:
- Kafka ISR·K8s etcd를 쓰지만 그 밑의 합의 알고리즘이 같은 거라는 걸 모르는 개발자
- "최종 일관성"을 단어로만 알고 언제 안전한지 설명 못하는 개발자
- 분산 트랜잭션(Saga)을 구현하지만 왜 2PC를 안 쓰는지 모르는 개발자
- CAP를 "셋 중 둘"로만 외운 개발자(실제론 P 하에서 C vs A)
- `msa-deep-dive`/`kafka`/`kubernetes`를 봤지만 공통 이론이 없는 개발자

## 🔗 레포 연결

**⬆️ 선행**: 없음(이론 레포). `network-deep-dive`(메시지 전달·지연·분실 모델 이해에 도움).
**🤝 시너지**: `kafka-deep-dive`(ISR·리더 선출=합의), `kubernetes-deep-dive`(etcd=Raft), `msa-deep-dive`(Saga·분산 트랜잭션), `spring-cloud-deep-dive`(Eureka·Circuit Breaker), `local-first-sync-deep-dive`(CRDT 실전).
**🧬 수렴**: 위 모든 분산 레포의 *이론적 뿌리*. "Kafka도 K8s도 결국 합의 문제"를 보여주는 메타 레포.

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 설계해줘:

### Chapter 1: 분산이 어려운 이유 (6개 문서)
- 분산 시스템이란 — 부분 실패(partial failure)가 정의하는 세계, 단일 머신과 근본적으로 다른 점
- 8가지 오류 가정(Fallacies) — "네트워크는 신뢰할 수 있다" 등, 각 가정이 깨질 때의 사고
- 장애 모델 — Crash-stop / Crash-recovery / Omission / Byzantine, 무엇을 가정하느냐가 설계를 결정
- 비동기 네트워크 — 메시지 지연·재정렬·분실·중복, "죽은 노드와 느린 노드를 구분 못 한다"
- 타임아웃의 딜레마 — 너무 짧으면 오탐, 너무 길면 느린 감지, 완벽한 장애 감지기는 불가능
- 멱등성과 정확히 한 번 — at-most/at-least/exactly-once의 진짜 의미, 멱등성으로 재시도 안전하게

### Chapter 2: 시간과 순서 (5개 문서)
- 물리 시계의 한계 — 클럭 드리프트, NTP의 한계, "두 이벤트 중 뭐가 먼저인지" 못 정하는 이유
- Lamport 논리 시계 — happens-before 관계, 인과 순서를 카운터로 정의
- Vector Clock — 동시성(concurrent) 이벤트 탐지, 인과관계를 정확히 포착, 충돌 감지
- 전순서 vs 부분순서 — 전역 순서가 필요한 곳과 인과 순서로 충분한 곳
- TrueTime과 하이브리드 — Spanner의 시계 불확실성 구간, HLC(Hybrid Logical Clock)

### Chapter 3: 합의(Consensus) — 분산의 심장 (7개 문서)
- 합의 문제 정의 — 여러 노드가 하나의 값에 동의, 안전성(Safety) vs 활성(Liveness)
- FLP 불가능성 — 비동기 + 한 노드 장애 가능 → 결정적 합의 불가, 그래도 실용 합의가 되는 이유
- Paxos — Basic Paxos의 prepare/accept, 왜 어렵기로 악명 높은가, Multi-Paxos
- Raft 완전 분해 ① — 리더 선출, term, 하트비트, 선출 안전성
- Raft 완전 분해 ② — 로그 복제, 커밋 규칙, 로그 일관성 보장
- Raft 완전 분해 ③ — 안전성 증명 직관, 멤버십 변경, 스냅샷, 직접 구현/실행
- 쿼럼과 정족수 — 과반(N/2+1)이 일관성을 보장하는 원리, etcd/ZooKeeper/Kafka에서의 적용

### Chapter 4: 복제와 일관성 모델 (6개 문서)
- 복제 전략 — 단일 리더/다중 리더/리더 없음, 동기 vs 비동기 복제
- 일관성 스펙트럼 — Linearizability → Sequential → Causal → Eventual, 강도와 비용
- Linearizability — "단일 복사본처럼 보이기", 비용, 어떻게 테스트하나(Jepsen)
- 최종 일관성과 충돌 — 다중 리더의 쓰기 충돌, LWW(Last-Write-Wins)의 위험
- 정족수 일관성 — R + W > N 공식, Dynamo 스타일, 읽기 복구·헌티드 핸드오프
- 읽기 일관성 보장 — read-your-writes, monotonic reads, 세션 보장

### Chapter 5: CAP, PACELC와 트랜잭션 (5개 문서)
- CAP 정리 정확히 — "셋 중 둘"의 오해, P는 선택이 아니라 전제, 실제론 C vs A
- PACELC — 분할 없을 때도(Else) Latency vs Consistency 트레이드오프가 있다
- 2PC와 그 한계 — 2단계 커밋의 블로킹 문제, 코디네이터 장애, 왜 MSA에서 기피되나
- Saga 패턴 — 보상 트랜잭션, 오케스트레이션 vs 코레오그래피, 격리 부재 대응
- 분산 트랜잭션 대안 — Outbox·CDC, 사가 vs 이벤트 소싱, 최종 일관성 수용 설계

### Chapter 6: CRDT — 충돌 없는 동시 편집 (5개 문서)
- CRDT 직관 — 락·합의 없이 수렴하는 자료구조, 수학적 조건(교환·결합·멱등)
- 상태 기반 vs 연산 기반 — CvRDT vs CmRDT, 전파 방식의 차이
- 기본 CRDT — G-Counter/PN-Counter, G-Set/OR-Set, LWW-Register 구현·수렴 관찰
- 텍스트 협업 CRDT — 공동 편집(RGA/Yjs류)의 원리, OT와의 비교
- CRDT의 현실 — 메타데이터 증가(tombstone), 적합한 곳과 부적합한 곳, local-first 연결

### Chapter 7: 장애·관측·검증 (4개 문서)
- 장애 감지와 멤버십 — 하트비트, Φ Accrual, Gossip(SWIM) 프로토콜
- 분산 디버깅 — 인과 추적, 분산 트레이싱이 Vector Clock의 실전판인 이유
- Jepsen 스타일 검증 — 네트워크 분할 주입(`tc`/`iptables`)으로 일관성 위반 찾기
- 케이스 종합 — Kafka·etcd·Cassandra를 "어떤 합의/일관성 선택을 했나"로 재해석

→ 각 챕터 4~7개 문서. **총 42개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 알고리즘·증명 직관, `💻 실전 실험`은 docker-compose 멀티 노드 + 분할 주입으로 *직접 깨뜨려 본다*. `📊`는 일관성 모델별 비용·지연 비교.

## 🎨 스타일 가이드

1. **이론 → 실전 레포로 착지** — 모든 개념을 "이게 Kafka/etcd/Saga의 무엇이다"로 연결
2. **깨뜨려서 증명** — 네트워크 분할을 주입해 일관성 위반을 *눈으로* 보여준다(Jepsen 정신)
3. **불가능성을 직시** — FLP·CAP를 "그래서 포기"가 아니라 "그래서 무엇을 선택했나"로
4. **공식의 의미** — R+W>N, N/2+1을 외우지 말고 *왜 그 수인지* 유도
5. Raft는 상태 전이·로그를 항상 다이어그램으로

## 🔬 검증 환경

> 여기는 **docker-compose 멀티 노드**가 제격이다. 노드를 띄우고 네트워크를 끊어 본다.

```yaml
# docker-compose.yml — 3노드 클러스터 + 분할 실험
services:
  node1: &node
    build: ./cluster-node
    environment: { NODE_ID: 1, PEERS: "node1,node2,node3" }
  node2:
    <<: *node
    environment: { NODE_ID: 2, PEERS: "node1,node2,node3" }
  node3:
    <<: *node
    environment: { NODE_ID: 3, PEERS: "node1,node2,node3" }
  # 합의 관찰용 실제 시스템
  etcd:
    image: quay.io/coreos/etcd:v3.5
    command: etcd --name e1 ... # 3노드 etcd로 Raft 실관찰
```

```bash
# 핵심 실험

# 네트워크 분할 주입 (한 노드를 격리)
docker network disconnect <net> node3      # 또는
tc qdisc add dev eth0 root netem loss 100% delay 0ms   # 패킷 전량 손실

# etcd로 Raft 직접 관찰
etcdctl member list
etcdctl endpoint status --write-out=table   # 리더·term 확인
# 리더 죽이고 재선출 관찰 → term 증가

# 지연/유실 주입해 일관성 깨기
tc qdisc add dev eth0 root netem delay 500ms 200ms loss 10%

# CRDT 수렴 데모: 두 노드가 분할 상태에서 각자 편집 → 재연결 → 수렴 확인
python3 crdt_orset_demo.py
```

핵심: Raft 미니 구현을 띄우고 **리더를 죽여 재선출**, **분할로 split-brain 방지** 확인. CRDT는 분할 중 동시 편집 후 **수렴**을 관찰.

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `cluster-node/`(Raft 미니 구현)
2. **README.md**: 🪨 Foundations 톤, "도구를 쓴다 vs 왜 어려운지 안다" 포지셔닝, Kafka/K8s/MSA가 이 이론의 응용임을 명시, `🔗 레포 연결`
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- *Designing Data-Intensive Applications* (Martin Kleppmann) — 핵심 교재
- Raft 논문 *In Search of an Understandable Consensus Algorithm* — https://raft.github.io/
- *Distributed Systems* (van Steen & Tanenbaum, 무료)
- MIT 6.824 강의 — https://pdos.csail.mit.edu/6.824/
- Jepsen 분석 — https://jepsen.io/analyses
- CRDT 논문 (Shapiro et al.), Martin Kleppmann의 local-first 글

## 💡 핵심 분석 대상

```
"느린 노드 vs 죽은 노드" 구분 불가:
  A → B 요청, 응답 없음
  B가 죽었나? 네트워크가 느린가? 응답이 유실됐나?
  → 구분 불가능 → 모든 분산 설계의 출발점

Raft 리더 선출:
  Follower --(타임아웃)--> Candidate --(과반 득표)--> Leader
       ↑                                              │
       └──────(더 큰 term 발견)──────────────────────┘
  과반(N/2+1) 필요 → 분할돼도 한쪽만 과반 → split-brain 방지

CAP (P는 전제):
  네트워크 분할 발생 시:
    CP 선택: 일관성 위해 응답 거부(etcd/ZooKeeper)
    AP 선택: 가용성 위해 오래된 값 반환(Dynamo/Cassandra)
  분할 없을 땐? → PACELC: Latency vs Consistency 여전히 트레이드오프

정족수 (R + W > N):
  N=3, W=2, R=2 → 쓰기 2노드 + 읽기 2노드 = 최소 1노드 겹침
  → 읽기가 최신 쓰기를 반드시 포함 → 강한 일관성
  W=1,R=1 → 안 겹칠 수 있음 → 최종 일관성(빠름)

CRDT 수렴 (G-Counter):
  노드A: [A:3, B:0]   노드B: [A:0, B:5]   (분할 중 각자 증가)
  재연결 → merge = 각 원소 max → [A:3, B:5] → 합 8
  순서 무관·중복 무관 → 락 없이 항상 수렴
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목(4~7개) + 핵심 내용(3~4줄) + 총 42개 확인 + docker-compose 멀티노드/분할주입 환경 + Kafka/etcd/MSA 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
