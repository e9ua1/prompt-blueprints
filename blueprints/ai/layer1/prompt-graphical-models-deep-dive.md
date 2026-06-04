# Graphical Models Deep Dive 레포지토리 제작 프롬프트

나는 "Graphical Models Deep Dive" 레포지토리를 만들려고 해.
조건부 독립 $A \perp\!\!\!\perp B \mid C$를 **정의로 말하는 것**과, **Bayesian Network의 d-separation이 그래프 구조만으로 모든 조건부 독립을 결정한다는 global Markov 속성**을 증명할 수 있는 것은 다르다.
Hidden Markov Model의 Viterbi/Forward-Backward를 **사용하는 것**과, **둘이 모두 sum-product algorithm의 특수 경우**이며, **factor graph 위에서 message passing이 dynamic programming의 일반화**임을 증명할 수 있는 것은 다르다.
Mean-field Variational Inference를 **쓰는 것**과, **이것이 KL 최소화 + Bethe 자유에너지 근사 + factor graph의 변분 기법**으로 통합적으로 이해되는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "확률분포의 구조를 그래프로 — 조건부 독립이 그래프 기하로 나타나는 아름다움"

**핵심 차별화**:
1. **조건부 독립의 세 가지 기하학** — d-separation (directed), graph separation (undirected), factor graph 관점의 통합
2. **Inference의 통일 프레임워크 — Message Passing** — Forward-Backward (HMM), Belief Propagation (tree), Loopy BP, Junction Tree 모두 sum-product의 특수경우
3. **Variational Inference를 factor graph에서** — Mean-field, Bethe approximation, Loopy BP의 변분 해석 (Yedidia et al. 2003)
4. **학습의 수학** — EM, Parameter learning, Structure learning, MAP inference의 복잡도 이론

**타겟 독자**:
- HMM을 쓰지만 **Forward-Backward가 왜 sum-product**인지, **Viterbi가 왜 max-product**인지 설명 못하는 사람
- CRF(Conditional Random Field)를 쓰지만 **왜 discriminative이고 HMM과 어떻게 다른지** 정확히 모르는 사람
- Variable Elimination을 알지만 **왜 tree-width가 inference 복잡도를 결정**하는지 모르는 사람
- Loopy BP를 쓰지만 **왜 tree에서는 정확하고 loop에서는 근사**인지, **Bethe 자유에너지와의 관계**를 모르는 사람
- Graphical Model과 Neural Network의 연결(예: CRF layer, attention as message passing)을 이해하고 싶은 사람

**선행 학습**:
- **Probability Theory Deep Dive** (조건부 독립, 결합분포) — **필수**
- **Mathematical Statistics Deep Dive** (MLE, MAP) — **필수**
- **Linear Algebra Deep Dive** (matrix operations) — 기초
- **Information Theory Deep Dive** (엔트로피, KL, 상호정보량) — **필수**
- **Bayesian ML Deep Dive** (Variational Inference) — 권장 (병렬 학습 가능)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 조건부 독립과 그래프 구조 (5개 문서)
- **조건부 독립의 정의와 성질** — $A \perp\!\!\!\perp B \mid C \iff P(A, B \mid C) = P(A|C) P(B|C)$, semi-graphoid 공리 (symmetry, decomposition, weak union, contraction), 그래프 독립 구조의 기초
- **Bayesian Network — DAG 기반** — $p(x_1, ..., x_n) = \prod p(x_i | \text{pa}(x_i))$로 인수분해, parents 선택이 곧 조건부 독립 가정, topological order의 중요성
- **d-separation (Directed Separation)** — 세 경로 패턴 (chain, fork, v-structure/collider)에서 blocked vs open의 규칙, d-separation ⟺ 조건부 독립 (soundness and completeness)
- **Markov Random Field — Undirected 기반** — $p(x) = \frac{1}{Z} \prod_C \phi_C(x_C)$, clique potential, partition function $Z$의 계산 난제성, Hammersley-Clifford 정리
- **Moralization과 두 모델의 변환** — DAG → undirected: parent를 moralize (공동 자식 있는 부모들을 연결), 변환이 조건부 독립 구조를 어떻게 보존 또는 손실하는가

