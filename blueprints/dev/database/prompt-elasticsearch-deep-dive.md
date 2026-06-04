# Elasticsearch Deep Dive 레포지토리 제작 프롬프트

나는 "Elasticsearch Deep Dive" 레포지토리를 만들려고 해.
검색 기능을 구현하는 것과, Lucene이 역색인을 어떻게 구축하고, 분산 샤드에서 점수를 어떻게 계산하고 병합하는지, 집계(Aggregation)가 메모리에서 어떻게 동작하는지를 완전히 파헤치는 레포야.

database-internals에서 B-Tree 인덱스를 배웠다면, 이제 "왜 전문 검색(Full-Text Search)에는 B-Tree가 아닌 역색인(Inverted Index)이 쓰이는가"라는 질문으로 시작하는 레포야.
MySQL이 데이터를 어떻게 저장하는지 알지만, Elasticsearch가 완전히 다른 철학으로 설계된 이유를 배우는 레포야.

## 📋 프로젝트 목표

**컨셉**: "검색 기능을 구현하는 것과, Lucene이 역색인을 어떻게 구축하고 분산 샤드에서 스코어를 병합하는지 아는 것은 다르다"

**핵심 차별화**:
1. 역색인 완전 분해 — B-Tree와 근본적으로 다른 Inverted Index가 왜 검색에 최적인가
2. 분산 아키텍처 — 샤드와 레플리카가 어떻게 데이터를 분산하고 Scatter-Gather로 검색하는가
3. 스코어링 원리 — BM25가 어떻게 "관련성"을 숫자로 계산하는가
4. 집계의 실체 — Terms/Date Histogram이 메모리에서 어떻게 동작하며 왜 힙을 많이 쓰는가

**타겟 독자**:
- match 쿼리를 쓰지만 내부에서 tokenize → posting list 탐색이 일어나는 걸 모르는 개발자
- 인덱스를 생성하지만 샤드 수가 성능에 미치는 영향을 모르는 개발자
- Aggregation이 느린 이유를 fielddata vs doc_values로 설명 못하는 개발자
- 매핑(Mapping)을 설정하지만 analyzer 체인이 토큰을 어떻게 변환하는지 모르는 개발자
- Elasticsearch가 왜 NoSQL인지, MySQL과 어떤 트레이드오프가 있는지 설명 못하는 개발자
- Near Real-Time(NRT) 검색이 왜 "Near"인지, refresh와 flush의 차이를 모르는 개발자

**선행 학습**:
- database-internals (B-Tree 인덱스 구조 이해 필수 — 역색인과 비교를 위해)
- linux-for-backend-deep-dive (Page Cache, mmap이 Lucene 세그먼트 접근과 연결, 시너지 큼)
- network-deep-dive (REST API 기반 클러스터 통신, 시너지 있음)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Elasticsearch 아키텍처 — 분산 시스템의 기초 (5개 문서)
- 전체 아키텍처 개요 — 클러스터/노드/인덱스/샤드/세그먼트 계층 구조, Elasticsearch가 Lucene을 분산화하는 방식
- 클러스터 상태와 Master 노드 — 클러스터 메타데이터(인덱스 매핑, 샤드 배치), Master Election(Bully → Raft 기반 변화), Split-Brain 문제
- 샤드와 레플리카 — Primary/Replica 역할 분리, 샤드 수를 나중에 변경할 수 없는 이유(라우팅 수식), Reindex 없이 샤드를 늘리지 못하는 이유
- 쓰기 경로 — Document가 Primary → Replica로 복제되는 과정, `wait_for_active_shards` 설정의 의미, 쓰기 실패와 재시도 처리
- 읽기 경로 — Scatter-Gather: Coordinator가 모든 샤드에 요청을 분산하고 결과를 병합하는 방식, 적응형 레플리카 선택

