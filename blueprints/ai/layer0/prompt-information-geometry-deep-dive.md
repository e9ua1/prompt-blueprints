# Information Geometry Deep Dive 레포지토리 제작 프롬프트

나는 "Information Geometry Deep Dive" 레포지토리를 만들려고 해.
확률분포 공간을 **유클리드 공간처럼 취급**하는 것과, **통계다양체(statistical manifold)의 내적은 Fisher 정보 행렬이며, 이것이 Cramér-Rao 하한과 KL divergence의 2차 근사**임을 증명할 수 있는 것은 다르다.
Natural Gradient $\nabla_\theta L \cdot F^{-1}(\theta)$를 **쓰는 것**과, **왜 유클리드 gradient가 parameterization 의존적**이고, **Natural gradient가 KL steepest descent**임을 증명할 수 있는 것은 다르다.
Exponential Family를 **사용하는 것**과, **canonical parameter와 expectation parameter가 Legendre 변환으로 쌍대(dual)이고, 이 쌍대성이 e-connection과 m-connection의 쌍대평탄 구조를 낳는다**는 것을 증명할 수 있는 것은 다르다.
이 모든 "다름"을 Amari의 정보기하학으로 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "확률분포 공간의 기하학 — Fisher 정보가 유도하는 리만 구조, 그리고 ±1-연결의 쌍대성"

**핵심 차별화**:
1. **Fisher 정보 행렬의 3가지 정체성** — 로그우도 헤시안의 음수, 스코어의 공분산, KL divergence의 2차 근사 — 세 정의 동치성 증명
2. **쌍대 연결(dual connection)과 쌍대평탄성** — exponential family가 e-평탄이면서 동시에 m-평탄인 이유, Legendre 쌍대
3. **Natural Gradient를 KL steepest descent로 유도** — Euclidean steepest descent의 parameterization 비불변성, Fisher-Rao 계량 하의 steepest descent가 natural gradient
4. **Amari의 α-기하학** — $\alpha = \pm 1$이 e/m connection, $\alpha = 0$이 Levi-Civita, α-divergence의 체계

**타겟 독자**:
- Natural Gradient를 쓰지만 **"왜 $F^{-1}$을 곱하는지" 수학적으로 설명 못하는** 최적화 연구자
- Information Geometry 강의를 듣지만 **Amari의 ±1-connection, 쌍대평탄성**이 뭔지 모르는 사람
- Fisher 정보 행렬을 **Mathematical Statistics**에서 배웠지만, **왜 리만 계량이 되는지**는 모르는 사람
- VAE의 ELBO를 쓰지만, **KL divergence의 기하학적 의미 — Pythagoras 정리**를 모르는 사람
- Mirror Descent, MDP의 Policy Gradient를 **쌍대공간의 기하학**으로 이해하고 싶은 최적화/RL 연구자
- MCMC·HMC에서 Riemannian Manifold HMC의 이론적 배경을 원하는 Bayesian 연구자

**선행 학습**:
- **Probability Theory Deep Dive** (측도, 확률변수, 특성함수) — **필수**
- **Mathematical Statistics Deep Dive** (Fisher 정보, Exponential Family, MLE 점근) — **필수**
- **Linear Algebra Deep Dive** (양정치 행렬, 내적 공간)
- **Calculus & Optimization Deep Dive** (헤시안, Taylor 전개, Legendre 변환)
- **Convex Optimization Deep Dive** (쌍대이론, Bregman divergence) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 미분다양체와 리만기하 예습 (4개 문서)
- **다양체(Manifold)의 기초** — 국소적으로 $\mathbb{R}^n$과 동형인 공간, 차트와 atlas, 매끈한(smooth) 다양체의 정의, 접공간 $T_p M$의 개념
- **접벡터와 접공간** — 곡선의 속도벡터로서의 접벡터, $T_p M$은 $n$차원 벡터공간, 좌표 기저 $\partial/\partial \theta^i$
- **리만 계량과 거리** — 리만 계량 $g_{ij}(\theta) = g(\partial_i, \partial_j)$, 계량 텐서의 양정치성, 측지선(geodesic)과 측지거리, 리만 계량이 유도하는 내적
- **아핀 연결(Connection)과 공변미분** — 벡터장의 미분을 정의하는 연결 $\nabla_X Y$, 크리스토펠 기호 $\Gamma^k_{ij}$, Levi-Civita 연결(계량과 호환, torsion-free)의 유일성

