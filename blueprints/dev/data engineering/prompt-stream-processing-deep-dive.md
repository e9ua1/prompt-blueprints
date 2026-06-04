# Stream Processing Deep Dive 레포지토리 제작 프롬프트

나는 "Stream Processing Deep Dive" 레포지토리를 만들려고 해.
끝없는 데이터 흐름을 어떻게 처리하는지 — 이벤트 시간 vs 처리 시간, Watermark, 윈도잉, 상태 관리, 체크포인트로 Exactly-Once를 보장하는 방식(Flink 중심)을 완전히 파헤치는 레포야.
"스트림을 처리하는 것"과 "왜 이벤트가 늦게 와도 정확히 집계되고 장애 후에도 정확히 한 번 처리되는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "스트림을 처리하는 것과, 이벤트 시간·Watermark·상태·체크포인트가 정확성을 어떻게 보장하는지 아는 것은 다르다"

**핵심 차별화**:
1. 이벤트 시간 vs 처리 시간 — "언제 일어났나" vs "언제 처리하나", 늦은 이벤트 문제
2. Watermark — 늦은 데이터를 기다리는 메커니즘, 완전성과 지연의 트레이드오프
3. 상태와 체크포인트 — 스트림 연산의 상태를 어떻게 보관·복구하나
4. Exactly-Once — 장애가 나도 정확히 한 번 처리되는 보장의 실체

**타겟 독자**:
- 스트림을 처리하지만 이벤트 시간·Watermark를 모르는 개발자
- 늦게 온 이벤트가 왜 집계에서 빠지는지 모르는 개발자
- "Exactly-Once"를 단어로만 아는 개발자
- 배치(Spark)와 스트림의 차이를 모르는 개발자
- `kafka`·`distributed-systems-theory`를 스트림으로 확장하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `kafka-deep-dive`(스트림 소스·오프셋), `distributed-systems-theory-deep-dive`(상태·일관성·Exactly-Once).
**🤝 시너지**: `spark-internals-deep-dive`(배치 vs 스트림), `columnar-storage-format-deep-dive`(싱크), `realtime-client-networking-deep-dive`(클라 스트림 대조).
**🧬 수렴**: 분산 이론의 Exactly-Once·상태 머신 응용. 데이터 레이어 완성.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 스트림 처리의 개념 (5개 문서)
- 배치 vs 스트림 — 유한 vs 무한 데이터, "데이터가 끝나지 않는다"
- 스트림 처리 모델 — 레코드별·마이크로배치, Flink vs Spark Streaming
- 왜 어려운가 — 순서·지연·장애·상태가 무한 흐름에서 만드는 문제
- 처리 보장 — at-most/at-least/exactly-once(distributed 연결)
- 아키텍처 — Source→연산→Sink, 분산 실행

### Chapter 2: 시간과 Watermark (6개 문서)
- 세 가지 시간 — 이벤트 시간·수집 시간·처리 시간, 왜 구분하나
- 이벤트 시간 처리 — "언제 일어났나" 기준 집계, 정확성의 기반
- 늦은 이벤트 — 네트워크·재시도로 순서 뒤바뀜, 무한정 기다릴 수 없음
- Watermark — "이 시간 이전 이벤트는 다 왔다"는 표시, 진행 신호
- Watermark 생성 — 단조 증가·휴리스틱, 지연 허용(bounded out-of-orderness)
- 늦은 데이터 처리 — allowed lateness·side output, 완전성 vs 지연

### Chapter 3: 윈도잉 (6개 문서)
- 윈도우 개념 — 무한 스트림을 유한 청크로, 집계 단위
- 텀블링 윈도우 — 고정·비겹침, 기본 집계
- 슬라이딩 윈도우 — 겹치는 윈도우, 이동 평균
- 세션 윈도우 — 활동 간격 기반, 동적 경계
- 윈도우 트리거 — 언제 윈도우 결과를 방출, Watermark와 결합
- 윈도우 + 늦은 데이터 — 윈도우 마감 후 온 이벤트 처리

### Chapter 4: 상태 관리 (6개 문서)
- 상태가 필요한 이유 — 집계·조인·중복 제거는 과거 기억 필요
- 상태 종류 — keyed state·operator state, value/list/map 상태
- 상태 백엔드 — 힙 vs RocksDB, 큰 상태의 디스크 저장
- 상태 크기 관리 — TTL·정리, 무한 증가 방지
- 상태와 키 — 키별 상태 파티셔닝, 재분배(distributed 연결)
- 상태 조회·진화 — 상태 스키마 변경, 마이그레이션

### Chapter 5: 체크포인트와 Exactly-Once (6개 문서)
- 체크포인트 — 상태 스냅샷을 주기적 저장, 장애 복구 기반
- Chandy-Lamport — 분산 스냅샷 알고리즘(distributed 연결), barrier
- barrier 정렬 — 스트림에 마커 삽입, 일관된 스냅샷
- 정확히 한 번 — 체크포인트 + 멱등/트랜잭션 싱크로 Exactly-Once
- 소스·싱크 보장 — Kafka 오프셋·트랜잭션, end-to-end Exactly-Once
- 장애 복구 — 마지막 체크포인트에서 재시작, 상태·오프셋 복원

