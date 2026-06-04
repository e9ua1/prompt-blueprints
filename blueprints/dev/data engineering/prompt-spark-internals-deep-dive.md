# Spark Internals Deep Dive 레포지토리 제작 프롬프트

나는 "Spark Internals Deep Dive" 레포지토리를 만들려고 해.
분산 데이터 처리가 어떻게 동작하는지 — DAG 스케줄러, Stage·Task 분할, Shuffle, Catalyst 옵티마이저, Tungsten 메모리 관리를 완전히 파헤치는 레포야.
"Spark로 데이터를 처리하는 것"과 "왜 어떤 연산이 Shuffle을 일으키고 어떻게 메모리에서 OOM이 나는지 알고 튜닝하는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "Spark API를 호출하는 것과, DAG가 어떻게 Stage로 나뉘고 Shuffle·메모리가 성능을 결정하는지 아는 것은 다르다"

**핵심 차별화**:
1. 지연 실행과 DAG — transformation이 즉시 실행 안 되고 action에서 DAG로 실행되는 원리
2. Shuffle이 비용의 중심 — 어떤 연산이 데이터 재분배를 일으키고 왜 느린가
3. Catalyst & Tungsten — 쿼리 최적화와 메모리/코드 생성이 성능을 끌어올리는 방식
4. 분산 실행 — Driver/Executor, 파티션·Task, 데이터 지역성

**타겟 독자**:
- Spark를 쓰지만 lazy evaluation·DAG를 모르는 개발자
- Shuffle이 왜 느린지·언제 생기는지 모르는 개발자
- OOM·데이터 스큐를 만나지만 원인을 모르는 개발자
- RDD vs DataFrame 차이를 성능으로 설명 못하는 개발자
- `distributed-systems-theory`·`jvm`을 데이터 처리로 확장하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `distributed-systems-theory-deep-dive`(분산 실행·장애·파티셔닝), `jvm-deep-dive`(Spark는 JVM·GC).
**🤝 시너지**: `stream-processing-deep-dive`(배치 vs 스트림), `columnar-storage-format-deep-dive`(Parquet 읽기), `kafka-deep-dive`(소스), `database-internals`(Join 알고리즘 대조).
**🧬 수렴**: 분산 이론의 데이터 처리 응용. `columnar`·`stream`과 함께 데이터 레이어.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: Spark 실행 모델 (6개 문서)
- Spark 아키텍처 — Driver·Executor·Cluster Manager, 역할 분담
- 지연 실행 — transformation(지연) vs action(실행), 왜 지연인가
- RDD — 불변 분산 컬렉션, 파티션·계보(lineage), 장애 복구
- DataFrame/Dataset — 구조화 API, RDD 대비 최적화 이점
- 파티션 — 데이터 분할 단위, 병렬성의 기본, 파티션 수 결정
- 실행 흐름 한눈에 — 코드→논리 계획→물리 계획→DAG→Task

### Chapter 2: DAG와 스케줄링 (6개 문서)
- DAG 구성 — 연산 그래프, action이 Job을 트리거
- Stage 분할 — Shuffle 경계로 Stage를 나누는 원리(narrow vs wide)
- narrow vs wide 의존성 — map(narrow) vs groupBy(wide), Shuffle 유무
- Task — Stage 내 파티션당 Task, 병렬 실행
- DAG 스케줄러 — Stage 순서·재시도, Task 스케줄러로 위임
- 데이터 지역성 — Task를 데이터 가까이, PROCESS/NODE/RACK 로컬

### Chapter 3: Shuffle 완전 분해 (6개 문서)
- Shuffle이란 — 파티션 간 데이터 재분배, 네트워크·디스크 비용
- 언제 발생 — groupBy·join·repartition·distinct, wide 의존성
- Shuffle 메커니즘 — map 측 쓰기·reduce 측 읽기, Shuffle 파일
- Shuffle 비용 — 직렬화·디스크·네트워크, 왜 가장 비싼가
- 데이터 스큐 — 한 파티션에 데이터 쏠림, OOM·느린 Task의 주범
- Shuffle 최적화 — 파티션 수·broadcast join·salting으로 스큐 완화

### Chapter 4: Catalyst 옵티마이저 (5개 문서)
- Catalyst 개요 — 쿼리 최적화 엔진, 논리→물리 계획
- 논리 최적화 — predicate pushdown·projection pruning·상수 폴딩(compiler 연결)
- 물리 계획 — Join 전략 선택(broadcast/sort-merge/shuffle-hash)
- Cost 기반 최적화 — 통계 기반 계획 선택, AQE(Adaptive Query Execution)
- 계획 읽기 — explain()으로 논리/물리 계획·Shuffle 확인

### Chapter 5: Tungsten과 메모리 (6개 문서)
- Tungsten — 메모리·CPU 효율 엔진, off-heap·캐시 친화(computer-architecture 연결)
- 메모리 관리 — 실행 메모리 vs 저장 메모리, 통합 메모리 모델
- 코드 생성 — Whole-Stage Codegen으로 인터프리터 오버헤드 제거
- 캐싱·영속화 — cache/persist 레벨, 메모리 vs 디스크
- 직렬화 — Kryo vs Java, 직렬화가 성능에 주는 영향
- OOM 원인 — Executor 메모리·스큐·큰 셔플, 진단·해결