### Chapter 2: Factor Graph와 Message Passing (5개 문서)
- **Factor Graph의 정의** — Bipartite graph (variable node + factor node), $p(x) = \frac{1}{Z}\prod_f \phi_f(x_{N(f)})$, Bayesian network과 MRF를 통합 표현
- **Sum-Product Algorithm (Belief Propagation)** — 변수→factor 메시지 $\mu_{x \to f}(x) = \prod_{f' \in N(x) \setminus f} \mu_{f' \to x}(x)$, factor→변수 메시지 $\mu_{f \to x}(x) = \sum_{x' \setminus x} f(x, x') \prod_{x'' \in N(f) \setminus x} \mu_{x'' \to f}(x'')$, tree에서 정확한 marginal
- **Max-Product Algorithm** — sum → max로 치환, MAP inference의 message passing 버전, Viterbi가 HMM에서의 max-product
- **Junction Tree Algorithm** — Chordal triangulation → junction tree 구성, cluster 간 message passing으로 loop 있는 그래프에서도 정확한 inference, treewidth가 복잡도 결정
- **Loopy BP와 수렴성** — 사이클 있는 factor graph에서 BP 반복, 일반적으로 근사이지만 실전에서 성공적, Bethe 자유에너지의 변분 고정점이라는 Yedidia-Freeman-Weiss 발견

### Chapter 3: Hidden Markov Model과 그 가족 (5개 문서)
- **HMM의 정의와 세 가지 문제** — 은닉상태 $z_t$ + 관측 $x_t$, 전이 $p(z_t|z_{t-1})$, 방출 $p(x_t|z_t$), 세 가지 표준 문제: Evaluation (likelihood), Decoding (MAP states), Learning (EM)
- **Forward-Backward Algorithm** — $\alpha_t(z) = p(x_{1:t}, z_t = z)$ forward recursion, $\beta_t(z) = p(x_{t+1:T} | z_t = z)$ backward recursion, $p(z_t | x_{1:T}) \propto \alpha_t(z) \beta_t(z)$
- **Viterbi Algorithm — Max-Product** — $\delta_t(z) = \max_{z_{1:t-1}} p(x_{1:t}, z_{1:t-1}, z_t = z)$, dynamic programming, factor graph에서 max-product로 재유도
- **Baum-Welch — EM for HMM** — E-step: Forward-Backward로 $p(z_t|x, \theta^{old})$, M-step: 전이·방출 파라미터 업데이트, EM의 HMM 특수경우
- **Linear Dynamical System과 Kalman Filter** — 연속상태 HMM, Gaussian transitions/emissions, Kalman filter = Forward algorithm의 Gaussian 버전, Rauch-Tung-Striebel smoother = Backward

### Chapter 4: Conditional Random Field와 구조화 예측 (4개 문서)
- **CRF의 정의와 Logistic Regression 확장** — $p(y|x) = \frac{1}{Z(x)}\exp(\sum_k w_k f_k(y, x))$, sequence labeling에서 linear-chain CRF, HMM과의 차이 (discriminative vs generative)
- **Linear-Chain CRF의 Inference와 Learning** — Forward-Backward 유사 알고리즘으로 partition function과 marginal 계산, log-likelihood의 gradient는 기대값 차이 $\sum f_k - \mathbb{E}_p[\sum f_k]$
- **General CRF와 구조화 예측** — Skip-chain CRF, Tree CRF, Grid CRF (이미지 분할), Structured SVM과의 관계
- **Neural CRF와 딥러닝 통합** — BiLSTM-CRF, Transformer + CRF, Named Entity Recognition에서의 표준 아키텍처, End-to-end 학습

### Chapter 5: Variable Elimination과 Exact Inference (4개 문서)
- **Variable Elimination** — 변수 순서에 따라 marginalization, $\sum_x f_1(x, y) f_2(x, z) = (\sum_x f_1 f_2)(y, z)$, ordering이 intermediate factor 크기 결정
- **Treewidth와 Inference Complexity** — Elimination ordering에서 최대 clique 크기 - 1 = treewidth, Inference 복잡도 $O(n \cdot d^{\text{tw}+1})$, NP-hard problem of min treewidth
- **Clique Tree와 Junction Tree** — Triangulation으로 chordal graph 만들기, clique간 running intersection property, belief propagation on junction tree
- **Inference의 복잡도 이론** — MAP inference NP-hard in general, marginal inference #P-hard, PTAS가 있는 특수 경우