### Chapter 2: Lucene 역색인 — 검색의 본질 (6개 문서)
- B-Tree vs Inverted Index — 왜 검색에는 B-Tree가 아닌 역색인이 쓰이는가, 각각의 강점과 약점 비교
- 역색인 구조 완전 분해 — Term Dictionary, Posting List, Term Frequencies, Position/Offset 정보가 저장되는 방식
- FST(Finite State Transducer) — Term Dictionary를 메모리에 압축 저장하는 방법, O(len) 탐색이 가능한 이유
- Lucene Segment — 세그먼트가 불변(Immutable)인 이유, 쓰기가 새 세그먼트를 생성하는 방식, 세그먼트 병합 정책
- Near Real-Time 검색 — refresh(메모리 버퍼 → 새 세그먼트), flush(세그먼트 → 디스크 fsync), translog 내구성 보장 역할
- doc_values vs fielddata — 정렬/집계를 위한 컬럼 지향 저장소, 힙 외부(off-heap) mmap vs 힙 내부 fielddata의 메모리 차이

### Chapter 3: 텍스트 분석 — 토큰이 만들어지는 과정 (5개 문서)
- 분석 파이프라인 — Character Filter → Tokenizer → Token Filter 체인, "Hello World!"가 ["hello", "world"]가 되는 과정
- Standard/Nori Analyzer — Standard Analyzer의 토큰화 규칙, Nori(한국어) 형태소 분석기의 분리 원리
- 커스텀 Analyzer 설계 — 동의어 처리, 불용어(Stopwords), Stemming, n-gram 토큰 생성 방식
- 매핑(Mapping) 완전 가이드 — text vs keyword 차이, dynamic mapping의 편의성과 함정, 매핑 폭발(Mapping Explosion)
- 인덱스 시점 vs 검색 시점 분석 — 두 분석기가 다를 때 검색이 실패하는 원리, `_analyze` API로 토큰 디버깅

### Chapter 4: 쿼리 엔진 — 검색과 스코어링 (6개 문서)
- Query DSL 내부 구조 — Query Context vs Filter Context, 스코어 계산 여부와 캐시 여부 차이
- BM25 스코어링 완전 분해 — TF(Term Frequency), IDF(Inverse Document Frequency), 필드 길이 정규화가 점수에 미치는 영향
- 분산 스코어링 문제 — 샤드마다 IDF가 다른 이유(로컬 IDF), 글로벌 IDF를 사용하는 DFS_QUERY_THEN_FETCH 방식
- Query 실행 계획 — `_explain` API로 쿼리가 어떻게 점수를 계산하는지 확인, Lucene의 Boolean Query 최적화
- 주요 쿼리 유형 비교 — match/term/range/bool/nested 내부 동작 차이, 각 쿼리의 비용 수준
- 벡터 검색(kNN) — Dense Vector 저장 방식, HNSW(Hierarchical Navigable Small World) 알고리즘, BM25와 하이브리드 검색

### Chapter 5: 집계(Aggregation) — 분석의 원리 (5개 문서)
- 집계 아키텍처 — Bucket/Metric/Pipeline 집계 계층, 집계가 분산 샤드에서 실행되고 병합되는 방식
- Terms Aggregation 내부 — 각 샤드에서 top-N을 구하고 병합할 때 발생하는 오차, `shard_size` 튜닝으로 정확도 향상
- Date Histogram 집계 — 시간 기반 버킷 생성, 타임존 처리, 캘린더 인식 간격(month/quarter) 계산 원리
- 집계와 메모리 — fielddata가 힙을 얼마나 사용하는가, Circuit Breaker가 OOM을 막는 방식, `indices.fielddata.cache.size` 설정
- 집계 성능 최적화 — filter aggregation으로 캐시 활용, eager_global_ordinals로 시작 시 warm-up, pre-aggregation 패턴

### Chapter 6: 운영과 성능 튜닝 (6개 문서)
- 인덱스 설계 전략 — 샤드 크기 가이드라인(10~50GB), 시계열 인덱스(인덱스 롤오버), 핫-웜-콜드 아키텍처(ILM)
- 클러스터 성능 진단 — `_cat/nodes`, `_cluster/health`, `_nodes/stats` 핵심 지표 해석, 느린 쿼리 로그
- 힙 메모리 튜닝 — 힙 크기를 RAM의 50% 이하로 제한하는 이유(OS Page Cache 확보), GC 오버헤드 줄이기
- 쓰기 성능 최적화 — refresh_interval=-1(쓰기 집중 시), bulk API 배치 처리, translog 설정
- 검색 성능 최적화 — Filter Cache 활용, Request Cache, 샤드 사전 로딩(_warmer), shard preference
- 운영 중 발생하는 문제 패턴 — Yellow/Red 상태 원인, 샤드 미할당(Unassigned Shards) 해결, Split-Brain 복구

