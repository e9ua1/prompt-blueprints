# Reactivity & State Compared 레포지토리 제작 프롬프트

나는 "Reactivity & State Compared" 레포지토리를 만들려고 해.
이건 이 연구소의 **가장 상징적인 횡단 비교(Synthesis)** 레포야. "상태가 변하면 의존하는 것만 정확히 갱신한다"는 한 아이디어가 프론트·모바일·DB에서 *놀랍도록 같은 모양*으로 나타남을 보여준다 — Signal, Compose Snapshot State, SwiftUI AttributeGraph, React, 그리고 그 동형(isomorphism)인 **InnoDB MVCC**까지.
"한 프레임워크의 반응성을 아는 것"과 "반응성·상태 일관성이 UI든 DB든 같은 문제의 같은 해법임을 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "한 프레임워크의 상태 관리를 아는 것과, '의존성 추적 + 일관된 스냅샷'이 UI 반응성과 DB 동시성 제어에서 같은 구조임을 아는 것은 다르다"

**핵심 차별화**:
1. 반응성의 공통 구조 — "읽기 추적 → 변경 감지 → 의존자만 갱신"이 모든 반응성의 뼈대
2. 의존성 추적의 스펙트럼 — 수동(Redux)·세밀(Signal)·그래프(AttributeGraph)·위치(Compose)
3. **스냅샷 동형성** — Compose Snapshot State ↔ SwiftUI AttributeGraph ↔ **InnoDB MVCC**가 *같은 문제*(일관된 읽기 + 동시 변경)를 푼다
4. 일관성은 어디에나 — UI의 "tearing 없는 렌더"와 DB의 "일관된 읽기"가 같은 개념

**타겟 독자**:
- 한 상태 라이브러리는 알지만 패러다임 간 비교를 못하는 개발자
- Signal·Snapshot·AttributeGraph가 같은 거라는 걸 모르는 개발자
- UI 상태와 DB 동시성 제어가 연결된다는 걸 상상 못한 개발자
- 반응성의 본질을 한 번에 잡고 싶은 개발자
- 프론트·모바일·DB 레포를 마치고 *가장 깊은 연결*을 보려는 개발자

## 🔗 레포 연결

**⬆️ 선행(이 비교의 입력)**:
`frontend-state-management-deep-dive`(Signal·Redux·Proxy), `react-internals-deep-dive`(리렌더), `jetpack-compose-internals-deep-dive`(Snapshot State), `swiftui-internals-deep-dive`(AttributeGraph), `database-internals`(InnoDB MVCC), `distributed-systems-theory-deep-dive`(일관성 모델).
**🤝 시너지**: `javascript-deep-dive`(Proxy·클로저), `postgresql-deep-dive`(MVCC 변형).
**🧬 본질**: 이 레포가 연구소의 *지적 정점* — UI 반응성과 DB 동시성을 하나의 개념으로 묶는다.

---

## 🎯 1단계: 전체 구조 설계

> "기술별"이 아니라 **"개념별"**로 구성, 각 개념에서 UI~DB를 나란히.

### Chapter 1: 반응성의 근본 구조 (5개 문서)
- 공통 질문 — "상태가 변하면 누구를 갱신해야 하나"를 어떻게 아나
- 세 단계 뼈대 — 읽기 추적 → 변경 감지 → 의존자 알림(모든 반응성 공통)
- 수동 vs 자동 — 명시적 통지(Redux/setState) vs 자동 추적(Signal/Proxy)
- 추적 정밀도 — 컴포넌트 단위 vs 값 단위, 갱신 범위가 성능을 결정
- 비교 프레임 — 평가 축(추적 방식·정밀도·일관성·동시성)

### Chapter 2: 의존성 추적의 스펙트럼 (6개 문서)
- 명시적 통지 — Redux dispatch·setState, 수동·예측 가능(state-management 연결)
- Proxy 자동 추적 — get/set 트랩으로 접근·변경 감지(MobX/Valtio, javascript 연결)
- Signal 세밀 추적 — 읽기 시 자동 구독, 그 값을 읽은 곳만(state-management 연결)
- 위치 기억 — Compose Slot Table, 호출 위치로 상태 매칭(compose 연결)
- 그래프 추적 — SwiftUI AttributeGraph, 의존성 그래프(swiftui 연결)
- 나란히 — 같은 "파생 값 + 의존 갱신"을 5방식으로, 갱신 범위 비교