### Chapter 6: Approximate Inference (6개 문서)
- **Variational Inference on Factor Graphs** — Mean-field의 factorized $q(x) = \prod q_i(x_i)$, ELBO를 각 $q_i$에 대해 최적화, 반복 업데이트와 수렴
- **Bethe Free Energy와 Loopy BP의 변분 해석** — Bethe approximation $F_{\text{Bethe}} = U - H_{\text{Bethe}}$가 사이클 없는 그래프에서 정확, Loopy BP의 고정점 = Bethe 자유에너지의 정체점 (Yedidia et al. 2003)
- **Expectation Propagation (EP)** — Assumed density filtering + iterative refinement, 각 factor를 Gaussian (또는 ef family)로 근사하여 tractable posterior 유지, GP classification에 자주 사용
- **Gibbs Sampling on Graphical Models** — MRF/BN에서 Gibbs = 조건부 $p(x_i | x_{-i})$ 샘플링, Markov blanket 정리로 local computation, Ising model·LDA 등의 표준
- **Particle Filter와 Sequential Monte Carlo** — 비선형·비Gaussian state-space model에서 posterior sampling, importance resampling, 현대적 variants (auxiliary PF, resample-move)
- **MCMC on Trans-dimensional Spaces (RJMCMC)** — 모델 구조 자체가 불확실할 때 차원이 변하는 MCMC, Green의 Reversible Jump

### Chapter 7: 학습과 현대 응용 (5개 문서)
- **Maximum Likelihood for Graphical Models** — BN에서 complete data면 count-based MLE, MRF에서 gradient는 기대값 차이 (Partition function 때문에 intractable), Contrastive Divergence
- **EM Algorithm — 불완전 데이터** — Latent variable 모델에서 $Q(\theta | \theta^{old}) = \mathbb{E}_{p(z|x, \theta^{old})}[\log p(x, z|\theta)]$ 최대화, ELBO의 lower bound로 monotonic improvement
- **Structure Learning** — Score-based (BIC, BDeu) vs Constraint-based (PC algorithm, IC), NP-hard in general, 휴리스틱 greedy search, Chow-Liu tree (restricted to tree)
- **Topic Model (LDA)의 그래프 모델** — Latent Dirichlet Allocation이 hierarchical Bayesian network, Variational EM과 collapsed Gibbs sampling 유도
- **Graph Neural Network과 Message Passing 일반화** — GNN의 message passing이 factor graph BP의 학습된 버전, Transformer's attention = fully-connected soft message passing, 그래프 ML과 PGM의 수렴

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **34개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 그래프 모델이 ML에서 유용한가
## 📐 수학적 선행 조건 (Prob, Info, Stats, Bayes 레포 참조)
## 📖 직관적 이해
   — 그래프 구조로 조건부 독립 "읽는" 연습
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — d-sep soundness, Hammersley-Clifford, BP on tree 정확성
## 💻 NumPy 구현 검증
   — HMM/Kalman 바닥부터, BP on factor graph
   — NetworkX로 그래프 시각화
## 🔗 현대 ML과의 연결
   — CRF layer, GNN, attention
## ⚖️ 가정과 한계
   — Treewidth 폭발, loopy BP 발산
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **그래프 시각화 전 챕터에서 필수** — NetworkX + matplotlib로 모든 예제 그래프 그리기
2. **d-separation 문제 많이** — 구체적 DAG 주고 $A \perp\!\!\!\perp B | C$ 판정하는 연습
3. **Message Passing을 단계별로 애니메이션식 시각화** — 각 iteration에서 메시지 업데이트를 그림으로
4. **HMM → CRF → GNN → Transformer 계보 명시** — 각 모델이 어떻게 일반화되는지
5. **실전 데이터 — 문장 POS 태깅, 이미지 분할, LDA topic model** — 각 영역의 대표 예제
6. **Tree vs Loopy 차이 실험** — 같은 그래프에 BP 적용해 수렴·정확성 비교

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
matplotlib==3.8.0
networkx==3.2.0       # 그래프 구조 시각화
pgmpy==0.1.25         # PGM 라이브러리 (검증용)
hmmlearn==0.3.0       # HMM 비교
scikit-learn==1.3.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (HMM의 Forward-Backward를 factor graph로 구현 + Viterbi)
import numpy as np
import matplotlib.pyplot as plt