### Chapter 7: Spring과 Elasticsearch 통합 (4개 문서)
- Spring Data Elasticsearch — Repository 패턴, `ElasticsearchOperations`, 매핑 어노테이션(@Document/@Field)
- 인덱싱 전략 — Spring에서 동기/비동기 인덱싱, Bulk 인덱싱 구현, 실패 재처리 패턴
- 검색 쿼리 작성 — NativeQuery vs CriteriaQuery vs StringQuery 비교, QueryBuilder 조합 방식
- 데이터 동기화 패턴 — MySQL → Elasticsearch 동기화(CDC, Debezium), 검색과 DB의 정합성 관리

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **37개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 실무에서 중요한가
## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)
## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)
## 🔬 내부 동작 원리 (Lucene/ES 내부 분석, 실제 인덱스 파일 구조)
## 💻 실전 실험 (Kibana Dev Tools, _explain API, _analyze API 등)
## 📊 성능/비용 비교 (B-Tree vs 역색인, fielddata vs doc_values, 쿼리 비용 등)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. MySQL/database-internals와 대비 — B-Tree vs 역색인, 행 지향 vs 문서 지향을 항상 대비하여 설명
2. Kibana Dev Tools와 `_explain`, `_analyze`, `_cat` API로 바로 재현할 수 있는 실험 포함
3. 각 설계 결정의 이유 명시 ("왜 세그먼트는 불변인가", "왜 샤드 수를 나중에 못 바꾸는가")
4. Spring Data Elasticsearch 연결 — 어노테이션 설정이 내부 매핑과 어떻게 연결되는지
5. 운영 장애 시나리오 → `_cat`, `_cluster` API 진단 명령어 세트

**실험 환경**:
```yaml
# docker-compose.yml
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - esdata:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    depends_on:
      - elasticsearch
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"

  elasticsearch-exporter:
    image: quay.io/prometheuscommunity/elasticsearch-exporter:latest
    command:
      - '--es.uri=http://elasticsearch:9200'
    ports:
      - "9114:9114"

volumes:
  esdata:
```

```bash
# 실험용 공통 명령어 세트 (Kibana Dev Tools 또는 curl)

# 분석기 동작 확인
GET /myindex/_analyze
{
  "analyzer": "standard",
  "text": "Hello World! Quick brown fox."
}

# 쿼리 점수 계산 과정 설명
GET /myindex/_explain/1
{
  "query": {
    "match": { "title": "elasticsearch" }
  }
}

# 클러스터 상태 확인
GET _cluster/health?pretty
GET _cat/shards?v&s=index
GET _cat/nodes?v&h=name,heap.percent,ram.percent,cpu,load_1m

# 느린 쿼리 로그 설정
PUT /myindex/_settings
{
  "index.search.slowlog.threshold.query.warn": "2s",
  "index.search.slowlog.threshold.query.info": "1s"
}

# 세그먼트 현황 확인
GET /myindex/_segments?pretty
GET _cat/segments/myindex?v

# fielddata 메모리 사용량
GET _nodes/stats/indices/fielddata?fields=*&pretty
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**:
   - mysql-deep-dive README 스타일 참고
   - "SQL을 쓰는 것과 Lucene 역색인을 아는 것의 차이"라는 포지셔닝 강조
   - database-internals, mysql-deep-dive와의 대비 연결 지점 명시
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료 섹션

README 작성 시 다음을 Reference로 포함해줘:
- Elasticsearch 공식 문서 — https://www.elastic.co/guide/en/elasticsearch/reference/current/
- Elasticsearch: The Definitive Guide (Clinton Gormley & Zachary Tong) — https://www.elastic.co/guide/en/elasticsearch/guide/current/
- Lucene in Action, 2nd Edition (Michael McCandless et al.)
- Elasticsearch 소스 코드 — https://github.com/elastic/elasticsearch
- Elastic Blog — https://www.elastic.co/blog/
- Adrien Grand's Blog (Lucene 코어 개발자) — https://www.elastic.co/blog/author/adrien-grand

---

## 💡 핵심 분석 대상

```
검색 쿼리 전체 흐름:

