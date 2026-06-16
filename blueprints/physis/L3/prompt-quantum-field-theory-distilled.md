# Quantum Field Theory Distilled 레포지토리 제작 프롬프트

나는 "Quantum Field Theory Distilled" 레포지토리를 만들려고 해.
파인만 도표로 진폭을 계산하는 것과, **입자가 장(field)의 들뜸**이라는 관점이 왜 근본인지 — 정규모드를 양자화하면 입자가 나온다는 것을 — 유도하는 것은 다르다.
경로적분 공식을 쓰는 것과, 그것이 **최소작용의 양자판**($Z=\int\mathcal D\phi\,e^{iS/\hbar}$)이고 $\hbar\to0$에서 $\delta S=0$이 살아남음을(→ L0) 아는 것은 다르다.
재규격화를 "무한대를 빼는 트릭"으로 여기는 것과, 그것이 *스케일에 따라 이론을 따라가는* 깊은 구조(재규격화군)이고 상전이와 같은 수학임을(→ L2) 아는 것은 다르다.
진공이 비어있지 않다는 것을 — 영점 요동·가상입자·카시미르가 실재임을 — 그리고 QFT가 어떻게 표준모형·우주론으로 이어지는지를 끝까지 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "입자는 장(field)의 들뜸이다 — 경로적분은 최소작용의 양자판, 재규격화는 스케일의 구조, 진공은 비어있지 않다"

**핵심 차별화**:
1. **입자 = 장(field)의 들뜸을 유도** — 정규모드 양자화 → 생성·소멸 → 입자, 동일입자성의 기원(→ L1 oscillations·field-theory)
2. **경로적분 = 최소작용의 양자판** — $Z=\int\mathcal D\phi\,e^{iS/\hbar}$, 고전극한 $\delta S=0$, 파인만 도표의 그림 언어(→ L0 variational)
3. **재규격화를 스케일의 구조로** — "무한대 빼기"가 아니라 달리는 결합·재규격화군, 상전이와 같은 수학(→ L2 phase-transitions), 유효장론(→ L6)
4. **진공은 비어있지 않다** — 영점 요동·가상입자·카시미르·진공 편극, 진공이 매질처럼(→ L2 fluctuations)

**타겟 독자**:
- 파인만 도표는 그리지만 *입자=장의 들뜸*의 의미를 모르는 사람
- 경로적분을 쓰지만 그것이 *최소작용의 양자판*임을(→ L0) 모르는 사람
- 재규격화를 "무한대 빼는 트릭"으로만 보고 *스케일의 구조*임을 모르는 사람
- "진공은 비어있다"고 여기고 영점 요동·가상입자의 실재를 모르는 사람
- QFT가 어떻게 표준모형·곡선시공간·우주론으로 *이어지는지* 보고 싶은 사람

**선행 학습**:
- `classical-field-theory-distilled`(L1) — **필수**. 라그랑지안 밀도·장(field)의 변분
- `oscillations-waves-distilled`(L1) — **필수**. 정규모드(장 양자화의 토대)
- `quantum-mechanics-distilled`(L3) — **필수**. 조화진동자·경로적분 / `variational-principles-distilled`(L0) — 경로적분 / `phase-transitions-distilled`(L2) — 재규격화군

## 🔗 레포 연결

**⬆️ 선행**: `classical-field-theory-distilled`·`oscillations-waves-distilled`(L1), `quantum-mechanics-distilled`(L3).
**🤝 시너지 (L3 내부)**: `standard-model-distilled`(게이지 장론의 본체 — 이 레포의 직접 후속), `quantum-mechanics-distilled`, `emergent-quantum-matter-distilled`(응집물질의 장론).
**🪜 창발 (위 레이어가 다시 쓴다)**:
- `general-relativity-distilled`(L4) — 곡선시공간 QFT, 호킹 복사
- `black-holes-distilled`(L4) — 호킹 복사 = 곡선시공간 진공 요동
- `cosmology-distilled`(L4) — 인플레이션 양자 요동, 우주상수 문제(진공 에너지)
- `quantum-gravity-distilled` / `string-theory-distilled`(L5) — QFT 너머
- `least-action-everywhere` / `emergence-distilled`(L6) — 경로적분·유효장론이 회수되는 곳

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 왜 장(field)인가 — QFT의 필요 (5개 문서)
- **QM의 한계** — 입자 수 변화·상대론·인과율, 왜 QM으론 부족한가
- **상대론 + 양자 = 반물질** — 디랙 방정식, 음에너지·반입자의 필연(→ L4 special relativity)
- **입자 = 장(field)의 들뜸** — 장론의 핵심 관점, 동일입자성의 기원(→ L1 field-theory)
- **장(field)의 양자화** — 정규모드 → 생성·소멸 연산자(→ L1 oscillations 정규모드)
- **진공은 비어있지 않다** — 영점 에너지·진공 요동, 카시미르 예고(→ L2 fluctuations)

