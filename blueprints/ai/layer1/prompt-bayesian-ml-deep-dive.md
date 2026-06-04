# Bayesian ML Deep Dive 레포지토리 제작 프롬프트

나는 "Bayesian ML Deep Dive" 레포지토리를 만들려고 해.
Bayes' rule $p(\theta | D) \propto p(D|\theta) p(\theta)$를 **암기하는 것**과, **왜 정규화 상수 $p(D) = \int p(D|\theta)p(\theta) d\theta$가 계산 불가능해서 VI와 MCMC가 필요**한지 설명할 수 있는 것은 다르다.
VAE의 ELBO를 **구현하는 것**과, **$\text{ELBO} = \log p(x) - \text{KL}(q(z|x) \| p(z|x))$의 좌변이 증거, 우변이 증거 하한이 되는 유도**와 **reparametrization trick의 편미분 교환 정당성**을 증명할 수 있는 것은 다르다.
Bayesian Neural Network을 **호출하는 것**과, **MC Dropout이 왜 approximate variational inference (Gal & Ghahramani 2016)**이고, **Laplace Approximation이 posterior를 헤시안 기반 Gaussian으로 근사**하는 것을 유도할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "불확실성의 ML — Bayes 추론·변분 추론·MCMC·Bayesian Deep Learning 한 흐름"

**핵심 차별화**:
1. **Bayesian 추론의 4단계 완전 유도** — Prior → Likelihood → Posterior → Predictive, 각 단계의 수학적 엄밀성
2. **ELBO를 3가지 방식으로 유도** — (1) Jensen 부등식 (2) KL divergence 분해 (3) Importance sampling 관점
3. **MCMC의 이론적 근거와 실전** — Metropolis-Hastings, Gibbs, HMC, NUTS의 수학 + 수렴 진단 ($\hat{R}$, ESS)
4. **Bayesian Deep Learning** — BNN, MC Dropout, Laplace, SWAG, SGLD 각각의 정확한 근사 의미

**타겟 독자**:
- "confidence interval이 있는 딥러닝"이 필요한 ML 연구자
- VAE를 구현하지만 **ELBO의 각 항 — reconstruction + KL — 이 왜 그렇게 나오는지** 증명 못하는 사람
- MCMC를 쓰지만 **Metropolis-Hastings acceptance rate $\min(1, ...)$의 유도**를 잊어버린 사람
- Bayesian Optimization을 쓰지만 **acquisition function (EI, UCB, TS) 각각의 Bayesian 해석**을 모르는 사람
- Variational Inference와 MCMC 중 **언제 어느 것을 쓸지** 기준을 모르는 실무자
- "Bayesian NN이 왜 uncertainty를 준다고 하는가"를 수학적으로 설명 못하는 딥러닝 엔지니어

**선행 학습**:
- **Probability Theory Deep Dive** (조건부 확률, 베이즈 정리) — **필수**
- **Mathematical Statistics Deep Dive** (MLE, MAP, Fisher) — **필수**
- **Stochastic Processes Deep Dive** (MCMC 기초, 마르코프 체인) — **필수**
- **Information Theory Deep Dive** (KL, entropy) — **필수**
- **Calculus & Optimization Deep Dive** (gradient ascent) — ELBO 최적화
- **Information Geometry Deep Dive** (Natural Gradient) — 고급 VI
- **Kernel Methods Deep Dive** (Gaussian Process) — BO와 관련

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Bayesian 추론의 기초 (5개 문서)
- **Bayes 정리와 4가지 역할** — $p(\theta|D) = p(D|\theta)p(\theta)/p(D)$, posterior = likelihood × prior / evidence, 각 항의 이름과 의미, 빈도주의와의 철학적 차이
- **MLE vs MAP vs Full Bayesian** — 3단계 점추정·분포 비교, MAP은 delta prior의 full Bayesian의 특수경우, MLE는 uniform prior의 MAP, 왜 completion이 점진적으로 많아지는가
- **Conjugate Priors의 수학** — Likelihood와 prior가 같은 family를 유지하는 분포 쌍, Beta-Bernoulli, Gamma-Poisson, Normal-Normal 등 주요 쌍, Exponential Family와의 연결
- **Predictive Distribution** — $p(y^*|D) = \int p(y^*|\theta) p(\theta|D) d\theta$, 단일 점추정 $p(y^*|\hat{\theta})$와 차이(불확실성 포함), Bayesian averaging
- **Posterior의 점근 성질** — Bernstein-von Mises 정리: 데이터가 많으면 posterior가 MLE 주변 Gaussian으로 수렴 $\mathcal{N}(\hat{\theta}_{MLE}, F^{-1}/n)$, Fisher 정보와 uncertainty의 관계

