# Columnar & Storage Format Deep Dive 레포지토리 제작 프롬프트

나는 "Columnar & Storage Format Deep Dive" 레포지토리를 만들려고 해.
분석 데이터가 왜 행이 아닌 열로 저장되는지 — Parquet/ORC의 인코딩·압축, Predicate Pushdown, 그리고 Iceberg/Delta 같은 Lakehouse 테이블 포맷을 완전히 파헤치는 레포야.
"Parquet로 저장하는 것"과 "왜 컬럼 저장이 분석에 빠르고 어떻게 스캔을 건너뛰는지 알고 설계하는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "데이터를 Parquet로 저장하는 것과, 컬럼 저장·인코딩·통계가 분석 쿼리를 어떻게 빠르게 하는지 아는 것은 다르다"

**핵심 차별화**:
1. 행 vs 열 저장 — 분석 쿼리(일부 컬럼·집계)에 컬럼 저장이 빠른 근본 이유
2. 인코딩과 압축 — RLE·딕셔너리·비트패킹이 컬럼 데이터를 작게 만드는 원리
3. Predicate Pushdown — 통계(min/max)로 안 읽어도 되는 데이터를 건너뛰기
4. Lakehouse 포맷 — Iceberg/Delta가 파일 위에 ACID·스키마 진화·시간여행을 얹는 방식

**타겟 독자**:
- Parquet를 쓰지만 왜 빠른지 모르는 데이터 개발자
- 컬럼 저장과 행 저장(database-internals)의 차이를 모르는 개발자
- Predicate Pushdown·파티션 프루닝을 단어로만 아는 개발자
- Iceberg/Delta를 "그냥 테이블"로 아는 개발자
- `database-internals`(행 저장)·`spark`를 저장 포맷으로 연결하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `database-internals`(행 저장·B-Tree·페이지와 대조), `computer-architecture-deep-dive`(캐시·SIMD가 컬럼 처리에 유리).
**🤝 시너지**: `spark-internals-deep-dive`(Parquet 읽기·pushdown), `stream-processing-deep-dive`(싱크), `distributed-systems-theory`(Lakehouse 일관성).
**🧬 수렴**: 데이터 레이어의 저장 토대. `elasticsearch`(역색인)·`database`(B-Tree)와 함께 "저장 구조 3종".

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 행 저장 vs 열 저장 (5개 문서)
- 저장 레이아웃 — 같은 테이블을 행으로 vs 열로 저장하는 차이
- 왜 열 저장인가 — 분석은 일부 컬럼·전체 행 집계, 행 저장의 낭비
- OLTP vs OLAP — 트랜잭션(행·database-internals) vs 분석(열), 워크로드 차이
- 캐시·SIMD 친화 — 같은 타입 연속 배치가 벡터 연산에 유리(computer-architecture 연결)
- 트레이드오프 — 단건 조회·쓰기는 행이 유리, 분석은 열이 유리

### Chapter 2: Parquet 구조 (6개 문서)
- 파일 구조 — Row Group·Column Chunk·Page 계층
- 스키마와 중첩 — Dremel 모델, repetition/definition level로 중첩 표현
- 메타데이터 — footer의 스키마·통계·오프셋, 한 번에 읽기
- Page — 데이터 페이지·딕셔너리 페이지, 인코딩 단위
- Row Group 크기 — 병렬성·메모리 트레이드오프
- 읽기 경로 — footer→필요 컬럼 청크만→페이지 디코딩

### Chapter 3: 인코딩과 압축 (6개 문서)
- 딕셔너리 인코딩 — 반복 값을 사전+인덱스로, 카디널리티 효과
- RLE/비트패킹 — 반복·작은 정수를 압축, 정렬 데이터 효과
- 델타 인코딩 — 증가하는 값(타임스탬프·ID)의 차이만 저장
- 압축 코덱 — Snappy/Zstd/Gzip, 압축률 vs CPU 트레이드오프
- 인코딩 + 압축 — 인코딩 후 압축의 시너지, 컬럼별 최적 선택
- 측정 — 같은 데이터를 행(CSV)·열(Parquet)로 크기·읽기 비교

### Chapter 4: 통계와 Predicate Pushdown (5개 문서)
- 컬럼 통계 — min/max/null count, Row Group·Page 수준
- Predicate Pushdown — 필터를 스캔으로 내려 Row Group 스킵
- Page Index — 페이지 수준 통계로 더 정밀한 스킵
- 파티션 프루닝 — 디렉토리 파티션으로 파일 자체 스킵(spark 연결)
- Bloom Filter — 동등 필터의 빠른 스킵, 컬럼 인덱스

### Chapter 5: ORC와 포맷 비교 (4개 문서)
- ORC 구조 — Stripe·인덱스, Parquet와의 차이
- Parquet vs ORC — 생태계·인코딩·통계 차이, 선택 기준
- Arrow — 인메모리 컬럼 포맷, 파일 포맷과의 관계, zero-copy
- 포맷 선택 — 엔진·생태계·압축 기준