### Chapter 2: 정준 양자화 (5개 문서)
- **스칼라장 양자화** — 클라인-고든 장, 모드 전개, 입자 해석(→ L1 field-theory)
- **생성·소멸 연산자** — 포크 공간, 입자 수, 진공 $|0\rangle$
- **전파자(propagator)** — 장의 상관함수, 시공간 두 점 사이 전파
- **스핀-통계 정리** — 보손 vs 페르미온, 정수/반정수 스핀의 기원(→ L2 statistical-mechanics)
- **디랙장** — 페르미온 장, 반교환, 스피너(→ standard-model)

### Chapter 3: 경로적분 (5개 문서)
- **경로적분(QM)** — 모든 경로의 합, 고전극한 $\delta S=0$(→ L0 variational)
- **장(field)의 경로적분** — $Z=\int\mathcal D\phi\,e^{iS/\hbar}$, 생성범함수
- **상관함수와 파인만 도표** — 섭동 전개의 그림 언어, 도표 → 진폭
- **가상입자** — 내부선, "잠깐 빌린 에너지", 실재인가 계산 도구인가(정직한 논의)
- **유클리드 경로적분** — 윅 회전, 통계역학과의 깊은 연결(→ L2)

### Chapter 4: 상호작용과 파인만 도표 (5개 문서)
- **상호작용 항** — $\phi^4$·QED 결합, 섭동론(→ L0 approximation)
- **파인만 규칙** — 도표에서 진폭으로, 꼭짓점·전파자·운동량 보존
- **산란 단면적** — S행렬, 실험과의 연결(→ L1 classical-mechanics 산란)
- **루프와 발산** — 고리 적분의 무한대, 자외선 발산의 등장
- **트리 vs 루프** — 고전 vs 양자 보정, 차수 세기(→ L0 approximation)

### Chapter 5: 재규격화 (6개 문서)
- **발산의 의미** — 무한대는 어디서 오나, 왜 물리량은 유한한가
- **재규격화 절차** — 차원 정규화·절단, 무한대를 물리량으로 흡수(→ L0 approximation)
- **달리는 결합상수** — 에너지에 따라 변하는 "상수", 베타 함수(→ L2 phase-transitions RG)
- **재규격화군(QFT)** — 스케일 변환·고정점, 임계현상과의 통일(→ L2 phase-transitions)
- **유효장론(EFT)** — 모르는 고에너지를 모르는 채로, *왜 작동하나*(→ L0 approximation, → L6 emergence)
- **비섭동 효과** — 인스탄톤·터널링, 섭동 너머의 물리(→ L0 approximation)

### Chapter 6: 게이지장과 진공 (5개 문서)
- **게이지 장론(양자)** — $U(1)$ QED, 게이지 불변의 양자판(→ L0 symmetry, → L1 EM)
- **QED의 정밀 승리** — 이상 자기모멘트 $g-2$, 가장 정밀하게 검증된 예측(→ L0 measurement)
- **진공 편극** — 진공이 매질처럼, 전하 가리기와 달리는 결합
- **진공 구조** — 응축·진공 에너지, 우주상수 문제 예고(→ L4 cosmology)
- **비가환 게이지 예고** — 양-밀스, 게이지장의 자기상호작용(→ standard-model)

### Chapter 7: 종합·계산 (5개 문서)
- **자유장 모드 수치** — 정규모드 양자화, 진공 요동 시각화(Python)
- **파인만 도표 계산** — 간단한 트리 진폭 수치·기호(Python/sympy)
- **달리는 결합 수치** — 베타 함수 적분, 결합상수의 에너지 의존(Python)
- **카시미르 효과 계산** — 진공 에너지 차이와 힘(Python)
- **종합** — "입자는 장의 들뜸, 진공은 비어있지 않다"의 통합(→ standard-model, L4, L6)

→ **총 36개 문서** 목표.

## 📐 문서 구조 — Principle → Boundary → Emergence

Physis 표준 10섹션, 마지막 세 단 종결:

```markdown
## 🎯 핵심 질문      — 어떤 장론 현상을, 어떤 원리에서 유도하나
## 🌍 어디서 나타나나  — 이 장론 구조가 작동하는 장면
## 🔍 직관의 함정     — "가상입자는 실재"·"재규격화는 트릭" 같은 오해
## ⚙️ 제1원리 유도    — 정준 양자화·경로적분·RG에서 결과를 끌어낸다
## 🧪 실험·관측 증거  — g−2·산란 단면적·카시미르의 실측(→ L0 measurement)
## 🔬 경계           — QFT가 깨지는 곳 (중력·플랑크 스케일·강결합)
## 🧬 횡단 원리       — 최소작용(경로적분)·대칭(게이지)·스케일(RG)이 어떻게 반복되나
## 🪜 창발           — 위 레이어(표준모형·곡선시공간·우주)에서 무엇이 나타나나
## 📐 예측·반증       — g−2·달리는 결합 등 검증 가능한 예측
## 🤔 다음 질문       — 다음 문서/레이어로의 연결

---
🧩 **Principle** — (어떤 장론 원리에서 무엇이 나오나)
🔬 **Boundary**  — (QFT가 깨지는 조건: 중력·플랑크 스케일)
🪜 **Emergence** — (위 레이어에서 같은 구조가 낳는 것 + 검증 실험)
```

이 레포에서 **⚙️ 제1원리 유도**는 *정준 양자화 또는 경로적분*에서 끌어내며, **🔬 경계**는 "중력은 재규격화 불가(→ L5)", "플랑크 스케일에서 QFT 붕괴", "강결합 섭동 실패" 같은 한계를 다룬다.

## 🎨 스타일 가이드

1. **입자 = 장(field)의 들뜸을 유도로** — 정규모드 양자화에서 입자가 나옴을 보인다(→ L1)
2. **경로적분 = 최소작용의 양자판** — $\hbar\to0$에서 $\delta S=0$ 회복을 명시(→ L0 variational)
3. **재규격화는 스케일의 구조** — "무한대 빼기" 오해를 교정, 달리는 결합·RG·유효장론으로(→ L2, L6)
4. **진공은 비어있지 않다** — 영점 요동·가상입자·카시미르를 실재(측정됨)로(→ L2 fluctuations)
5. **수치로 부순다** — 정규모드·파인만 진폭·달리는 결합·카시미르를 직접 계산
6. 모드 전개·파인만 도표·RG 흐름·진공 편극은 다이어그램으로

## 🔬 검증 프로토콜

> 정규모드 양자화 + 파인만 진폭 + 달리는 결합(베타 함수) + 카시미르 에너지.

```python
# 환경: Python 3 + numpy + scipy + sympy + matplotlib

# 1) 자유 스칼라장 = 무한개 진동자
import numpy as np
def field_modes(L=10.0, N=64, m=0.3):
    k=2*np.pi*np.fft.fftfreq(N, d=L/N)
    omega=np.sqrt(k**2+m**2)            # 분산관계 ω_k=√(k²+m²)
    return omega    # 각 모드 = 독립 진동자, 영점에너지 Σ ½ℏω_k (→ 카시미르)

# 2) 파인만 트리 진폭 (sympy)
#    φ⁴ 이론의 4점 트리 진폭, 운동량 보존 꼭짓점 → 미분단면적
#    → 도표 규칙을 코드로 검증

# 3) 달리는 결합 (베타 함수 적분)
#    QED: dα/d(lnμ) = α²/(3π) (1루프) → α(μ)가 에너지와 함께 증가
def run_alpha(alpha0, mu0, mu):
    return alpha0/(1 - (alpha0/(3*np.pi))*np.log(mu/mu0))
# → 전하 가리기(진공 편극)의 결과 (→ L2 phase-transitions RG와 같은 구조)

# 4) 카시미르 효과
#    두 판 사이 허용 모드 합(정규화) − 자유공간 → 단위면적당 에너지
#    E/A = −π²ℏc/(720 d³) → 인력, 진공 요동의 측정 가능한 증거
def casimir_energy(d, hbar=1, c=1):
    return -np.pi**2*hbar*c/(720*d**3)

# 5) 유클리드 ↔ 통계역학
#    윅 회전 t→−iτ 로 경로적분이 분배함수가 됨을 수치로 시연
#    → QFT와 통계역학의 통일 (→ L2)

# 6) 영점 에너지·진공 요동
#    조화진동자 바닥상태의 ⟨x²⟩≠0 (영점 운동)을 수치로
#    → "진공은 비어있지 않다" (→ L2 fluctuations)
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md 작성**:
   - 🌌 Physis 톤(atom·인디고 `#6366f1`), "Derive, don't accept", `From the fewest principles, the widest world.`
   - 포지셔닝: "파인만 도표 vs 입자=장의 들뜸·경로적분=최소작용의 양자판 — 진공은 비어있지 않다"
   - **The Knowledge Tetrad** 각주
   - `🔗 레포 연결`: L1 `classical-field-theory`·`oscillations`·L3 `quantum-mechanics`(선행), L3 standard-model·창발물질, L4 GR·블랙홀·우주, L6 `least-action-everywhere`·`emergence`로의 화살 명시
   - **Document Format — Principle → Boundary → Emergence** 안내