### Chapter 2: 변분 추론 (Variational Inference) (6개 문서)
- **VI의 아이디어와 ELBO 유도** — Intractable $p(\theta|x)$를 tractable $q_\phi(\theta)$로 근사, $\text{KL}(q \| p(\cdot | x))$ 최소화가 ELBO $\mathcal{L} = \mathbb{E}_q[\log p(x, \theta) - \log q(\theta)]$ 최대화와 동치, Jensen 부등식으로도 유도
- **ELBO의 세 가지 분해** — (1) $\log p(x) - \text{KL}(q\|p(\cdot|x))$ (evidence - gap), (2) $\mathbb{E}_q[\log p(x|\theta)] - \text{KL}(q\|p(\theta))$ (reconstruction + prior regularization), (3) $-\text{KL}(q\|p(\theta)) + \mathbb{E}_q[\log p(x|\theta)]$
- **Mean-Field VI** — 분해 가정 $q(\theta) = \prod_i q_i(\theta_i)$, 좌표 최적화 업데이트 $q_i^*(\theta_i) \propto \exp \mathbb{E}_{-i}[\log p(x, \theta)]$, CAVI (Coordinate Ascent VI) 알고리즘
- **Exponential Family와 자연매개변수 VI** — Conjugate exponential family에서 CAVI의 닫힌 형태, Stochastic Variational Inference (Hoffman et al. 2013)로 대규모 데이터 처리
- **Reparameterization Trick** — $z \sim q_\phi(z|x)$를 $z = g_\phi(\epsilon, x)$로 재매개변수화 ($\epsilon$은 base 분포), $\nabla_\phi \mathbb{E}_q[f] = \mathbb{E}_{p(\epsilon)}[\nabla_\phi f(g_\phi(\epsilon))]$로 low-variance gradient
- **REINFORCE Gradient와 Control Variate** — Reparameterization 불가능한 discrete latent의 경우 score function estimator, $\nabla \mathbb{E}_q[f] = \mathbb{E}_q[f \nabla \log q]$, 분산 감소 기법

### Chapter 3: VAE와 현대 변분 모델 (5개 문서)
- **VAE 완전 유도 (Kingma & Welling 2013)** — Encoder $q_\phi(z|x)$ + Decoder $p_\theta(x|z)$, ELBO $= \mathbb{E}_{q_\phi}[\log p_\theta(x|z)] - \text{KL}(q_\phi(z|x) \| p(z))$, Gaussian $q_\phi$와 Gaussian prior에서 KL 해석해
- **VAE의 변종 — β-VAE, Conditional VAE, VQ-VAE** — β로 KL 항 가중 (disentanglement), class-conditional generation, vector quantization으로 discrete latent
- **Normalizing Flows** — 역가능 변환 $z_k = f_k \circ \ldots \circ f_1(z_0)$, Jacobian $\log p(z_k) = \log p(z_0) - \sum \log |\det J_{f_i}|$, 더 유연한 posterior
- **Amortized Inference** — 각 데이터 $x_i$마다 $q_i$ 최적화하지 않고, encoder NN $q_\phi(\cdot | x)$로 inference amortize, 계산과 표현력의 trade-off
- **Importance-weighted VAE (IWAE)** — ELBO의 tighter bound $\log \mathbb{E}_q[\frac{p(x,z)}{q(z|x)}]$를 $K$ 샘플로 추정, $K \to \infty$에서 실제 $\log p(x)$로 수렴

