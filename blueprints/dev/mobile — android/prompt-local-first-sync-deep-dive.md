# Local-first & Sync (CRDT) Deep Dive 레포지토리 제작 프롬프트

나는 "Local-first & Sync (CRDT) Deep Dive" 레포지토리를 만들려고 해.
오프라인에서도 즉시 동작하고 재연결 시 충돌 없이 수렴하는 앱을 어떻게 만드는지 — 로컬 DB(SQLite/Room), 동기화 엔진, 충돌 해결(CRDT)을 완전히 파헤치는 레포야.
"서버에 매번 요청하는 것"과 "로컬을 진실의 원천으로 두고 백그라운드로 수렴시키는 것"의 차이를 만드는 레포다. 분산 이론의 CRDT가 *손에 잡히는 앱*이 되는 곳이다.

## 📋 프로젝트 목표

**컨셉**: "서버 요청-응답으로 앱을 만드는 것과, 로컬 우선 + 최종 수렴으로 오프라인에서도 즉각 반응하는 앱을 만드는 것은 다르다"

**핵심 차별화**:
1. Local-first 철학 — 로컬이 진실의 원천, 네트워크는 동기화 수단, 즉각 응답·오프라인
2. 로컬 저장 엔진 — SQLite/Room 내부, 트랜잭션·인덱스·반응형 쿼리
3. 동기화 엔진 — 변경 추적·전송·병합, 멱등·순서·재시도
4. CRDT 실전 — 분산 이론의 CRDT를 실제 동기화 충돌 해결로(distributed 레포의 착지점)

**타겟 독자**:
- 매 동작마다 서버 요청하고 로딩 스피너를 보여주는 개발자
- 오프라인 지원을 "캐시"로만 생각하는 개발자
- 동기화 충돌을 "마지막이 이김"으로 대충 처리하는 개발자
- CRDT를 이론으로만 배운 개발자(distributed 레포 수강자)
- 모바일·웹 양쪽에서 오프라인·동기화를 다루는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `distributed-systems-theory-deep-dive`(CRDT·일관성·벡터시계), `database-internals`(저장·트랜잭션·MVCC).
**🤝 시너지**: `android-architecture-deep-dive`(데이터 레이어·Repository), `realtime-client-networking-deep-dive`(실시간 동기 전송), `kotlin-deep-dive`(Flow 반응형 쿼리).
**🧬 수렴**: `distributed-systems-theory`의 CRDT가 여기서 *제품*이 됨. 모바일+웹 공통 클라이언트 데이터.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: Local-first 철학 (5개 문서)
- 문제 정의 — 매 동작 서버 왕복의 지연·오프라인 무력, 로딩 지옥
- Local-first 7원칙 — 즉각성·오프라인·다기기·협업·장기보존·프라이버시·소유
- 진실의 원천 이동 — 서버 권위 → 로컬 권위 + 수렴, 멘탈 모델 전환
- 적합/부적합 — 어떤 앱이 local-first에 맞나(노트·할일 vs 은행 잔고)
- 아키텍처 개요 — 로컬 DB + 동기화 엔진 + 충돌 해결의 3층

### Chapter 2: 로컬 저장 엔진 (6개 문서)
- SQLite 내부 — 파일 기반 DB, 페이지·B-Tree·WAL(database-internals 연결)
- Room — SQLite 추상화, 컴파일타임 검증 쿼리, 엔티티·DAO
- 트랜잭션 — 원자성·일관성, 동시 접근, WAL 모드
- 반응형 쿼리 — Flow로 데이터 변경 관찰(kotlin 연결), 자동 UI 갱신
- 인덱스·성능 — 모바일 제약에서의 쿼리 튜닝
- 마이그레이션 — 스키마 변경·버전, 데이터 보존

### Chapter 3: 동기화 엔진 기초 (6개 문서)
- 동기화 모델 — push/pull, 변경 추적(change log), 마지막 동기 시점
- 변경 추적 — 무엇이 변했나 기록, tombstone(삭제 추적)
- 멱등성 — 같은 변경 두 번 적용해도 안전(distributed 연결), 재시도 안전
- 순서와 인과 — 변경 순서 보장, 벡터/논리 시계(distributed 연결)
- 전송 — 배치·델타·압축, 실시간(realtime-networking 연결) vs 주기적
- 부분 동기화 — 큰 데이터셋의 일부만, 페이지네이션·구독 범위

### Chapter 4: 충돌과 해결 전략 (5개 문서)
- 충돌이 생기는 이유 — 다기기 동시 편집, 오프라인 분기
- 단순 전략 — LWW(마지막 쓰기 승), 그 위험(데이터 유실)
- 수동 병합 — 사용자에게 충돌 제시(git 유사), 적합한 경우
- 의미 기반 병합 — 필드별·도메인 규칙 병합
- 전략 선택 — 데이터 성격별(카운터·텍스트·집합) 적합한 해결

### Chapter 5: CRDT 실전 (6개 문서)
- CRDT 복습 — 수렴 조건(교환·결합·멱등), 이론에서 실전으로(distributed 연결)
- 기본 CRDT 구현 — 카운터(PN-Counter)·집합(OR-Set)·레지스터(LWW)
- 텍스트 CRDT — 협업 편집(Yjs/Automerge), 문자 단위 동시성
- 리스트·맵 CRDT — 순서 있는 리스트, 중첩 구조
- 메타데이터 비용 — tombstone 누적·압축(GC), CRDT의 현실적 한계
- 라이브러리 — Yjs/Automerge로 실제 동기화 구축