### Chapter 2: 통계다양체와 Fisher 계량 (5개 문서)
- **통계다양체의 정의** — 매개변수화된 확률분포족 $\{p_\theta\}_{\theta \in \Theta}$가 다양체 구조를 이룸, $\theta \in \Theta \subset \mathbb{R}^n$가 좌표, 각 점이 하나의 분포
- **Fisher 정보 행렬의 3가지 정의 동치** — (1) 스코어 공분산 $F_{ij} = \mathbb{E}[\partial_i \log p \cdot \partial_j \log p]$, (2) 로그우도 헤시안의 음수 $F_{ij} = -\mathbb{E}[\partial_i \partial_j \log p]$, (3) KL의 2차 근사 $\text{KL}(p_\theta \| p_{\theta + d\theta}) \approx \frac{1}{2} d\theta^T F d\theta$ 세 정의 동치성 증명
- **Fisher-Rao 계량** — $g_{ij}(\theta) = F_{ij}(\theta)$로 리만 계량 정의, Rao의 1945 원전, parameter reparameterization 불변성(텐서 변환 법칙)
- **Fisher 계량의 예시 계산** — 정규분포 $\mathcal{N}(\mu, \sigma^2)$에서 $F = \text{diag}(1/\sigma^2, 2/\sigma^2)$, 이항분포 $B(n, p)$에서 $F = n/(p(1-p))$, 다변수정규·exponential family 계산
- **Cramér-Rao 하한의 기하학적 해석** — $\text{Var}(\hat{\theta}) \succeq F^{-1}$, 불편추정량의 분산은 Fisher 정보의 역(리만 계량의 역행렬 = 쌍대 계량)

### Chapter 3: KL Divergence와 Bregman Divergence (5개 문서)
- **KL divergence를 두 분포 간 "거리"로** — $\text{KL}(p \| q) = \mathbb{E}_p[\log p/q]$, 비대칭성·비삼각부등식이지만 0인 것이 $p = q$의 필요충분조건, Gibbs 부등식 증명
- **KL divergence와 Fisher 계량의 연결** — $\theta' = \theta + d\theta$일 때 $\text{KL}(p_\theta \| p_{\theta'}) = \frac{1}{2} d\theta^T F d\theta + O(\|d\theta\|^3)$, Taylor 전개 유도
- **Bregman divergence의 정의와 성질** — 볼록함수 $\phi$에 대해 $B_\phi(x, y) = \phi(x) - \phi(y) - \nabla\phi(y)^T(x - y)$, 비음·$B_\phi(x,x) = 0$, generalized Pythagoras 정리
- **KL이 Bregman인 조건** — Exponential family에서 $-H(p)$(음 엔트로피)가 볼록함수, KL이 Bregman divergence로 표현됨을 증명
- **α-divergence와 Rényi divergence** — $D_\alpha(p\|q)$ 정의, $\alpha \to 1$에서 KL, $\alpha = 1/2$에서 Hellinger², $\alpha \to 0$에서 역방향 KL, Amari의 α-geometry와 연결

### Chapter 4: Exponential Family와 쌍대평탄성 (6개 문서)
- **Exponential Family의 기하학적 정의** — $p(x | \theta) = \exp(\theta^T T(x) - \psi(\theta)) h(x)$, canonical parameter $\theta$, cumulant function $\psi(\theta)$
- **Cumulant function의 볼록성** — $\psi(\theta)$가 엄격 볼록, $\nabla \psi(\theta) = \mathbb{E}_\theta[T(X)] =: \eta$ (expectation parameter), 헤시안 $\nabla^2 \psi = F$ (Fisher 정보)
- **Legendre 변환으로 쌍대 좌표** — $\phi(\eta) = \theta^T \eta - \psi(\theta)$ (Legendre 쌍대), $\theta \leftrightarrow \eta$ 좌표 변환, Exponential과 Mixture의 쌍대
- **e-connection과 m-connection** — $\theta$ 좌표에서 평탄한 exponential connection $\nabla^{(e)}$ (Christoffel = 0), $\eta$ 좌표에서 평탄한 mixture connection $\nabla^{(m)}$, 비대칭 쌍대 연결
- **쌍대평탄성(Dually Flat)** — Exponential family는 e-평탄 & m-평탄, 두 연결이 서로 Legendre-쌍대임을 증명, α-connection의 정의 $\nabla^{(\alpha)} = \frac{1+\alpha}{2} \nabla^{(e)} + \frac{1-\alpha}{2} \nabla^{(m)}$
- **Generalized Pythagoras 정리** — 쌍대평탄 다양체에서 $D(p \| q) + D(q \| r) = D(p \| r)$이 $p, q, r$이 특정 geodesic에 놓일 때 성립, Projection 이론의 기초