### Chapter 4: MCMC 실전 (6개 문서)
- **Metropolis-Hastings 재정리** — proposal $q(\theta'|\theta)$ + acceptance $\min(1, \frac{p(\theta'|D)q(\theta|\theta')}{p(\theta|D)q(\theta'|\theta)})$, detailed balance 증명 (Stochastic Processes 레포에서 이미 커버되었지만 Bayesian 맥락)
- **Gibbs Sampler와 조건부 분포** — 각 차원을 조건부 $p(\theta_i | \theta_{-i}, D)$로 업데이트, Conjugate 관계에서 효율적, Collapsed Gibbs로 일부 latent 주변화
- **Hamiltonian Monte Carlo (HMC)** — 보조 momentum $p$ 도입, Hamiltonian $H(\theta, p) = U(\theta) + K(p)$, Leapfrog integrator, gradient 활용으로 high-dimensional에서 효율적
- **No-U-Turn Sampler (NUTS)** — HMC의 step size와 trajectory length를 자동 조정, Stan/PyMC의 기본 sampler
- **수렴 진단 — $\hat{R}$과 ESS** — Gelman-Rubin $\hat{R}$: 여러 체인의 within/between variance 비교, Effective Sample Size: autocorrelation 보정된 샘플 수, 실전 진단 체크리스트
- **MCMC의 한계와 VI와의 비교** — 고차원에서 mixing 어려움, multimodal posterior, VI vs MCMC: 정확도 vs 속도, 문제 특성별 선택 기준

### Chapter 5: Bayesian Neural Networks (5개 문서)
- **BNN의 수학적 정식화** — 가중치 $W \sim p(W)$, likelihood $p(y|x, W)$, posterior $p(W|D)$ 추론 → 예측 $p(y^*|x^*, D) = \int p(y^*|x^*, W) p(W|D) dW$, intractable
- **Laplace Approximation** — Posterior 최빈값 $W^*$ 주변 2차 Taylor: $p(W|D) \approx \mathcal{N}(W^*, H^{-1})$ where $H = -\nabla^2 \log p(W|D)|_{W^*}$ (Fisher 정보), 현대 LA는 Kronecker-factored 근사
- **Variational BNN — Bayes by Backprop (Blundell et al. 2015)** — Factorized Gaussian $q_\phi(W) = \prod \mathcal{N}(\mu_i, \sigma_i^2)$, reparam + ELBO gradient, 분산 표현으로 uncertainty
- **MC Dropout = Approximate VI (Gal & Ghahramani 2016)** — Dropout with rate $p$가 Bernoulli variational posterior와 동치, test time에 dropout 유지하여 $T$번 샘플링 → Monte Carlo predictive, 실전 BNN
- **SWAG와 SGD의 Bayesian 해석** — SGD 궤적에서 mean + covariance 추정, Stochastic Weight Averaging-Gaussian, implicit Bayesian의 증거

### Chapter 6: Bayesian Optimization (4개 문서)
- **GP 기반 BO 프레임워크** — Black-box $f(x)$ 최적화, GP surrogate로 $f$ 모델링, acquisition function으로 다음 탐색점 결정, GP prior의 역할
- **Acquisition Function들** — Expected Improvement (EI), Upper Confidence Bound (UCB), Thompson Sampling (TS), Probability of Improvement, 각각의 exploration-exploitation 균형
- **BO의 수렴 분석** — GP-UCB의 sublinear regret $O(\sqrt{T \gamma_T})$ (Srinivas et al. 2010), $\gamma_T$는 maximum information gain, kernel별 차이
- **BO의 실전 확장** — High-dimensional BO, Multi-fidelity, Bayesian Quadrature, BoTorch 같은 현대 라이브러리, hyperparameter tuning에서의 활용