### Chapter 6: 플랫폼 통합 (5개 문서)
- 안드로이드 — Room + 동기화 + WorkManager(백그라운드 동기)
- 웹/멀티플랫폼 — IndexedDB(web-apis 연결)·SQLite WASM, 같은 동기 로직 공유
- 백엔드 — 동기 서버, 변경 브로드캐스트(realtime-networking·kafka 연결)
- 인증·권한 — 동기 데이터의 접근 제어, 멀티테넌시
- 기존 동기 솔루션 — PowerSync·ElectricSQL·Replicache 비교

### Chapter 7: 운영과 측정 (4개 문서)
- 동기 디버깅 — 충돌·수렴 실패 추적, 상태 일관성 검증
- 성능 — 동기 지연·배터리·저장 비용, 모바일 제약
- 데이터 무결성 — 수렴 검증, 분할 후 재연결 테스트(distributed 연결)
- 종합 — 오프라인 협업 할일 앱을 Room + CRDT + 동기 서버로 끝까지 구현

→ **총 35개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 동기 알고리즘·CRDT 수렴, `💻 실전 실험`은 다기기/오프라인 시뮬레이션 + 분할 후 수렴 검증. `📊`는 동기 전략별 충돌·지연·저장 비교.

## 🎨 스타일 가이드

1. **distributed 이론의 착지** — CRDT를 이론이 아닌 *동작하는 앱*으로
2. **오프라인을 직접 시뮬** — 네트워크 끊고 양쪽 편집→재연결→수렴 *눈으로*
3. **LWW의 위험을 보여준다** — 단순 전략이 데이터를 잃는 사례
4. **모바일+웹 공유** — 같은 동기 로직이 Room·IndexedDB에서
5. 동기 흐름·CRDT 병합은 다이어그램으로

## 🔬 검증 환경

> docker(동기 서버) + 안드로이드/브라우저 클라이언트. CRDT 수렴은 코드로 검증.

```yaml
# docker-compose.yml — 동기 서버 + DB
services:
  sync-server:
    build: ./sync-server     # 변경 수신·브로드캐스트(WebSocket)
    ports: ["8080:8080"]
  postgres:
    image: postgres:16       # 서버측 영속
    ports: ["5432:5432"]
```

```bash
# 검증 방법
# 1) 오프라인 시뮬: 두 클라이언트(에뮬2개/탭2개)
#    네트워크 끊기 → 각자 편집 → 재연결 → 수렴 확인
# 2) CRDT 수렴 단위 테스트:
#    두 복제본에 다른 순서로 연산 적용 → 결과 동일(수렴) 검증
#    assert(replicaA.value == replicaB.value)
# 3) LWW 위험 재현: 동시 편집 → 한쪽 유실 관찰
# 4) Room 반응형 쿼리: Flow로 DB 변경→UI 자동 갱신
# 5) tombstone 증가 관찰 → 압축 전후 저장 비교
# 6) Yjs/Automerge 데모로 텍스트 협업 수렴
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `demo/`(오프라인 협업 앱)
2. **README.md**: 🤖 Android(+웹) 톤, "서버 왕복 vs 로컬 우선+수렴" 포지셔닝, `🔗 레포 연결`(distributed·database-internals·realtime)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- "Local-first software" (Ink & Switch) — https://www.inkandswitch.com/local-first/
- Martin Kleppmann CRDT 강의·Automerge — https://automerge.org/
- Yjs 문서 — https://docs.yjs.dev/
- Room 문서 — https://developer.android.com/training/data-storage/room
- ElectricSQL·PowerSync·Replicache 문서

## 💡 핵심 분석 대상

```
Local-first 흐름:
  사용자 동작 → 로컬 DB 즉시 반영 → UI 즉각 갱신(0ms 체감)
                    │ (백그라운드)
                    ▼ 동기 엔진 → 서버 → 다른 기기
  ↔ 기존: 동작 → 서버 요청 → 로딩 → 응답 → 갱신(지연·오프라인 무력)

오프라인 분기 → 수렴:
  기기A(오프라인): 할일 추가 X
  기기B(오프라인): 할일 추가 Y, 할일 Z 완료
  재연결 → 변경 교환 → CRDT 병합
  → 양쪽 모두 {X, Y, Z완료} 로 수렴(유실 0)

LWW vs CRDT:
  LWW: 동시 편집 시 타임스탬프 늦은 것만 남음 → 다른 편집 유실
  CRDT: 둘 다 보존하도록 자료구조가 보장 → 유실 0

CRDT 수렴 조건:
  교환법칙: a·b = b·a (순서 무관)
  결합법칙: (a·b)·c = a·(b·c)
  멱등성:   a·a = a (중복 무관)
  → 어떤 순서·중복으로 와도 같은 결과 (distributed 이론)

OR-Set (추가-삭제 충돌):
  각 원소에 고유 태그 → 추가/삭제를 태그로 추적
  동시 추가+삭제 → 추가가 이김(태그가 다름) → 의도 보존
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 35개 확인 + 동기서버/오프라인시뮬/CRDT수렴테스트 검증 환경 + distributed/database-internals/realtime 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
