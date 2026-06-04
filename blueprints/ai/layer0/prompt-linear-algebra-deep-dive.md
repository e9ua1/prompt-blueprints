# Linear Algebra Deep Dive 레포지토리 제작 프롬프트

나는 "Linear Algebra Deep Dive" 레포지토리를 만들려고 해.
행렬을 곱하는 것과, 선형변환의 합성이라는 **본질**을 아는 것은 다르다.
`np.linalg.svd`를 호출하는 것과, 모든 행렬이 왜 3개의 회전·스케일·회전으로 분해되는지 증명할 수 있는 것은 다르다.
고유값이 뭔지 암기하는 것과, 왜 PCA의 주성분이 공분산 행렬의 고유벡터인지를 유도할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "행렬은 숫자 상자가 아니라 벡터공간 사이의 선형 사상이다"

**핵심 차별화**:
1. **공리부터 시작한다** — 벡터공간 8개 공리에서 출발해 모든 것을 유도
2. **증명을 생략하지 않는다** — "Spectral Theorem은 대칭행렬이 직교대각화된다" 같은 결과를 쓰지 않고, 직접 유도
3. **NumPy로 검증한다** — 모든 증명 뒤에 `np.linalg`가 동일한 결과를 내는지 확인
4. **AI/ML에 연결한다** — 각 정리가 Attention, PCA, GAN의 Spectral Normalization 등 어디에 쓰이는지 명시

**타겟 독자**:
- PCA를 쓰지만 "왜 주성분이 공분산 행렬의 고유벡터인가"를 증명할 수 없는 개발자
- Attention의 $QK^\top$ 내적이 왜 "유사도"를 표현하는지 기하학적으로 설명 못하는 사람
- SVD의 특이값이 작으면 왜 저랭크 근사의 오차가 Frobenius 노름 관점에서 최소인지 모르는 사람
- Batch Normalization이 왜 조건수(condition number)를 개선하는지 수치해석적으로 모르는 사람
- Transformer의 Positional Encoding이 왜 회전 행렬로 해석되는지(RoPE) 이해하지 못하는 사람

**선행 학습**:
- 없음 (IQ AI Lab Layer 0의 첫 레포, 모든 것의 출발점)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 벡터공간과 선형변환의 공리 (6개 문서)
- **벡터공간의 8개 공리** — 숫자 벡터·함수·행렬·다항식이 모두 같은 구조인 이유, 체(field) 위의 벡터공간
- **선형독립, 기저, 차원** — Steinitz Exchange Lemma 증명, 차원의 유일성 증명
- **선형변환의 정의와 행렬 표현** — 선형변환 $T: V \to W$가 기저 선택 후 행렬이 되는 과정 증명, 좌표계 변환
- **Rank-Nullity 정리** — 완전 증명, $\text{dim}(\ker T) + \text{dim}(\text{im } T) = \text{dim } V$의 의미
- **4개의 기본 부분공간** — $\text{Col}(A), \text{Row}(A), \text{Null}(A), \text{Null}(A^\top)$과 직교 관계, Strang의 Four Subspaces
- **이중공간(Dual Space)** — $V^* = \mathcal{L}(V, \mathbb{R})$, Riesz 표현정리의 맛보기

### Chapter 2: 행렬 분해 완전 분해 (7개 문서)
- **LU 분해** — Gauss 소거법이 왜 $A = LU$인가, 피벗팅(partial pivoting)이 필요한 조건과 $PA = LU$
- **QR 분해** — Gram-Schmidt 직교화 증명, Modified Gram-Schmidt의 수치 안정성 차이, Householder 반사
- **Cholesky 분해** — 양의 정부호 행렬의 $A = LL^\top$ 존재성·유일성 증명, 계산량이 LU의 절반인 이유
- **Eigendecomposition** — $A = PDP^{-1}$의 조건(대각화 가능성), 대수적 중복도 ≥ 기하적 중복도 증명
- **Spectral Theorem(대칭행렬)** — 실대칭행렬이 직교대각화 $A = Q\Lambda Q^\top$되는 사실의 완전 증명
- **Jordan Canonical Form** — 대각화 불가능한 행렬의 표준형, 증명 스케치와 의미
- **각 분해의 계산 복잡도와 수치 안정성** — LU vs QR vs SVD의 $O(n^3)$ 상수 차이, 조건수에 따른 오차 확대