### Chapter 7: 고급 주제 — Deep Generative, Probabilistic Programming (4개 문서)
- **Diffusion Model의 Bayesian 해석** — Forward process = sequentially applying Gaussian noise, reverse = posterior denoising, ELBO가 DDPM 손실과 동치 (SDE 레포와 교차)
- **Probabilistic Programming 언어** — Stan, PyMC, NumPyro, Pyro 비교, posterior 추론의 추상화, automatic differentiation VI
- **Bayesian Deep Learning의 불확실성 분해** — Epistemic uncertainty (모델 불확실성, 데이터로 감소) vs Aleatoric uncertainty (관측 노이즈, 감소 불가), 각각의 수학적 정의와 실전 분리
- **Out-of-Distribution Detection과 Calibration** — Bayesian predictive가 OOD에서 높은 uncertainty를 보이는 이유, ECE (Expected Calibration Error)와 temperature scaling, Bayesian이 왜 calibrated인가

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **35개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 Bayesian 접근이 이 문제에 필요한가
## 📐 수학적 선행 조건 (Prob, Stats, Stoch, Info 레포 참조)
## 📖 직관적 이해
   — Prior/Likelihood/Posterior의 요리 비유
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — ELBO 분해, Reparam trick, MH detailed balance
## 💻 NumPy/PyMC 구현 검증
   — 바닥부터 VI, MCMC 구현 → PyMC/Pyro와 결과 비교
## 🔗 실전 활용
   — 언제 Bayesian, 언제 frequentist?
## ⚖️ 가정과 한계
   — Prior 선택 감도, multimodal posterior, 고차원 문제
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **세 가지 패러다임 병렬 제시** — 모든 주요 문제에서 MLE/MAP/Bayesian의 세 접근 비교
2. **Uncertainty 시각화 필수** — 1D/2D regression에서 posterior predictive distribution plot
3. **ELBO 최적화 궤적 plot** — 훈련 동안 reconstruction loss와 KL term 분리해서 추적
4. **MCMC trace plot과 진단** — $\hat{R}$, ESS, autocorrelation을 매번 확인하는 체크리스트
5. **Conjugate 분석과 근사 추론 비교** — 가능한 경우 closed-form + VI/MCMC 근사 결과 나란히
6. **실제 데이터셋 Bayesian 분석** — coin flip, MNIST(VAE), housing price (BLR) 등

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
matplotlib==3.8.0
pymc==5.10.0          # MCMC 표준
arviz==0.17.0         # Bayesian 진단
torch==2.1.0          # VAE, BNN
pyro-ppl==1.8.6       # PPL
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Beta-Binomial Bayesian 추론 + VI + MCMC 비교)
import numpy as np
import matplotlib.pyplot as plt
from scipy import stats

# 데이터: 10번 던져 7번 앞면
n_heads, n_flips = 7, 10

# 1. Conjugate posterior — Beta(α+k, β+n-k)
alpha, beta_ = 2, 2  # Beta(2,2) prior
posterior_exact = stats.beta(alpha + n_heads, beta_ + n_flips - n_heads)

# 2. Variational Inference — Mean-field Gaussian in logit space
# q(θ) ≈ logit^{-1}(N(μ, σ²))
# 수치적으로 ELBO 최대화
from scipy.optimize import minimize
def neg_elbo(params, n_mc=1000):
    mu, log_sigma = params
    sigma = np.exp(log_sigma)
    eps = np.random.randn(n_mc)
    theta = 1 / (1 + np.exp(-(mu + sigma * eps)))  # reparameterization
    log_lik = n_heads * np.log(theta) + (n_flips - n_heads) * np.log(1 - theta)
    log_prior = (alpha-1) * np.log(theta) + (beta_-1) * np.log(1-theta)
    log_q = -0.5 * eps**2 - log_sigma  # Gaussian entropy
    return -(log_lik + log_prior - log_q).mean()

result = minimize(neg_elbo, [0.0, 0.0], method='Nelder-Mead')
mu_vi, log_sigma_vi = result.x

