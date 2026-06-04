# Git In-Depth 레포지토리 제작 프롬프트

나는 "Git In-Depth" 레포지토리를 만들려고 해.
JVM Deep Dive, Spring Core/MVC/Batch Deep Dive를 완성한 경험을 바탕으로, **Git의 객체 모델부터 분산 협업 프로토콜까지 내부 동작을 완전히 파헤치는** 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "`.git` 디렉토리 안에서 무슨 일이 벌어지고 있는가 — Plumbing부터 Porcelain까지"

**핵심 차별화**:
1. "어떤 명령어를 쓰나"가 아닌 "Git이 객체를 어떻게 다루나" (Plumbing 중심)
2. `.git/` 디렉토리 직접 해부 — `cat-file`, `hash-object`, `update-ref` 활용
3. Pack file 포맷, Delta compression, Wire Protocol 같은 저수준 분석
4. 복잡한 시나리오를 객체 그래프 변화로 시각화
5. 명령어 → 내부 동작 매핑 (예: `git commit` → blob/tree/commit 객체 생성 + ref 업데이트)

**타겟 독자**:
- Git을 매일 쓰지만 `.git/objects/`에 뭐가 들어있는지 본 적 없는 개발자
- "Branch는 그저 포인터다"는 들었지만 정확히 어떤 파일에 어떻게 저장되는지 모르는 개발자
- Merge와 Rebase의 차이를 객체 그래프 레벨에서 설명하고 싶은 개발자
- Reflog로 복구가 가능한 진짜 이유가 궁금한 개발자
- 대규모 모노레포에서 partial clone, sparse checkout을 설계해야 하는 개발자
- Git 면접 질문에 internals 레벨로 답하고 싶은 개발자

**기존 레포와의 관계**:
- **JVM Deep Dive**: "런타임 시스템 내부 분석" 같은 깊이 — Git을 하나의 분산 데이터베이스로 보고 분해
- **Linux for Backend**: 파일 시스템, inode, 파이프 등 OS 레벨 지식과 연결 (Git 객체 저장 방식)
- **Architecture Patterns**: Content-Addressable Storage, Merkle Tree 같은 패턴 연결

**기존 Git In-Depth 레포와의 차이**:
- 명령어 사용법 위주 ❌ → Plumbing 명령어로 internals 분석 ✅
- 18개 카테고리 평면 구조 ❌ → 16개 챕터로 점진적 깊이 ✅
- "이렇게 하세요" 가이드 ❌ → "왜 이렇게 동작하는가" 원리 분석 ✅
- 60개 토픽 ❌ → 90개 문서, 각 문서 깊이 강화 ✅

---

## 🎯 1단계: 전체 구조 설계

다음 16개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Git Object Model — Content-Addressable Storage (8개 문서)
- `.git/` 디렉토리 완전 해부 (`HEAD`, `refs/`, `objects/`, `index`, `config`, `hooks/`)
- 4가지 객체 (Blob, Tree, Commit, Tag) 구조와 직렬화 포맷
- SHA-1 해시 생성 알고리즘 (`hash-object` 분해, 헤더 + 본문)
- Content-Addressable Storage 패턴 (Git이 사실은 분산 KV Store인 이유)
- Loose Object vs Pack File (zlib 압축, fanout)
- Delta Compression 알고리즘 (xdelta, base object 선택)
- Reachability와 Garbage Collection (`git gc`, `git fsck`)
- Merkle Tree 구조: Git이 무결성을 보장하는 방식

### Chapter 2: References & HEAD (5개 문서)
- `refs/heads/`, `refs/tags/`, `refs/remotes/` 디렉토리 구조
- HEAD의 본질: Symbolic Reference vs Direct Reference
- `packed-refs` 파일과 ref 압축 (`git pack-refs`)
- 특수 참조 (`ORIG_HEAD`, `FETCH_HEAD`, `MERGE_HEAD`, `CHERRY_PICK_HEAD`)의 역할
- Detached HEAD 상태 분석: 위험성과 활용

### Chapter 3: Index와 Working Directory (6개 문서)
- `.git/index` 바이너리 파일 포맷 (`git ls-files --stage`)
- 3 Tree 모델: HEAD Tree vs Index vs Working Directory
- `git status`가 내부적으로 비교하는 3가지 (HEAD↔Index, Index↔Working, untracked)
- `git add`의 본질: blob 객체 생성 + index 업데이트 (`hash-object` + `update-index`)
- `assume-unchanged` vs `skip-worktree`: 언제 어떤 걸 쓰나
- `.gitignore` 매칭 알고리즘과 패턴 우선순위 (`git check-ignore -v`)