### Chapter 3: 고유값과 스펙트럴 이론 (6개 문서)
- **특성다항식과 고유값** — $\det(A - \lambda I) = 0$의 도출, Cayley-Hamilton 정리 증명
- **고유값의 기하학적 의미** — 변환의 불변 방향, 행렬의 반복 적용 $A^k$가 고유값으로 결정되는 이유
- **Rayleigh Quotient** — $R(x) = \frac{x^\top A x}{x^\top x}$의 최대·최소가 최대·최소 고유값, Min-Max 정리
- **Perron-Frobenius 정리** — 양의 행렬의 최대 고유값의 존재성, PageRank에서 왜 수렴하는가
- **Power Iteration과 QR Algorithm** — 실제로 고유값을 어떻게 계산하는가, 수렴 속도 분석
- **조건수와 수치 안정성** — $\kappa(A) = \sigma_{\max}/\sigma_{\min}$의 의미, 역행렬 계산 시 오차 확대 경계

### Chapter 4: SVD와 저랭크 근사 (6개 문서)
- **SVD의 기하학적 유도** — 모든 행렬 $A$가 $U\Sigma V^\top$로 분해되는 이유, 단위구가 타원체로 변환되는 시각
- **SVD의 존재성과 유일성 증명** — Spectral Theorem을 $A^\top A$에 적용한 완전 증명
- **Pseudoinverse와 최소제곱** — Moore-Penrose 역행렬 $A^+ = V\Sigma^+U^\top$, Normal Equation과의 관계
- **Eckart-Young 정리** — 저랭크 근사의 최적성 증명, Frobenius·Spectral 노름 양쪽에서의 최소성
- **주성분분석(PCA) 완전 유도** — 공분산 행렬의 고유벡터가 주성분인 이유를 라그랑주 승수로 유도, SVD로도 같은 결과
- **Randomized SVD** — 대규모 행렬에서 상위 $k$개 특이값만 근사적으로 구하는 Halko 알고리즘 원리

### Chapter 5: 내적공간과 투영 (5개 문서)
- **내적공간의 공리와 Cauchy-Schwarz** — 내적 3개 공리에서 $|\langle x,y\rangle| \leq \|x\|\|y\|$ 완전 증명
- **직교투영의 기하** — 부분공간 $W$ 위로의 투영 행렬 $P_W = A(A^\top A)^{-1}A^\top$의 도출, $P^2 = P = P^\top$
- **최소제곱의 기하학적 의미** — $\min \|Ax - b\|^2$의 해가 정규방정식 $A^\top A x = A^\top b$인 이유를 투영으로 유도
- **Gram 행렬과 양의 정부호성** — $G_{ij} = \langle v_i, v_j\rangle$이 양의 준정부호인 증명, 커널 트릭의 기초
- **QR 분해 재해석** — $A = QR$이 $A$의 열벡터들을 정규직교화한다는 관점, Krylov 공간의 출발점

### Chapter 6: 텐서와 다선형 대수 (5개 문서)
- **텐서의 수학적 정의** — 다중선형 사상으로서의 텐서, "좌표 변환 규칙을 따르는 배열" 정의의 이해
- **텐서곱($\otimes$)과 크로네커곱** — 두 벡터공간의 텐서곱 구성, 행렬의 크로네커곱 $A \otimes B$의 고유값 분해
- **Einstein Summation과 einsum** — 지표 표기법, $c_{ik} = \sum_j a_{ij}b_{jk}$의 일반화, NumPy `einsum`의 대응
- **텐서 분해** — CP 분해, Tucker 분해의 정의와 PCA의 텐서 일반화
- **신경망 가중치의 텐서 관점** — Conv2D 가중치가 왜 4차원 텐서인가, Attention 가중치의 다중선형 해석