# 3. MCMC — Metropolis-Hastings
def log_posterior(theta):
    if theta <= 0 or theta >= 1: return -np.inf
    return n_heads*np.log(theta) + (n_flips-n_heads)*np.log(1-theta) \
         + (alpha-1)*np.log(theta) + (beta_-1)*np.log(1-theta)

theta_chain = []
theta = 0.5
for _ in range(20000):
    theta_prop = np.clip(theta + 0.1*np.random.randn(), 1e-6, 1-1e-6)
    if np.log(np.random.rand()) < log_posterior(theta_prop) - log_posterior(theta):
        theta = theta_prop
    theta_chain.append(theta)
theta_chain = np.array(theta_chain[2000:])  # burn-in

# 시각화: 3가지 방법 비교
fig, ax = plt.subplots(figsize=(10, 5))
x = np.linspace(0, 1, 200)
ax.plot(x, posterior_exact.pdf(x), 'k-', lw=2, label='Exact (Beta posterior)')
# VI density via samples
eps = np.random.randn(10000)
samples_vi = 1/(1+np.exp(-(mu_vi + np.exp(log_sigma_vi)*eps)))
ax.hist(samples_vi, bins=50, density=True, alpha=0.3, label='VI (Mean-field)')
ax.hist(theta_chain, bins=50, density=True, alpha=0.3, label='MCMC (MH)')
ax.set_xlabel('θ'); ax.set_ylabel('posterior density')
ax.legend(); plt.show()
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Prob, Stats, Stoch, Info 선행 필수" 명시
   - Conjugate → VI → MCMC → Bayesian DL의 흐름
   - Diffusion Model(SDE 레포), Regularization Theory(L2=Gaussian prior)와의 연결
3. **챕터별 문서 작성**: Bayes 기초 → VI → VAE → MCMC → BNN → BO → 고급

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Bayesian Data Analysis** (Gelman, Carlin, Stern, Dunson, Vehtari, Rubin) — "BDA3", Bayesian 바이블
- **Pattern Recognition and Machine Learning** (Bishop) — Chap 10 VI 표준
- **Machine Learning: A Probabilistic Perspective** (Murphy) — Chap 19-24 Bayesian
- **Probabilistic Machine Learning: Advanced Topics** (Murphy 2023) — 최신
- **Information Theory, Inference, and Learning Algorithms** (MacKay) — 직관적
- **Auto-Encoding Variational Bayes** (Kingma & Welling 2013) — VAE 원전
- **Bayes by Backprop: Weight Uncertainty in Neural Networks** (Blundell et al. 2015)
- **Dropout as a Bayesian Approximation** (Gal & Ghahramani 2016) — MC Dropout 원전
- **Practical Bayesian Optimization of Machine Learning Algorithms** (Snoek et al. 2012)

---

## 💡 핵심 분석 대상