### Chapter 3: 파생 상태와 일관성 (5개 문서)
- 파생 상태 — computed·derivedStateOf, 의존이 변하면 재계산
- 글리치 — 일관성 없는 중간 상태(다이아몬드 의존), 어떻게 피하나
- 일관된 전파 — 올바른 순서로 갱신, tearing 방지
- UI tearing — 동시 렌더 중 다른 상태를 보는 문제(react useSyncExternalStore 연결)
- DB와의 연결 — UI tearing ↔ DB의 일관된 읽기(다음 챕터로)

### Chapter 4: 스냅샷 — UI와 DB의 동형성 ★핵심 (7개 문서)
- 스냅샷 아이디어 — "변경 중에도 일관된 한 버전을 본다"
- Compose Snapshot State — 여러 스냅샷 동시 존재, 격리된 변경·apply(compose 연결)
- SwiftUI AttributeGraph — 일관된 갱신 단위(swiftui 연결)
- **InnoDB MVCC** — Undo Log로 버전 관리, 트랜잭션이 일관된 스냅샷 읽기(database 연결)
- **동형성 증명** — Compose Snapshot과 MVCC가 *같은 문제*(동시 변경 + 일관 읽기)를 *같은 방식*(다중 버전)으로 푼다
- 충돌 해결 — Snapshot apply 충돌 ↔ MVCC 쓰기 충돌, 같은 구조
- 나란히 — UI 스냅샷과 DB MVCC를 같은 다이어그램으로(이 레포의 하이라이트)

### Chapter 5: 일관성 모델 — UI에서 분산까지 (5개 문서)
- 일관성 스펙트럼 — 강한 일관성 ~ 최종 일관성(distributed 연결)
- UI의 일관성 — 단일 진실 원천, tearing 없는 렌더
- 낙관적 업데이트 — UI 즉시 반영 후 확정, DB 낙관적 동시성과 동형
- 최종 일관성 UI — 로컬 우선·나중 수렴(local-first·CRDT 연결)
- 나란히 — UI 상태 일관성 ↔ DB 격리 수준 ↔ 분산 일관성의 공통 축

### Chapter 6: 같은 개념, 여러 구현 (6개 문서)
- 과제 정의 — 동일한 반응성 과제(파생 상태 + 동시 변경 + 일관 읽기)
- 구현 ① Signal(웹) / ② Compose Snapshot(Android)
- 구현 ③ SwiftUI AttributeGraph / ④ React + 외부 스토어
- 구현 ⑤ DB 트랜잭션(MVCC)으로 같은 일관성 문제
- 코드/구조 비교 — 5구현의 추적·일관성 메커니즘 나란히
- 미니 반응성 구현 — 직접 만든 Signal에 스냅샷 일관성 추가(원리 통합)

### Chapter 7: 종합 — 하나의 개념 (4개 문서)
- 대통일 — "의존성 추적 + 일관된 스냅샷"이 UI·모바일·DB·분산을 관통
- 트레이드오프 — 추적 정밀도·일관성 강도·구현 복잡도 표
- 왜 같은 모양인가 — 같은 본질 문제(변경 전파 + 일관성)이기 때문
- 종합 — 반응성·일관성 지형도, 새 프레임워크/DB를 만나도 이 프레임으로

→ **총 38개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). 비교가 핵심이라 `🔬`·`📊`에서 *항상 UI~DB를 나란히*. `💻 실전 실험`은 같은 일관성 과제를 UI와 DB로.

## 🎨 스타일 가이드

1. **항상 나란히, UI~DB 가로질러** — Signal·Snapshot·AttributeGraph·MVCC를 같은 표에
2. **동형성을 증명** — Compose Snapshot ↔ InnoDB MVCC가 *같은 문제·같은 해법*임을 단계적으로
3. **선행 레포로 깊이 위임** — 각 메커니즘 내부는 해당 레포로, 여기선 *연결*
4. **"같은 본질"로 환원** — 모든 걸 "변경 전파 + 일관성"으로
5. 추적 스펙트럼·스냅샷 동형성은 다이어그램으로(특히 Ch4)

## 🔬 검증 환경

> UI 멀티 프레임워크 + DB. 같은 일관성 과제를 양쪽에서.