### Chapter 6: Lakehouse 테이블 포맷 (6개 문서)
- 파일만으로 부족한 점 — 원자적 변경·동시 쓰기·스키마 진화의 부재
- Iceberg — 메타데이터 트리·스냅샷, 매니페스트
- Delta Lake — 트랜잭션 로그(_delta_log), ACID(distributed 연결)
- 시간 여행 — 스냅샷으로 과거 버전 조회, 버전 관리
- 스키마 진화 — 컬럼 추가/변경을 안전하게, 호환성
- 동시성·ACID — 낙관적 동시성, 충돌 해결(distributed 연결)

### Chapter 7: 성능과 실전 (4개 문서)
- 파일 크기 문제 — 작은 파일 폭발, compaction
- 레이아웃 최적화 — 정렬·파티셔닝·Z-ordering으로 스킵 극대화
- 측정 — pushdown·프루닝 효과를 스캔 바이트로 정량화
- 종합 — 데이터셋을 Parquet+Iceberg로 설계, 쿼리 스캔량 before/after

→ **총 34개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 파일 구조·인코딩·통계, `💻 실전 실험`은 parquet-tools·pyarrow·스캔량 측정. `📊`는 행 vs 열·인코딩·pushdown의 크기/스캔 비교.

## 🎨 스타일 가이드

1. **행 저장과 항상 대조** — database-internals의 B-Tree·페이지와 비교
2. **바이트를 까본다** — parquet-tools로 Row Group·인코딩·통계 *직접 본다*
3. **스킵을 증명** — pushdown 전후 읽은 바이트 수로
4. **database/spark 레포로 착지** — 행 저장은 database, 읽기는 spark
5. 파일 구조·pushdown은 다이어그램으로

## 🔬 검증 환경

> docker 가벼움(python). parquet-tools·pyarrow로 까보기.

```dockerfile
FROM python:3.12-slim
RUN pip install pyarrow pandas duckdb
RUN pip install parquet-tools
```

```bash
# 검증 방법
# 1) 같은 데이터를 CSV vs Parquet → 크기 비교
python -c "import pandas as pd; df=pd.read_csv('data.csv'); df.to_parquet('data.parquet')"
ls -la data.csv data.parquet     # 압축 효과

# 2) Parquet 내부 까보기
parquet-tools inspect data.parquet      # Row Group·인코딩·통계
parquet-tools meta data.parquet         # min/max 통계 확인

# 3) Predicate Pushdown(DuckDB):
duckdb -c "SELECT * FROM 'data.parquet' WHERE id > 1000"
#   EXPLAIN으로 읽은 Row Group·바이트 확인 → 통계로 스킵 검증

# 4) 인코딩 실험: 정렬 데이터 vs 무작위 → RLE/딕셔너리 효과 크기 비교

# 5) Iceberg/Delta: pyiceberg/delta-rs로 스냅샷·시간여행·스키마 진화
#    트랜잭션 로그(_delta_log) 열어 변경 추적 확인
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 📊 Data Engineering 톤, "Parquet로 저장한다 vs 왜 빠른지 안다" 포지셔닝, `🔗 레포 연결`(database-internals·spark·distributed)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- Parquet 공식 — https://parquet.apache.org/docs/
- "Dremel: Interactive Analysis of Web-Scale Datasets" 논문(중첩 인코딩)
- Apache Arrow 문서 — https://arrow.apache.org/
- Iceberg·Delta Lake 공식 문서
- "The Design and Implementation of Modern Column-Oriented Database Systems" 논문

## 💡 핵심 분석 대상

```
행 vs 열 (분석 쿼리):
  테이블: id, name, age, ... (20컬럼)
  쿼리: SELECT avg(age)
  행 저장: 모든 행의 20컬럼을 다 읽음(age만 필요한데)
  열 저장: age 컬럼만 읽음 → I/O 1/20
  + age가 연속 배치 → SIMD 벡터 합 → CPU도 빠름

인코딩 (작게):
  age 컬럼: [25,25,25,30,30,...]
  RLE: (25,3)(30,2)... → 반복 압축
  딕셔너리: name=["kim","lee","kim"] → 사전{0:kim,1:lee} + [0,1,0]
  델타: timestamp 증가 → 차이만 [+1,+1,+1]

Predicate Pushdown (안 읽기):
  쿼리: WHERE age > 100
  Row Group 통계: max(age)=50 → 이 Row Group 전체 스킵(안 읽음!)
  → 통계만 보고 데이터 스킵 → 스캔 바이트 급감

Lakehouse (파일 + ACID):
  Parquet 파일들 + 트랜잭션 로그(Delta) / 메타데이터 트리(Iceberg)
  → 원자적 커밋·동시 쓰기·시간 여행·스키마 진화
  → "파일에 DB의 ACID를 얹기"(distributed 이론)

저장 구조 3종 (수렴):
  행 저장(B-Tree)  : OLTP 단건 조회 (database-internals)
  열 저장(Parquet) : OLAP 분석 (이 레포)
  역색인(Lucene)   : 전문 검색 (elasticsearch)
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 34개 확인 + parquet-tools/DuckDB 검증 환경 + database-internals/spark/distributed 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