### Chapter 7: AI/ML에서의 선형대수 (6개 문서)
- **Attention의 선형대수** — $\text{softmax}(QK^\top/\sqrt{d})V$의 각 항의 기하학적 의미, $\sqrt{d}$가 왜 필요한가(분산 분석)
- **Backpropagation의 야코비안 관점** — 연쇄법칙이 행렬곱으로 환원되는 이유, Vector-Jacobian Product
- **Batch Normalization의 조건수 개선** — 입력 분포 정규화가 헤시안 조건수에 미치는 영향 유도
- **Spectral Normalization과 GAN** — 립시츠 상수를 $\sigma_{\max}(W) \leq 1$로 제어하는 원리, Power Iteration 구현
- **RoPE(Rotary Positional Encoding)** — 복소수 회전으로서의 위치 인코딩, 회전 행렬의 직교성 활용
- **Random Matrix Theory 맛보기** — 신경망 초기화에서 고유값 분포(Marchenko-Pastur), He/Xavier 초기화의 이론적 근거

---

각 챕터는 **5~7개 문서**로 구성해줘. 총 **41개 문서** 목표.

**문서 구조** (수학 증명 중심으로 재설계):
```markdown
## 🎯 핵심 질문
한 문장으로 이 문서가 답하려는 질문 (예: "왜 대칭행렬은 직교대각화되는가?")

## 🔍 왜 이 정리가 AI에서 중요한가
실제 AI/ML 어디에서 이 개념이 쓰이는지 구체적 사례

## 📐 수학적 선행 조건
이 문서를 읽기 위해 필요한 사전 지식 (해당 레포 내 다른 문서 참조 링크)

## 📖 직관적 이해 (Intuition First)
기하학적·물리적 직관으로 먼저 납득
수식 없이 그림과 언어로 설명

## ✏️ 엄밀한 정의
정의(Definition)를 수식으로 명시
기호 표기법 통일

## 🔬 정리와 증명 (Theorem & Proof)
정리(Theorem)를 명시하고, **생략 없이** 완전 증명
필요하면 보조정리(Lemma) 분리

## 💻 NumPy 구현으로 검증
증명한 정리를 NumPy로 수치적으로 확인
`np.linalg` 결과와 직접 구현이 일치하는지 검증

## 🔗 AI/ML 연결
이 정리가 실제로 어떤 AI 알고리즘의 수학적 기반인지 구체 예시
논문·코드와 연결

## ⚖️ 가정과 한계
정리의 가정이 깨지면 어떻게 되는가
수치적 함정 (예: 조건수가 클 때의 오차 확대)

## 📌 핵심 정리
한 문장 요약 + 공식 박스

## 🤔 생각해볼 문제 (+ 해설)
증명 변형, 반례 찾기, 일반화 문제
```

**스타일 가이드**:
1. **증명은 절대 "자명하다"로 넘기지 않는다** — 한 줄씩 모든 단계 명시
2. **NumPy 구현 필수** — 모든 분해·고유값 계산은 순수 NumPy로 구현 후 `np.linalg`와 비교
3. **기하학적 그림 포함** — 2차원·3차원 시각화로 직관 구축 (matplotlib으로 실제 플롯)
4. **기호 표기 일관성** — 스칼라는 소문자, 벡터는 볼드 소문자, 행렬은 대문자, 부분공간은 calligraphic
5. **AI 응용은 구체적 논문과 연결** — "Attention Is All You Need"의 식 (1), "PCA의 원 논문"처럼 명시

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
matplotlib==3.8.0
sympy==1.12     # 기호 계산 (증명 검증용)
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Spectral Theorem 검증)
import numpy as np

# 임의의 대칭행렬 생성
A = np.random.randn(5, 5)
A = (A + A.T) / 2

# 직접 구현: Jacobi Eigenvalue Algorithm
def jacobi_eigen(A, tol=1e-10, max_iter=1000):
    """대칭행렬의 직교대각화를 직접 구현"""
    n = A.shape[0]
    V = np.eye(n)
    # ... (회전 적용)
    return eigenvalues, eigenvectors