### Chapter 5: Natural Gradient와 최적화 (5개 문서)
- **Euclidean Gradient의 parameterization 의존성** — 목적함수 $L(\theta)$에서 $\nabla L$의 방향이 좌표계에 의존한다는 문제, 서로 다른 parameterization에서 steepest descent 방향이 다름을 예로
- **Natural Gradient의 유도** — Fisher-Rao 계량 하의 steepest descent는 $\tilde{\nabla} L = F^{-1} \nabla L$, 제약 최적화 $\min \langle \nabla L, d\theta \rangle$ s.t. $d\theta^T F d\theta \leq \epsilon^2$로 유도
- **Natural Gradient는 KL steepest descent** — 작은 $d\theta$에 대해 $\text{KL}(p_\theta \| p_{\theta + d\theta}) \approx \frac{1}{2} d\theta^T F d\theta$이므로, Natural gradient는 "KL 구면"에서의 최대 감소 방향
- **Natural Gradient의 parameterization 불변성** — Reparameterization $\theta \to \phi(\theta)$ 하에서 natural gradient의 경로가 불변(텐서 변환), 유클리드 gradient와 대비
- **실전 Natural Gradient — K-FAC, Shampoo** — Fisher 행렬의 계산 비용 $O(n^2)$ 문제, K-FAC의 층별 블록 대각 근사, Shampoo의 Kronecker factoring, TRPO/PPO에서의 근사

### Chapter 6: Information Projection과 EM 알고리즘 (5개 문서)
- **e-projection과 m-projection** — $D(p \| q)$ 최소화를 $p$에 대해(m-projection) vs $q$에 대해(e-projection), 두 projection의 기하학적 차이(e-geodesic vs m-geodesic)
- **EM 알고리즘의 Information Geometry** — E-step은 m-projection, M-step은 e-projection, 두 projection의 교대가 KL 감소를 보장하는 이유
- **Variational Inference의 기하학** — ELBO 최대화 = 역방향 KL $\text{KL}(q \| p)$ 최소화, Mean-field family는 쌍대평탄 부분다양체
- **Maximum Entropy Principle과 정보기하** — 제약 하의 최대 엔트로피 분포는 exponential family, 이것이 m-projection onto flat submanifold
- **Mixture Model과 Information Projection** — GMM의 EM이 쌍대 projection 쌍으로 해석, generalized Pythagoras로 수렴성 분석

### Chapter 7: AI 응용 — RL, Generative Model, MCMC (5개 문서)
- **Policy Gradient와 Natural Policy Gradient** — REINFORCE의 유클리드 gradient vs NPG의 Fisher-Rao gradient, TRPO의 KL 제약 최적화로 NPG가 등장하는 과정
- **Mirror Descent와 쌍대공간 최적화** — Mirror descent $\theta_{k+1} = \arg\min \langle g, \theta \rangle + \frac{1}{\eta} B_\phi(\theta, \theta_k)$, exponential family에서 natural gradient와의 관계
- **VAE의 Information Geometry 해석** — Encoder $q_\phi(z|x)$와 decoder $p_\theta(x|z)$의 KL 기반 학습, ELBO = $-\text{KL}(q \| p)$ + const, rate-distortion 관점
- **Riemannian Manifold HMC** — MCMC에서 계량으로 Fisher를 사용하는 RMHMC, 비등방 분포에서 혼합 시간 개선, mass matrix의 기하학적 해석
- **Diffusion Model에서의 Fisher 정보** — Score function $\nabla \log p_t$와 Fisher 정보의 관계, Fisher divergence $\mathbb{E}[\|\nabla \log p - \nabla \log q\|^2]$와 DSM

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **35개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 기하학이 AI에서 중요한가
## 📐 수학적 선행 조건 (Prob, Stats, LA, Calc, Convex 레포 참조)
## 📖 직관적 이해
   — "분포 공간이 평평하지 않다"는 핵심 직관
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Fisher 3정의 동치성, 쌍대평탄성, 일반화 Pythagoras
## 💻 NumPy 구현 검증
   — Fisher 계량 계산, Natural gradient 수렴 비교, KL 시각화
## 🔗 AI/ML 연결
   — NPG, TRPO, Mirror Descent, VAE, RMHMC