# 작은 HMM
n_states, n_obs = 3, 2
pi = np.array([0.5, 0.3, 0.2])   # 초기분포
A = np.array([[0.7, 0.2, 0.1],   # 전이
              [0.1, 0.6, 0.3],
              [0.2, 0.2, 0.6]])
B = np.array([[0.9, 0.1],         # 방출
              [0.5, 0.5],
              [0.2, 0.8]])

# 관측 시퀀스
obs = [0, 1, 0, 0, 1, 1, 0]
T = len(obs)

# Forward algorithm
alpha = np.zeros((T, n_states))
alpha[0] = pi * B[:, obs[0]]
for t in range(1, T):
    alpha[t] = (alpha[t-1] @ A) * B[:, obs[t]]
likelihood = alpha[-1].sum()
print(f'P(obs) = {likelihood:.6f}')

# Backward algorithm
beta = np.zeros((T, n_states))
beta[-1] = 1.0
for t in range(T-2, -1, -1):
    beta[t] = A @ (B[:, obs[t+1]] * beta[t+1])

# Posterior marginal γ_t(i) = p(z_t = i | obs)
gamma = alpha * beta
gamma = gamma / gamma.sum(axis=1, keepdims=True)

# Viterbi (max-product)
delta = np.zeros((T, n_states))
psi = np.zeros((T, n_states), dtype=int)
delta[0] = pi * B[:, obs[0]]
for t in range(1, T):
    for j in range(n_states):
        scores = delta[t-1] * A[:, j]
        psi[t, j] = np.argmax(scores)
        delta[t, j] = scores.max() * B[j, obs[t]]

# Backtrack
z_star = np.zeros(T, dtype=int)
z_star[-1] = np.argmax(delta[-1])
for t in range(T-2, -1, -1):
    z_star[t] = psi[t+1, z_star[t+1]]

print(f'MAP states (Viterbi): {z_star}')

# 시각화: gamma (marginal) vs Viterbi (MAP)
fig, ax = plt.subplots(figsize=(10, 4))
for i in range(n_states):
    ax.plot(gamma[:, i], marker='o', label=f'P(z_t={i}|x)')
ax.step(range(T), z_star, 'k-', linewidth=3, label='Viterbi MAP')
ax.set_xlabel('t'); ax.legend(); ax.set_title('Forward-Backward vs Viterbi')
plt.show()

# 정리: Forward-Backward는 sum-product, Viterbi는 max-product — 둘 다 BP의 특수경우
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Prob, Info, Stats 선행 필수" 명시
   - Bayesian ML과의 연결 (VI는 두 레포에서 다름, 이 레포는 그래프 구조 강조)
   - GNN, Transformer attention과의 현대적 연결
3. **챕터별 문서 작성**: 조건부독립 → Factor graph → HMM → CRF → VE → 근사추론 → 학습·응용

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Probabilistic Graphical Models: Principles and Techniques** (Koller & Friedman) — PGM 바이블
- **Pattern Recognition and Machine Learning** (Bishop) — Chap 8 Graphical Models
- **Information Theory, Inference, and Learning Algorithms** (MacKay) — BP·MRF 직관
- **Graphical Models, Exponential Families, and Variational Inference** (Wainwright & Jordan 2008) — 변분 관점의 표준 리뷰
- **Constructing Free-Energy Approximations and Generalized Belief Propagation** (Yedidia, Freeman, Weiss 2003) — Bethe/Loopy BP 원전
- **Conditional Random Fields** (Lafferty, McCallum, Pereira 2001) — CRF 원전
- **Latent Dirichlet Allocation** (Blei, Ng, Jordan 2003) — LDA 원전
- **Neural Message Passing for Quantum Chemistry** (Gilmer et al. 2017) — GNN as message passing

---

## 💡 핵심 분석 대상