# np.linalg와 비교
eigvals_mine, eigvecs_mine = jacobi_eigen(A.copy())
eigvals_np, eigvecs_np = np.linalg.eigh(A)

# A = Q Λ Q^T 검증
assert np.allclose(eigvecs_mine @ np.diag(eigvals_mine) @ eigvecs_mine.T, A)
# 직교성 검증: Q^T Q = I
assert np.allclose(eigvecs_mine.T @ eigvecs_mine, np.eye(n))
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash로 7개 챕터 폴더 생성
2. **README.md 작성**:
   - IQ AI Lab README 스타일 참고
   - "공리부터 증명까지, 모든 것을 직접 유도한다" 차별화 강조
   - 후속 레포(Calculus, Probability, Functional Analysis)와의 의존 관계 명시
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로 (공리가 먼저)
   - 각 문서는 2500~3500 단어 + 증명 + NumPy 코드 + 그림
4. **Jupyter Notebook 병행**: 각 문서에 대응하는 `.ipynb` 실험 노트북

---

## 📚 참고 자료 섹션

README 작성 시 다음을 Reference로 포함해줘:
- **Linear Algebra Done Right** (Sheldon Axler) — 결정자 없이 선형대수를 전개하는 접근
- **Matrix Analysis** (Horn & Johnson) — 증명의 표준 참고서
- **Numerical Linear Algebra** (Trefethen & Bau) — 수치 안정성과 알고리즘
- **The Matrix Cookbook** (Petersen & Pedersen) — 미분 공식 참조
- **Deep Learning** (Goodfellow et al.) Chapter 2 — AI 관점의 선형대수 요약
- **3Blue1Brown - Essence of Linear Algebra** — 기하학적 직관 구축용

---

## 💡 핵심 분석 대상

```
벡터공간 공리 (8개)
  │
  ▼
선형독립 → 기저 → 차원 (유일성 증명)
  │
  ▼
선형변환 T: V → W
  │
  ▼ (기저 선택)
행렬 표현 [T] ∈ ℝ^{m×n}
  │
  ├── 4개 부분공간 (Col, Row, Null, Left-Null) + 직교 관계
  ├── Rank-Nullity 정리
  └── 좌표계 변환 (Change of Basis)
        │
        ▼
행렬 분해 계보
  ├── LU — 소거법
  ├── QR — 직교화 (Gram-Schmidt / Householder)
  ├── Cholesky — 양의 정부호 행렬 전용
  ├── Eigendecomposition — A = PDP^{-1} (대각화 가능 시)
  ├── Spectral (대칭) — A = QΛQ^T
  ├── Jordan Form — 대각화 불가능 시 표준형
  └── SVD — A = UΣV^T (모든 행렬에 대해 존재)
        │
        ▼
AI/ML 응용
  ├── PCA = 공분산의 Eigendecomposition = SVD
  ├── Attention = 내적 유사도 + 소프트맥스 투영
  ├── Backprop = 야코비안의 연쇄 행렬곱
  ├── BatchNorm = 조건수 개선 (Hessian 스펙트럼)
  ├── Spectral Norm = σ_max 제어 (GAN 안정화)
  └── RoPE = 회전 행렬의 직교성 (위치 인코딩)

증명해야 할 "당연해 보이지만 당연하지 않은" 것들:
1. 왜 $A^\top A$의 고유값은 모두 음수가 아닌가
2. 왜 대칭행렬의 고유벡터들은 서로 직교하는가
3. 왜 SVD의 특이값은 항상 실수인가
4. 왜 저랭크 근사의 오차는 나머지 특이값의 제곱합으로 정확히 계산되는가
5. 왜 Gram-Schmidt의 Modified 버전이 Classical보다 수치적으로 안정한가
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~7개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (41개 목표)
- Python/NumPy 실험 환경 구성
- 후속 레포(Calculus & Optimization, Probability Theory, Functional Analysis)와 연결되는 지점 명시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