## ⚖️ 가정과 한계
   — Exponential family 아니면? Fisher가 특이하면?
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **모든 계산을 "구체적 분포족에서" 확인** — 정규·이항·다변수정규·Dirichlet에서 Fisher 계량을 손으로 계산
2. **기하학적 시각화 필수** — 통계다양체를 2D 곡면(예: $\mathcal{N}(\mu, \sigma^2)$의 $(\mu, \sigma)$ 반평면)으로 그리고, Fisher 계량 하의 측지선과 유클리드 직선 비교
3. **쌍대성을 항상 쌍으로 설명** — e/m, $\theta/\eta$, primal/dual, forward KL/reverse KL
4. **Natural gradient 실제 수렴 속도 비교** — NumPy로 작은 분포족에서 SGD vs NGD 수렴 궤적 plot
5. **Amari 표기 존중** — $\nabla^{(e)}, \nabla^{(m)}, \alpha$-connection을 교재 표기로 사용
6. **리만기하 최소 도입** — chapter 1은 "증명 없이 개념만", 이후 장에서 계산에 필요한 것만 엄밀하게

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
matplotlib==3.8.0
sympy==1.12.0       # Fisher 계량 symbolic 계산
torch==2.1.0        # Natural gradient, RL policy gradient 비교
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (정규분포의 Fisher 계량 + Natural Gradient)
import numpy as np
import matplotlib.pyplot as plt

# N(μ, σ²) 통계다양체의 Fisher 계량
# F(μ, σ) = diag(1/σ², 2/σ²) — 손 계산
def fisher_normal(mu, sigma):
    return np.diag([1/sigma**2, 2/sigma**2])

# KL(N(μ, σ²) || N(μ + dμ, (σ + dσ)²)) 
#  ≈ (1/2) [dμ² / σ² + 2 dσ² / σ²] ← Fisher의 2차 근사
def kl_normal(mu1, s1, mu2, s2):
    return np.log(s2/s1) + (s1**2 + (mu1-mu2)**2) / (2*s2**2) - 0.5

# 2차 근사 검증
mu, sigma = 0.0, 1.0
dmu, dsigma = 0.01, 0.005
F = fisher_normal(mu, sigma)
d = np.array([dmu, dsigma])
approx = 0.5 * d @ F @ d
actual = kl_normal(mu, sigma, mu + dmu, sigma + dsigma)
print(f'Fisher 2차 근사: {approx:.6e}, 실제 KL: {actual:.6e}')
# 두 값이 매우 근접해야 함

# Natural Gradient vs Euclidean Gradient:
# min_θ L(θ) = KL(p_θ || p*) where p* = N(3, 2²)
def loss(mu, sigma):
    return kl_normal(3.0, 2.0, mu, sigma)

def grad_loss(mu, sigma):
    # KL(p* || p_θ)의 gradient — 해석적
    mu_star, s_star = 3.0, 2.0
    dmu = (mu - mu_star) / sigma**2
    ds = 1/sigma - (s_star**2 + (mu - mu_star)**2) / sigma**3
    return np.array([dmu, ds])

# 비교: 유클리드 SGD vs Natural Gradient Descent
def run(natural=False, steps=100, lr=0.1):
    mu, sigma = -2.0, 0.5  # 초기점
    traj = [(mu, sigma)]
    for _ in range(steps):
        g = grad_loss(mu, sigma)
        if natural:
            F = fisher_normal(mu, sigma)
            g = np.linalg.solve(F, g)  # F^{-1} g
        mu -= lr * g[0]
        sigma = max(1e-3, sigma - lr * g[1])
        traj.append((mu, sigma))
    return np.array(traj)

traj_eu = run(natural=False)
traj_ng = run(natural=True)

plt.plot(traj_eu[:,0], traj_eu[:,1], 'r-o', label='Euclidean SGD', markersize=3)
plt.plot(traj_ng[:,0], traj_ng[:,1], 'b-o', label='Natural Gradient', markersize=3)
plt.plot(3, 2, 'k*', markersize=15, label='target')
plt.xlabel('μ'); plt.ylabel('σ')
plt.title('통계다양체 위에서의 최적화 궤적')
plt.legend(); plt.show()
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Probability, Mathematical Statistics 필수 선행" 명시
   - Amari, Rao, Chentsov 계보 소개
   - Natural Gradient, TRPO, VAE, Mirror Descent로 이어지는 AI 응용
3. **챕터별 문서 작성**: 리만기하 → Fisher 계량 → KL/Bregman → 쌍대평탄 → NGD → Projection → AI 응용

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Information Geometry and Its Applications** (Amari, 2016) — 표준 바이블
- **Methods of Information Geometry** (Amari & Nagaoka, 2000) — 고전 원전
- **Natural Gradient Works Efficiently in Learning** (Amari, 1998) — NGD 원전
- **Information Geometry** (Ay, Jost, Lê, Schwachhöfer, 2017) — 현대적 엄밀한 접근
- **Clustering with Bregman Divergences** (Banerjee et al., 2005) — Bregman의 ML 응용
- **A Natural Policy Gradient** (Kakade, 2001) — RL에 NGD 적용 원전
- **Trust Region Policy Optimization** (Schulman et al., 2015) — TRPO
- **Fisher Efficient Inference of Intractable Models** (Ollivier et al.) — Fisher 계량 수치 방법