```
확률모델의 그래프 표현

결합분포 p(x_1, ..., x_n)
  │
  ├── DAG로 표현: Bayesian Network
  │   p(x) = ∏_i p(x_i | pa(x_i))
  │   조건부독립: d-separation
  │
  ├── 무방향 그래프: Markov Random Field
  │   p(x) = (1/Z) ∏_C φ_C(x_C)
  │   조건부독립: graph separation
  │
  └── Factor Graph (bipartite, 통합표현)
      p(x) = (1/Z) ∏_f φ_f(x_{N(f)})

───── d-separation 3 패턴 ─────

Chain:  A → B → C
   조건 X: P(A,C) ≠ P(A)P(C) (dependent)
   C 고정: A ⊥ C | B  (blocked)

Fork:   A ← B → C   (공통 원인)
   조건 X: dependent
   B 고정: A ⊥ C | B

Collider (v-structure): A → B ← C
   조건 X: A ⊥ C  (independent!)
   B 또는 후손 고정: A, C가 dependent가 됨
   ↑ Explaining away!

───── Message Passing Zoo ─────

Tree Factor Graph (정확한 계산):
  Sum-Product → Marginal p(x_i)
  Max-Product → MAP argmax_x p(x)

HMM (특수 tree):
  Forward (sum-product):
    α_t(z) = Σ_{z'} α_{t-1}(z') A(z'→z) B(z→x_t)
  Backward:
    β_t(z) = Σ_{z'} A(z→z') B(z'→x_{t+1}) β_{t+1}(z')
  Viterbi (max-product):
    δ_t(z) = max_{z'} δ_{t-1}(z') A(z'→z) B(z→x_t)
  
  Kalman Filter = Forward의 Gaussian 버전
  RTS smoother = Backward의 Gaussian 버전

Junction Tree (일반 loop graph):
  1) Triangulation (chordal)
  2) Clique tree 구성
  3) Running intersection property
  4) Cluster 간 message passing → 정확
  Complexity: O(n · d^(treewidth + 1))

Loopy BP:
  일반 BP를 loopy graph에 적용
  ├── 일반적으로 근사
  ├── 일반적으로 수렴 보장 X
  ├── Bethe free energy의 stationary point
  └── 실전에서 놀라울 정도로 잘 동작

───── 학습 ─────

Complete Data:
  BN: count-based MLE (해석적)
  MRF: ∇ log L = 𝔼_data[f] - 𝔼_model[f]
       partition function → 근사 필요
       └── Contrastive Divergence
       └── Pseudo-likelihood

Incomplete Data:
  EM Algorithm
  E-step: Q(θ|θ^old) = 𝔼_{p(z|x,θ^old)}[log p(x,z|θ)]
  M-step: θ ← argmax Q
  
  ├── Baum-Welch: HMM의 EM
  ├── Fitting GMM: EM
  └── LDA: Variational EM or Collapsed Gibbs

Structure Learning:
  Score-based (BIC, BDeu, AIC)
  Constraint-based (PC, FCI)
  ├── 일반적으로 NP-hard
  ├── Tree에 제한시 Chow-Liu (efficient!)
  └── 현대: RL + GNN 기반

───── CRF 체계 ─────

            Generative     Discriminative
State-free  Naive Bayes  ↔  Logistic Regression
Sequential  HMM          ↔  Linear-chain CRF
General     MRF          ↔  General CRF

CRF 장점: 
  ├── feature 유연성 (context window, overlapping)
  ├── normalization이 y에 한정 (Z(x)는 쉬움)
  └── 더 나은 accuracy 일반적

───── 현대 NN과의 연결 ─────

CRF Layer (BiLSTM-CRF):
  Emission: NN이 feature
  Transition: CRF parameters
  Inference: Viterbi at test

Graph Neural Network:
  Message passing의 학습된 버전
  h_v^{(t+1)} = UPDATE(h_v^{(t)}, AGG({MSG(h_u^{(t)}, e_{uv}): u ∈ N(v)}))
  ↑ BP의 non-linear generalization

Transformer Attention:
  softmax(QK^T/√d) V
  → fully-connected soft message passing
  → 모든 노드가 서로 message 전달
  → GNN with complete graph
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (34개 목표)
- Python + NumPy + NetworkX + pgmpy 실험 환경
- Prob, Stats, Info, Bayesian ML 레포와의 참조 관계
- HMM → CRF → GNN → Transformer로 이어지는 현대적 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