### Chapter 4: Commit & DAG History (5개 문서)
- Commit 객체 내부 (tree 참조, parent(s), author, committer, 메시지)
- DAG (Directed Acyclic Graph) 구조: 부모 관계로 이루어진 그래프
- Linear vs Non-linear History (Merge commit이 만드는 분기)
- Reachability 분석 (`git rev-list`, ancestor 탐색 알고리즘)
- Commit-Graph 파일 (Git 2.18+): 대규모 레포에서 history 탐색 가속

### Chapter 5: Branch 메커니즘 (5개 문서)
- Branch는 그저 40바이트 텍스트 파일 (`.git/refs/heads/<name>`)
- `git branch <name>`이 내부적으로 하는 일 (`update-ref refs/heads/<name>`)
- Branch 이동: `git switch`/`checkout`이 HEAD 심볼릭 ref를 어떻게 바꾸나
- Tracking Branch 메커니즘 (`branch.<name>.remote`, `branch.<name>.merge`)
- Branch 명명 규칙과 충돌 (`feature/foo` vs `feature/foo/bar` 충돌 이유)

### Chapter 6: Merge Internals (8개 문서)
- 3-way Merge 알고리즘: base, ours, theirs를 어떻게 결합하는가
- Common Ancestor 찾기: LCA(Lowest Common Ancestor) 알고리즘
- Fast-forward의 본질: 새 커밋 없이 ref만 이동
- Merge Strategies 비교 (`resolve`, `recursive`, `ort`, `octopus`, `subtree`)
- `recursive` → `ort` 전환 (Git 2.34+): 왜 더 빨라졌는가
- Conflict Detection 알고리즘과 충돌 마커 생성
- Rerere (REuse REcorded REsolution): 충돌 자동 해결 메커니즘
- Merge Driver 커스터마이징 (`.gitattributes` + `merge` driver)

### Chapter 7: Rebase Internals (6개 문서)
- Rebase가 새 커밋을 만드는 이유 (커밋은 immutable)
- Patch 적용 방식 vs Merge 방식
- Interactive Rebase 내부 (`.git/rebase-merge/git-rebase-todo` 파일 구조)
- `pick`, `reword`, `edit`, `squash`, `fixup`, `drop` 각각의 객체 변화
- `git rebase --onto <newbase> <upstream> <branch>` 분해
- Rebase 충돌 해결 메커니즘과 `--continue`/`--skip`/`--abort` 동작

### Chapter 8: Reset, Restore, Revert 분해 (5개 문서)
- `git reset` 3가지 모드 (`--soft`, `--mixed`, `--hard`)가 바꾸는 영역
- `git restore`가 새로 도입된 이유 (Reset의 모호함 해결)
- `git revert`의 본질: 역방향 패치를 새 커밋으로 추가
- Merge commit revert (`-m 1` 옵션이 필요한 이유)
- Reset이 위험한 진짜 이유: reflog 만료와 GC 시점

### Chapter 9: Stash & Cherry-pick (5개 문서)
- Stash의 본질: 특수한 merge commit (parent 2~3개)
- `.git/refs/stash`와 reflog로 관리되는 stash 스택
- Stash 충돌 해결 메커니즘
- Cherry-pick 알고리즘: 부모와의 diff를 새 base에 적용
- Cherry-pick vs Rebase: 본질적으로 같은 작업의 다른 표현

### Chapter 10: Refspec & Remote Protocol (6개 문서)
- Refspec 문법 완전 분석 (`+refs/heads/*:refs/remotes/origin/*`)
- Smart Protocol vs Dumb Protocol (HTTP, SSH, Git Protocol)
- Push 메커니즘 (Capability negotiation → Pack 전송 → Ref 업데이트)
- Fetch 메커니즘과 Pack Negotiation
- `--force` vs `--force-with-lease` vs `--force-if-includes` 차이
- Atomic Push와 다중 ref 업데이트

### Chapter 11: Reflog & 복구 메커니즘 (5개 문서)
- `.git/logs/HEAD`와 `.git/logs/refs/heads/<branch>` 파일 구조
- Reflog 항목 포맷과 만료 정책 (`gc.reflogExpire`)
- `git fsck --lost-found`로 unreachable 객체 찾기
- 잃어버린 커밋/브랜치 복구 시나리오 (force push, rebase 실수, hard reset)
- GC와 reflog 만료의 관계 (왜 30일/90일 안에 복구해야 하나)