### Chapter 6: 데이터 소스와 통합 (5개 문서)
- 파일 소스 — Parquet/ORC 읽기, 컬럼 프루닝(columnar 연결)
- predicate pushdown — 필터를 소스로 내려 읽기 최소화
- 파티셔닝된 데이터 — 디렉토리 파티션, 파티션 프루닝
- Kafka·스트리밍 소스 — 배치로 읽기(stream-processing 연결)
- 쓰기 — 파티셔닝·버킷팅·작은 파일 문제

### Chapter 7: 성능과 운영 (4개 문서)
- Spark UI — Job/Stage/Task, Shuffle·스큐·메모리 진단
- 튜닝 — 파티션·메모리·병렬성·join 전략 조정
- 흔한 함정 — collect()·스큐·작은 파일·과한 셔플
- 종합 — 느린 Job을 Spark UI로 진단(스큐·셔플)→튜닝→before/after

→ **총 38개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 DAG·Shuffle·Catalyst·Tungsten, `💻 실전 실험`은 Spark UI·explain()·로컬 클러스터. `📊`는 join 전략·파티션·셔플의 정량 비교.

## 🎨 스타일 가이드

1. **DAG/Shuffle로 환원** — 모든 성능을 Stage 분할·Shuffle 비용으로
2. **explain()으로 증명** — 계획·Shuffle을 *직접 본다*
3. **스큐를 재현** — 데이터 쏠림→느린 Task→해결을 측정으로
4. **distributed/columnar 레포로 착지** — 분산은 distributed, 저장은 columnar
5. DAG·Shuffle·메모리 모델은 다이어그램으로

## 🔬 검증 환경

> docker-compose(Spark 클러스터) 또는 로컬 Spark.

```yaml
# docker-compose.yml — Spark 클러스터
services:
  spark-master:
    image: bitnami/spark:3.5
    environment: { SPARK_MODE: master }
    ports: ["8080:8080", "7077:7077"]   # UI, master
  spark-worker:
    image: bitnami/spark:3.5
    environment: { SPARK_MODE: worker, SPARK_MASTER_URL: "spark://spark-master:7077" }
    deploy: { replicas: 2 }
```

```bash
# 검증 방법
# 1) explain(): df.explain(true) → 논리/물리 계획·Exchange(Shuffle) 확인
spark.sql("...").explain("formatted")
# 2) Spark UI(:8080·:4040): Job→Stage→Task, Shuffle Read/Write, 스큐 Task
# 3) Shuffle 관찰: groupBy vs map → Stage 수·Exchange 비교
# 4) 스큐 재현: 한 키에 데이터 몰아 → 느린 Task → salting으로 분산
# 5) broadcast join: 작은 테이블 broadcast → Shuffle 제거 확인
# 6) AQE on/off: 적응 최적화 효과 비교
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 📊 Data Engineering 톤, "API를 호출한다 vs DAG·Shuffle을 안다" 포지셔닝, `🔗 레포 연결`(distributed·jvm·columnar)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- *Spark: The Definitive Guide* (Chambers & Zaharia)
- *Learning Spark, 2nd* (무료 일부)
- Spark 공식 문서·튜닝 가이드 — https://spark.apache.org/docs/latest/tuning.html
- "Deep Dive into Spark SQL's Catalyst Optimizer" (Databricks 블로그)
- Spark 소스 — https://github.com/apache/spark

## 💡 핵심 분석 대상

```
지연 실행 + DAG:
  val r = data.filter(...).map(...).groupBy(...)   // 아무것도 실행 안 됨(lazy)
  r.count()   // action! → 여기서 DAG 생성·실행
  → 최적화 기회(필터를 먼저, 불필요 연산 제거)

Stage 분할 (Shuffle 경계):
  filter → map  : narrow(파티션 독립) → 같은 Stage
  groupBy/join  : wide(파티션 간 재분배) → Shuffle → 새 Stage
  → Stage 경계 = Shuffle 경계

Shuffle (가장 비싼 연산):
  map 측: 각 파티션이 key별로 분류·디스크 쓰기
  네트워크: reduce 측으로 전송
  reduce 측: 같은 key 모아 처리
  → 직렬화 + 디스크 + 네트워크 = 느림

데이터 스큐:
  groupBy(key) 에서 한 key에 90% 데이터
  → 한 Task만 거대 → 느림·OOM (다른 Task는 놀고)
  → salting: key에 랜덤 접미사 → 분산 → 재집계

Catalyst Join 선택:
  작은 테이블 → Broadcast Join(셔플 없음, 작은 쪽 복제)
  둘 다 큼   → Sort-Merge Join(양쪽 셔플·정렬)
  explain으로 어떤 join 골랐는지 확인
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 38개 확인 + Spark클러스터/UI/explain 검증 환경 + distributed/jvm/columnar 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