GET /products/_search
{
  "query": { "match": { "description": "fast elasticsearch" } }
}
  │
  ▼ 1. Coordinator 노드 수신
  │
  ▼ 2. Query Phase (Scatter)
  │   모든 샤드에 요청 병렬 전송
  │   각 샤드에서:
  │   ├── "fast elasticsearch" → Analyzer → ["fast", "elasticsearch"]
  │   ├── Term Dictionary에서 "fast" 검색 (FST 탐색)
  │   ├── Posting List에서 해당 문서 ID 목록 추출
  │   ├── "elasticsearch" 동일 처리
  │   ├── 두 Posting List 교집합/합집합 (Boolean Query)
  │   └── BM25 점수 계산 후 상위 N개 문서 ID + 점수 반환
  │
  ▼ 3. Merge Phase
  │   Coordinator가 모든 샤드의 상위 N개를 병합
  │   전체 상위 K개 문서 ID 선정
  │
  ▼ 4. Fetch Phase (Gather)
  │   선정된 문서 ID에 해당하는 샤드에서 실제 _source 데이터 가져오기
  │
  ▼ 5. 응답 반환

분산 스코어링 문제:
  샤드 1: IDF("elasticsearch") = log(1000/800) → 문서 많음 → 낮은 IDF
  샤드 2: IDF("elasticsearch") = log(100/5)  → 문서 적음 → 높은 IDF
  → 같은 문서가 어느 샤드에 있느냐에 따라 점수가 달라짐!

  해결책 1 (기본): 로컬 IDF 무시 (대규모 클러스터에서는 자연스럽게 수렴)
  해결책 2 (정확): search_type=dfs_query_then_fetch
    → 먼저 모든 샤드에서 글로벌 통계 수집
    → 글로벌 IDF로 점수 계산 (요청 2배)

B-Tree vs 역색인 비교:

  MySQL (B-Tree):
    "title LIKE '%elasticsearch%'" → Full Table Scan (B-Tree 무용)
    "title = 'elasticsearch'" → B-Tree 탐색 O(log N), 빠름
    단어를 포함하는 문서 찾기 → 비효율

  Elasticsearch (역색인):
    Term Dictionary: {"elasticsearch": [3, 7, 42, 101], ...}
    "elasticsearch" 검색 → Term Dictionary에서 O(len) → Posting List [3,7,42,101]
    → 즉시 해당 문서 ID 목록 획득

  역색인의 제약:
    "문서 ID 42의 5번째 필드 값은?" → 불가 (역방향 조회)
    → _source 저장 또는 doc_values로 해결
    → 이 때문에 UPDATE가 비싸고 Immutable Segment 설계

세그먼트 불변성과 NRT:
  문서 인덱싱 → 인메모리 버퍼
    ↓ refresh (기본 1초마다)
  새 세그먼트 생성 (검색 가능, 디스크 미반영)
    ↓ flush (translog 크기 초과 or 명시적 호출)
  디스크에 fsync (내구성 보장)

  "Near Real-Time"인 이유:
    디스크가 아닌 OS 파일 시스템 캐시(Page Cache)에만 있어도 검색 가능
    refresh=1초 → 최대 1초 지연
    → 실시간이 아닌 "거의 실시간"
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (37개 목표)
- Docker Compose 실험 환경 (ES + Kibana + Exporter)
- database-internals, mysql-deep-dive, Spring 레포와 연결되는 지점 명시 (어느 문서에서 어느 레포와 연결하는지)

**준비됐으면 1단계 구조 설계부터 시작해줘!**