### Chapter 12: Hooks & 자동화 (4개 문서)
- Client-side Hooks 13종 실행 시점과 환경변수
- Server-side Hooks (`pre-receive`, `update`, `post-receive`)와 표준 입력
- Hook으로 만드는 자동화 (commit-msg 검증, pre-push 테스트)
- Husky / lint-staged 동작 원리 (Node.js 기반 hook 관리)

### Chapter 13: Submodule, Subtree, Monorepo (5개 문서)
- Submodule이 별도 저장소인 이유 (gitlink 객체)
- `.gitmodules`와 `.git/modules/<name>/` 구조
- Subtree merge strategy: 별도 history를 하나의 트리로 통합
- Submodule vs Subtree 트레이드오프
- `git filter-repo`로 모노레포 마이그레이션 (filter-branch 대체)

### Chapter 14: 대용량 처리 (LFS, Pack, Partial Clone) (5개 문서)
- Pack File 포맷 분석 (header, object entries, delta chains, idx)
- Git LFS 구조: Pointer 객체 + 외부 스토리지 + Smudge/Clean Filter
- LFS Protocol (Batch API, transfer agents)
- Partial Clone (`--filter=blob:none`) 내부 동작 (Git 2.19+)
- Sparse Checkout과 Sparse Index (Git 2.32+)

### Chapter 15: 워크플로우 전략 (5개 문서)
- Centralized vs Feature Branch vs Forking 워크플로우 선택 기준
- Git Flow vs GitHub Flow vs Trunk-Based Development 비교
- Release Train 모델과 멀티 버전 유지 (Backport 전략)
- Stacked PR / Stacked Diff 워크플로우 (Graphite, Sapling 영감)
- 회사 규모/문화별 워크플로우 설계 (스타트업 / 중견 / 빅테크 / OSS)

### Chapter 16: 트러블슈팅 시나리오 (7개 문서)
- Push Rejected 모든 경우 (non-fast-forward, hook reject, protected branch)
- 손상된 `.git` 복구 (`git fsck` 활용, dangling object 복구)
- 대용량 파일 잘못 커밋 → 히스토리에서 제거 (`git filter-repo`)
- 히스토리 재작성 후 협업 (force-with-lease, 팀 통보 프로토콜)
- Detached HEAD 안전 탈출 시나리오
- Rebase 도중 50개 연속 충돌 지옥 탈출 (`rerere`, 분할 rebase)
- 잘못 merge한 거대 PR 되돌리기 (revert merge → re-merge 함정)

---

각 챕터는 **4~8개 문서**로 구성해줘. 총 **90개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 메커니즘이 필요한가
## 😱 흔한 오해 또는 실수 (Before)
## ✨ 올바른 이해와 사용 (After)
## 🔬 내부 동작 원리 (`.git/` 직접 해부 + Plumbing 명령어)
## 💻 실험으로 확인하기 (실행 가능한 시나리오)
## 📊 객체 그래프 시각화 (Before → After)
## 🤔 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. Plumbing 명령어 적극 활용 (`cat-file`, `hash-object`, `update-ref`, `ls-files`, `rev-list`, `update-index`, `write-tree`, `commit-tree`)
2. `.git/` 디렉토리 직접 열어서 분석 (loose object 풀어보기, refs 파일 읽기)
3. 객체 그래프 시각화 (mermaid `gitGraph` 또는 `git log --graph --oneline --all`)
4. "명령어가 객체 그래프를 어떻게 변화시키는가" 항상 표시
5. 실수 시나리오 직접 만들고 reflog로 복구 실습
6. Git 소스코드 일부 추적 (가능한 경우, 예: merge 알고리즘)

**실험 환경**:
- Git 2.40+ (최신 기능 활용 — sparse index, ort merge 등)
- Bash + tree (디렉토리 탐색)
- xxd / hexdump (.git/index 바이너리 분석)
- mermaid `gitGraph` (객체 그래프 시각화)
- 테스트 레포지토리 빌드 스크립트 제공

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성 (`01-object-model`, `02-references`, ..., `16-troubleshooting`)
2. **README.md 작성**: 
   - 기존 Git In-Depth README의 비주얼 톤 유지 (배지, 테이블, details 토글)
   - 18개 카테고리 → 16개 챕터로 재구성
   - 난이도 체계 (⭐, ⭐⭐, ⭐⭐⭐) 유지
   - 학습 경로 (입문 4주 / 실무자 4주 / 면접 준비 / 고급자 / 빠른 복습) 유지하되 챕터 매핑 갱신
   - Plumbing 명령어 강조 추가