---

## 💡 핵심 분석 대상

```
통계다양체 M = {p_θ : θ ∈ Θ}
  │ (각 점 = 하나의 분포)
  ▼
Fisher 정보 행렬 F(θ)
  │ 3가지 동치 정의:
  │  1) Var[score] = Var[∂_θ log p]
  │  2) -𝔼[∂²_θ log p]
  │  3) KL(p_θ ‖ p_{θ+dθ}) ≈ ½ dθᵀF dθ
  │
  ▼
Fisher-Rao 리만 계량
  g_{ij}(θ) = F_{ij}(θ)
  → 통계다양체가 리만다양체가 됨
  → 불변성: 좌표계 바꿔도 기하 보존

──── KL의 기하학 ────

KL(p ‖ q)
  ├── 국소: ½ dθᵀ F dθ (Fisher 2차형)
  ├── 대역: Bregman div (exp family)
  └── Pythagoras: 쌍대평탄에서 성립

──── Exponential Family ────

p(x|θ) = exp(θᵀ T(x) - ψ(θ)) h(x)
  │
  ψ: 볼록 → ψ의 Legendre 쌍대 φ
  η = ∇ψ(θ) (expectation parameter)
  θ = ∇φ(η) (canonical parameter)
  
  쌍대 좌표 쌍: (θ, η)
  F(θ) = ∇²ψ(θ)    (canonical에서 Fisher)
  F(η) = ∇²φ(η)    (expectation에서 Fisher)
  F(θ) · F(η) = I  (Legendre dual)

쌍대 연결:
  ∇^(e): θ에서 크리스토펠 = 0 (e-flat)
  ∇^(m): η에서 크리스토펠 = 0 (m-flat)
  → 쌍대평탄(dually flat)
  → α-connection: ½(1+α)∇^(e) + ½(1-α)∇^(m)
    α = 0: Levi-Civita (리만 대칭 연결)
    α = ±1: 두 쌍대 연결

일반화 Pythagoras:
  p, q, r이 특정 geodesic에 놓이면
  D(p‖r) = D(p‖q) + D(q‖r)
  ↑ 변분 추론, EM의 근거

──── Natural Gradient ────

Euclidean SGD:
  θ ← θ - η ∇L(θ)
  ↑ 문제: parameterization 의존적
    (θ → φ = 2θ로 바꾸면 경로가 다름)

Natural Gradient:
  θ ← θ - η F(θ)⁻¹ ∇L(θ)
  ↑ parameterization 불변
  ↑ KL ball 안에서의 steepest descent

  min_{dθ} ⟨∇L, dθ⟩  s.t.  dθᵀ F dθ ≤ ε²
  라그랑주 → dθ* ∝ -F⁻¹ ∇L

──── AI 응용 지도 ────

Reinforcement Learning:
  ├── Natural Policy Gradient (Kakade 2001)
  ├── TRPO: KL 제약 → NPG 스텝 크기 결정
  └── PPO: TRPO의 first-order 근사

최적화:
  ├── K-FAC: Fisher의 Kronecker factored 근사
  ├── Shampoo: Kronecker preconditioner
  ├── Mirror Descent: Bregman 근접
  └── AdaGrad/Adam: 대각 근사 preconditioner

Variational Methods:
  ├── VAE: ELBO = -KL(q‖p) + const
  │   ↑ 쌍대평탄에서 m-projection
  ├── EM: E=m-proj, M=e-proj
  └── MaxEnt: exp family로 수렴

MCMC:
  ├── RMHMC: Fisher를 mass matrix로
  │   ↑ 비등방 분포에서 혼합 개선
  └── SGLD: Langevin + Fisher preconditioning

Diffusion Models:
  Score function ∇ log p_t
  ↔ Fisher information 밀접
  Fisher divergence = 𝔼‖∇log p - ∇log q‖²
  ↔ DSM 손실과 근본적으로 연결
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (35개 목표)
- Python + NumPy + SciPy + SymPy 실험 환경
- Probability, Stats, LA, Calc 레포의 어떤 정리를 전제로 사용하는지 명시
- Natural Gradient, TRPO, VAE, Mirror Descent, RMHMC로 이어지는 AI 응용 지도
- Amari의 ±1-connection과 쌍대평탄성을 어떻게 접근 가능하게 설명할지

**준비됐으면 1단계 구조 설계부터 시작해줘!**