```bash
# UI 반응성 (웹/모바일)
npx create-vite reactivity-lab --template react-ts
npm i solid-js valtio        # Signal·Proxy 비교
# Compose/SwiftUI는 별도 미니 예제

# DB MVCC
docker run -d -p 3306:3306 -e MYSQL_ROOT_PASSWORD=root mysql:8.0

# 검증 방법
# 1) 의존성 추적 정밀도: 같은 파생 상태를 Signal vs Redux
#    → DevTools로 갱신 범위(리렌더 횟수) 비교

# 2) 스냅샷 동형성 (이 레포의 하이라이트):
#    Compose: 두 스냅샷에서 같은 State를 다르게 변경 → apply → 충돌
#    MySQL:   두 트랜잭션이 같은 행을 다르게 변경 (REPEATABLE READ)
#      SELECT ... (일관된 스냅샷 읽기) — Undo Log 버전
#      → 같은 "동시 변경 + 일관 읽기" 문제를 같은 다중 버전으로 푼다
#    → 두 코드를 나란히 놓고 동형 확인

# 3) tearing: 외부 스토어 동시 변경 중 React 렌더 일관성
#    useSyncExternalStore 유무로 tearing 재현/해결

# 4) 낙관적 업데이트: UI 즉시 반영 vs DB 낙관적 락 — 같은 패턴 확인

# 5) 미니 구현: signal()에 버전 스냅샷 추가 → 일관된 읽기 구현
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `comparisons/`(UI·DB 동일 과제)
2. **README.md**: 🧬 Synthesis 톤, "한 프레임워크 반응성 vs UI·DB를 관통하는 한 개념" 포지셔닝, 선행 6개 레포를 `🔗 레포 연결` 입력으로. **Compose Snapshot ↔ MVCC 동형성을 대표 문구로**
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어, 항상 UI~DB 가로지르는 비교

## 📚 참고 자료

- 각 선행 레포의 참고자료(이 레포는 그 종합)
- "A Hands-on Introduction to Fine-Grained Reactivity" (Ryan Carniato)
- Jetpack Compose Snapshot 시스템 문서·소스
- *Designing Data-Intensive Applications* (MVCC·일관성)
- "Demystify SwiftUI" (AttributeGraph)

## 💡 핵심 분석 대상

```
반응성 공통 뼈대 (UI 전부):
  ① 읽기 추적: 누가 어떤 상태를 읽었나 기록
  ② 변경 감지: 상태가 바뀜
  ③ 의존자 알림: 그 상태를 읽은 것만 갱신
  → Signal·Compose·SwiftUI·MobX 전부 이 3단계

추적 스펙트럼 (정밀도):
  Redux/setState : 수동 통지, 컴포넌트 단위(넓음)
  Proxy(MobX)    : 자동, 접근 기반
  Signal         : 자동, 값 단위(가장 좁음)
  Compose        : 위치 기억(Slot Table)
  SwiftUI        : 의존성 그래프(AttributeGraph)

★ 스냅샷 동형성 (이 레포의 핵심):
  문제: "동시에 변경하면서도 일관된 한 버전을 읽고 싶다"

  Compose Snapshot State:
    여러 스냅샷 동시 존재 → 각자 격리된 변경 → apply 시 충돌 감지
    읽는 쪽은 일관된 스냅샷을 봄

  InnoDB MVCC:
    Undo Log로 행의 여러 버전 보관 → 트랜잭션은 일관된 스냅샷 읽기
    동시 쓰기 → 버전 충돌 감지

  → 둘은 *같은 문제*(동시 변경 + 일관 읽기)를
    *같은 방식*(다중 버전 + 충돌 감지)으로 푼다!
    UI 프레임워크와 데이터베이스가 같은 구조에 도달

일관성 (UI ~ 분산 관통):
  UI tearing 방지 ↔ DB 일관된 읽기 ↔ 분산 일관성
  낙관적 UI 업데이트 ↔ DB 낙관적 동시성 ↔ CRDT 수렴
  → 전부 "변경 전파 + 일관성"의 다른 얼굴
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 38개 확인 + UI멀티+DB 검증 환경 + 선행 6개 레포(state-management·react·compose·swiftui·database-internals·distributed)를 입력으로 하는 연결 명시. **Compose Snapshot ↔ InnoDB MVCC 동형성(Chapter 4)이 이 레포의 하이라이트**이자 연구소 전체의 지적 정점임을 분명히.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