3. **챕터별 문서 작성**: 
   - Chapter 1부터 순서대로
   - 한 챕터씩 완성 후 다음으로
   - 각 문서는 2500~3500 단어 분량
   - Chapter 1 (Object Model)이 모든 후속 챕터의 기반이므로 가장 공들여 작성

---

## 📚 참고 자료

<JVM Deep Dive README를 여기에 붙여넣기>

<Spring Core Deep Dive README를 여기에 붙여넣기>

<기존 Git In-Depth README를 여기에 붙여넣기>

위 README들을 참고해서 비슷한 구조와 깊이로 새 버전을 만들어줘.

**비주얼 톤은 기존 Git In-Depth README 유지 (배지, 테이블, details), 깊이는 Spring Core Deep Dive 수준.**

---

## 💡 핵심 분석 대상

### `.git/` 디렉토리 — 모든 것의 시작

```bash
# 기본 구조
.git/
├── HEAD                  # 현재 브랜치 심볼릭 참조
├── config                # 로컬 설정
├── description           # GitWeb용 설명
├── hooks/                # Git Hooks
├── info/
│   └── exclude           # 로컬 .gitignore
├── objects/              # 모든 객체 저장소
│   ├── ab/cdef...        # Loose Object (SHA-1 첫 2자리/나머지)
│   ├── pack/             # Pack files
│   └── info/
├── refs/
│   ├── heads/            # 로컬 브랜치 (각각 40바이트 SHA)
│   ├── tags/
│   └── remotes/
├── packed-refs           # 압축된 ref 모음
├── index                 # Staging Area (바이너리)
└── logs/                 # Reflog
    ├── HEAD
    └── refs/heads/<branch>
```

### Plumbing 명령어로 객체 직접 보기

```bash
# Blob 객체 만들고 보기
$ echo "hello" | git hash-object -w --stdin
ce013625030ba8dba906f756967f9e9ca394464a

$ git cat-file -t ce013625
blob

$ git cat-file -p ce013625
hello

# Loose object 직접 풀어보기 (zlib 압축됨)
$ python3 -c "import zlib; print(zlib.decompress(open('.git/objects/ce/013625030ba8dba906f756967f9e9ca394464a','rb').read()))"
b'blob 6\x00hello\n'

# Tree 객체 보기
$ git cat-file -p HEAD^{tree}
100644 blob ce013625...    hello.txt
040000 tree abcd1234...    src

# Commit 객체 보기
$ git cat-file -p HEAD
tree abcd1234...
parent ef567890...
author IQ <e9ua1@example.com> 1730000000 +0900
committer IQ <e9ua1@example.com> 1730000000 +0900

Initial commit
```

### `git commit`이 내부적으로 하는 일 분해

```bash
# 수동으로 commit 만들어보기 (porcelain 없이!)
$ echo "hello" > hello.txt
$ git hash-object -w hello.txt        # blob 생성
$ git update-index --add --cacheinfo 100644 <blob-sha> hello.txt
$ tree_sha=$(git write-tree)          # index → tree 객체
$ commit_sha=$(echo "first" | git commit-tree $tree_sha)
$ git update-ref refs/heads/main $commit_sha
```

→ `git commit` 한 줄이 사실 4단계 작업이라는 걸 직접 분해

### Merge 알고리즘 핵심 — 3-way merge

```
       A---B---C  (feature)
      /
 ---o---X---Y---Z  (main)

LCA: o (common ancestor)
ours: Z (main의 tip)
theirs: C (feature의 tip)

각 파일에 대해:
  if ours == base && theirs != base: take theirs
  if ours != base && theirs == base: take ours
  if ours == theirs: no conflict
  if ours != base && theirs != base && ours != theirs: CONFLICT
```

→ 이 단순한 알고리즘이 어떻게 복잡한 충돌을 만드는지, recursive/ort가 어떻게 개선했는지 추적

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~8개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (90개 목표)
- 주요 분석 대상 (Plumbing 명령어, `.git/` 파일, Git 소스코드 등)
- 각 챕터에서 다룰 Git 버전 명시 (특히 2.18+ commit-graph, 2.32+ sparse index, 2.34+ ort merge 등 최신 기능)
- 챕터 간 의존성 (Chapter 1 Object Model이 모든 후속의 기반)

설계가 완료되면 내가 검토하고, 승인하면 2단계(디렉토리 생성)로 넘어가자.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