3. **챕터별 문서**: Chapter 1부터, 문서당 2500~3500 단어 + 유도 + 코드/그림

## 📚 참고 자료

- *Quantum Field Theory in a Nutshell* (A. Zee) — 경로적분·직관의 명저, 이 레포의 영혼
- *An Introduction to Quantum Field Theory* (Peskin & Schroeder) — 표준 대학원 교재
- *Quantum Field Theory* (Mark Srednicki) — 체계적 표준(무료 공개)
- *Student Friendly Quantum Field Theory* (Klauber) — 친절한 유도
- *The Conceptual Framework of Quantum Field Theory* (Duncan) — 개념적 엄밀함
- *Lectures of Sidney Coleman on Quantum Field Theory* — 전설적 강의록

## 💡 핵심 분석 대상

```
입자 = 장(field)의 들뜸:
  고전 장 = 무한개 정규모드 (→ L1 oscillations)
       │  각 모드를 양자화 (조화진동자 → 사다리 연산자)
       ▼
  생성연산자 a† → 진공에 들뜸 하나 = 입자 한 개
  |0⟩ (진공), a†|0⟩ (입자 1), a†a†|0⟩ (입자 2), ...
  ⇒ "입자"는 장의 들뜸. 동일입자성도 자동 (같은 장의 같은 들뜸)

경로적분 = 최소작용의 양자판:
  Z = ∫ Dφ e^{iS[φ]/ℏ}      (모든 장 배위의 합)
       │  ℏ→0 → 위상 e^{iS/ℏ} 빠르게 진동 → 상쇄
       ▼
  δS=0 (고전 경로)만 살아남음  (→ L0 variational, 정류위상)
  ⇒ 고전역학 = 경로적분의 ℏ→0 극한

재규격화 = 스케일의 구조 (트릭 아님):
  루프 적분 → 자외선 발산 (∞)
       │  절단 Λ 또는 차원 정규화
       ▼
  물리량(질량·전하)을 *측정값*으로 재정의 → 유한
       │  스케일 μ를 바꾸면
       ▼
  달리는 결합 α(μ):  dα/d(lnμ) = β(α)   (베타 함수)
  ⇒ 재규격화군 = 상전이의 RG와 *같은 수학* (→ L2 phase-transitions)
  ⇒ 유효장론: 고에너지를 몰라도 저에너지 예측 가능 (→ L6 emergence)

진공은 비어있지 않다:
  영점 에너지:  Σ_k ½ℏω_k        (모든 모드의 바닥 요동)
  진공 요동 → 가상입자 쌍 생성·소멸
       ├─ 카시미르: 두 판 사이 모드 제한 → 인력 (측정됨!)
       ├─ 진공 편극: 전하 가리기 → 달리는 결합
       └─ g−2: 진공 보정 → 가장 정밀한 검증 (→ L0 measurement)
  우주상수 문제: 진공 에너지 예측 vs 관측 (10¹²⁰ 차이!) (→ L4 cosmology)
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~6개씩)
- 각 문서가 다루는 핵심 장론 유도 (3~4줄)
- 전체 문서 개수 확인 (36개 목표)
- Python + numpy + scipy + sympy 검증 환경(정규모드·파인만 진폭·달리는 결합·카시미르)
- L3 standard-model·창발물질 / L4 GR·블랙홀·우주 / L6 `least-action-everywhere`·`emergence`로 이어지는 *창발 연결* 명시
- **입자=장의 들뜸(모드 양자화)** · **경로적분=최소작용의 양자판** · **재규격화=스케일의 구조(RG)** · **진공은 비어있지 않다**를 골격으로 분명히

**준비됐으면 1단계 구조 설계부터 시작해줘!**