```
Bayesian ML의 4대 기둥

   Bayes' Rule
   ────────────────────────
   p(θ|D) = p(D|θ) p(θ) / p(D)
           ↑        ↑     ↑       ↑
     posterior  likelihood prior  evidence
                                   ↑
                         ∫p(D|θ)p(θ)dθ
                     (계산 불가능한 경우)
                           │
          ┌────────────────┼─────────────────┐
          ▼                ▼                 ▼
   Conjugate 관계      Variational         MCMC
   (closed form)       Inference      (샘플링)
   
   ───── Conjugate ─────
   Beta-Bernoulli, Gamma-Poisson, Normal-Normal, etc.
   → 정확한 해, 교육·prototype
   
   ───── Variational Inference ─────
   
   Goal: min_q KL(q(θ) ‖ p(θ|D))
   ⟺ max_q ELBO
   
   ELBO의 3가지 표현:
   ┌─────────────────────────────────────────────┐
   │ L(q) = log p(D) - KL(q‖p(·|D))              │
   │      = 𝔼_q[log p(D,θ)] - 𝔼_q[log q(θ)]      │
   │      = 𝔼_q[log p(D|θ)] - KL(q(θ)‖p(θ))      │
   │         └── reconstruction ──┘ └─ prior reg ┘│
   └─────────────────────────────────────────────┘
   
   Mean-field: q(θ) = ∏ q_i(θ_i)
     → CAVI: q_i* ∝ exp 𝔼_{-i}[log p(D, θ)]
   
   Reparameterization:
     z ~ q_φ(z|x)  ≡  z = g_φ(ε, x), ε ~ p(ε)
     ∇_φ 𝔼_q[f(z)] = 𝔼_p(ε)[∇_φ f(g_φ(ε, x))]
                      ↑ low-variance gradient
   
   VAE: Encoder q_φ(z|x) + Decoder p_θ(x|z)
     ELBO = 𝔼_q[log p(x|z)] - KL(q(z|x)‖p(z))
          └── reconstruction ─┘└── regularization ┘
   
   ───── MCMC ─────
   
   목표: p(θ|D)에서 샘플 뽑기
   
   Metropolis-Hastings:
     propose θ' ~ q(θ'|θ)
     accept with α = min(1, p(θ'|D)q(θ|θ') / p(θ|D)q(θ'|θ))
     → Detailed balance ⇒ p(·|D)가 정상분포
   
   Gibbs (proposal이 조건부):
     θ_i ← sample p(θ_i | θ_{-i}, D)
   
   HMC (gradient 활용):
     Hamiltonian H(θ,p) = -log p(θ|D) + ½pᵀM⁻¹p
     Leapfrog 적분 → 큰 step, 고차원 효율
   
   NUTS: HMC의 자동 튜닝
   
   진단:
     ├── R̂ (Gelman-Rubin): ≈ 1.0 목표
     ├── ESS (Effective Sample Size): n_eff / n
     └── Trace plot: mixing 시각 확인

   ───── Bayesian Deep Learning ─────
   
   W ~ p(W) | y|x,W ~ L(...) | posterior p(W|D) | p(y*|x*, D)
   
   ├── Laplace: p(W|D) ≈ N(W*, H⁻¹)
   │   H = Hessian at MAP (= Fisher 정보)
   │
   ├── Variational BNN (Bayes by Backprop):
   │   q(W) = ∏ N(μ_i, σ_i²) + reparam + ELBO
   │
   ├── MC Dropout (Gal 2016):
   │   Dropout p = Bernoulli variational posterior
   │   Test time dropout T번 → MC predictive
   │
   ├── SWAG: SGD iterates로 Gaussian posterior 근사
   │
   └── SGLD: SGD + Gaussian noise
       = Langevin SDE의 이산화
       → stationary = posterior
   
   ───── Uncertainty 분해 ─────
   
   𝔼[y|x*] posterior predictive variance =
     𝔼_W[Var(y|x*, W)]  ←  Aleatoric (데이터 노이즈)
     + Var_W[𝔼(y|x*, W)] ←  Epistemic (모델 불확실성)
                            ↑ 데이터 많아지면 감소
   
   Calibration: confidence = accuracy?
     ECE = |confidence - accuracy|
     Bayesian: 자동으로 calibrated (Bernstein-von Mises)
   
   ───── Bayesian Optimization ─────
   
   Black-box f(x) 최적화:
     Surrogate: GP(f)
     Acquisition a(x):
       ├── EI:  𝔼[max(0, f(x) - f_best)]
       ├── UCB: μ(x) + κ σ(x)
       ├── TS:  sample f̃ ~ GP, maximize f̃
       └── PI:  P(f(x) > f_best)
     Next x* = argmax a(x)
   
   GP-UCB regret: O(√(T γ_T))
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (35개 목표)
- Python + NumPy + PyMC + PyTorch 실험 환경
- Prob, Stats, Stoch, Info, Calc, Kernel 레포의 어떤 정리를 전제로 사용하는지
- Diffusion (SDE), Regularization Theory, Neural Net Theory와의 연결

**준비됐으면 1단계 구조 설계부터 시작해줘!**