### Chapter 6: 조인과 고급 패턴 (5개 문서)
- 스트림 조인 — 윈도우 조인, 두 스트림 결합
- 스트림-테이블 조인 — 참조 데이터 enrichment
- CEP — 복합 이벤트 처리, 패턴 탐지
- 백프레셔 — 느린 다운스트림 대응, 흐름 제어(realtime 연결)
- 정확성 vs 지연 vs 처리량 — 스트림의 핵심 트레이드오프

### Chapter 7: 운영과 측정 (4개 문서)
- 모니터링 — 처리량·지연·체크포인트 시간·백프레셔
- 스케일·재조정 — 병렬성 변경, 상태 재분배
- 흔한 함정 — Watermark 잘못 설정·상태 폭발·체크포인트 실패
- 종합 — 이벤트 시간 집계 파이프라인을 Watermark+체크포인트로 구현·장애 복구 검증

→ **총 36개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 Watermark·체크포인트·상태 백엔드, `💻 실전 실험`은 Flink 로컬·장애 주입 복구·늦은 이벤트. `📊`는 윈도우·보장 수준의 정확성/지연 비교.

## 🎨 스타일 가이드

1. **이벤트 시간으로 환원** — 정확성 문제를 이벤트 시간·Watermark로
2. **장애를 주입** — 체크포인트 복구·Exactly-Once를 *깨뜨려 검증*(distributed 정신)
3. **늦은 이벤트 재현** — 순서 뒤바뀐 입력→집계 정확성 관찰
4. **kafka/distributed 레포로 착지** — 소스는 kafka, 보장은 distributed
5. Watermark·체크포인트 barrier는 다이어그램으로

## 🔬 검증 환경

> docker-compose(Flink + Kafka).

```yaml
# docker-compose.yml — Flink + Kafka
services:
  jobmanager:
    image: flink:1.18
    command: jobmanager
    ports: ["8081:8081"]   # Flink UI
  taskmanager:
    image: flink:1.18
    command: taskmanager
    depends_on: [jobmanager]
  kafka:
    image: bitnami/kafka:3.6
    ports: ["9092:9092"]
```

```bash
# 검증 방법
# 1) 이벤트 시간 집계: 타임스탬프 있는 이벤트 → 윈도우 집계
# 2) 늦은 이벤트 실험: 순서 뒤섞인 이벤트 주입 → Watermark·allowed lateness 효과
# 3) Watermark 관찰: Flink UI에서 Watermark 진행 확인
# 4) 체크포인트: Flink UI 체크포인트 탭, 주기·크기·시간
# 5) Exactly-Once 검증: 처리 중 TaskManager 죽이기 → 재시작 → 중복/누락 0 확인
# 6) 상태 백엔드: 힙 vs RocksDB로 큰 상태 처리 비교
# 7) 백프레셔: 느린 싱크 → Flink UI 백프레셔 지표
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 📊 Data Engineering 톤, "스트림을 처리한다 vs 정확성을 어떻게 보장하나" 포지셔닝, `🔗 레포 연결`(kafka·distributed·spark)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- *Streaming Systems* (Tyler Akidau et al.) — 핵심 교재
- Flink 공식 문서 — https://nightlies.apache.org/flink/flink-docs-stable/
- "The Dataflow Model" 논문 (Google)
- "Streaming 101/102" (Tyler Akidau, O'Reilly)
- Kafka Streams 문서(대안 비교)

## 💡 핵심 분석 대상

```
이벤트 시간 vs 처리 시간:
  이벤트 발생: 12:00 (이벤트 시간)
  네트워크 지연·재시도 → 시스템 도착: 12:05 (처리 시간)
  처리 시간 집계 → 12:00 이벤트가 12:05 윈도우에(틀림!)
  이벤트 시간 집계 → 12:00 윈도우에 정확히(올바름)

Watermark (늦은 데이터 vs 진행):
  "Watermark(12:00) = 12:00 이전 이벤트는 다 왔다고 가정"
  → 12:00 윈도우 마감·결과 방출 가능
  너무 빠른 Watermark → 늦은 이벤트 누락
  너무 느린 Watermark → 결과 지연
  → 완전성 vs 지연 트레이드오프

체크포인트 (Chandy-Lamport):
  스트림에 barrier 삽입 → 각 연산이 barrier 받으면 상태 스냅샷
  모든 연산 스냅샷 = 일관된 글로벌 체크포인트
  장애 → 마지막 체크포인트에서 상태·오프셋 복원

Exactly-Once (end-to-end):
  소스(Kafka 오프셋) + 상태(체크포인트) + 싱크(트랜잭션/멱등)
  장애 → 체크포인트 복원 + 오프셋 되감기 + 싱크 롤백
  → 중복도 누락도 없음 (distributed 이론의 실전)
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 36개 확인 + Flink+Kafka/장애주입 검증 환경 + kafka/distributed/spark 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
