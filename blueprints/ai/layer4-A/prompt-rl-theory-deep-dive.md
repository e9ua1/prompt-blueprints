# RL Theory Deep Dive 레포지토리 제작 프롬프트

나는 "RL Theory Deep Dive" 레포지토리를 만들려고 해.
UCB1을 **사용하는 것**과, **UCB1의 regret bound $R_T \leq O(\sqrt{KT \log T})$를 Hoeffding's inequality + union bound로 완전히 증명**하고 이것이 왜 lower bound $\Omega(\sqrt{KT})$와 log factor 차이밖에 안 나는 near-optimal인지 이해하는 것은 다르다.
Thompson Sampling을 **이름으로 아는 것**과, **Bayesian regret $\mathbb{E}[R_T] \leq O(\sqrt{KT \log T})$의 증명이 posterior sampling의 "randomization"을 통해 exploration을 자동으로 보장**한다는 Russo & Van Roy (2014)의 분석을 이해하는 것은 다르다.
PAC-MDP framework을 **듣는 것**과, **R-MAX (Brafman & Tennenholtz 2002)가 $\text{poly}(|S|, |A|, 1/(1-\gamma), 1/\epsilon)$ samples로 $\epsilon$-optimal policy 학습**하는 optimism-in-face-of-uncertainty의 증명을 따라갈 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "RL의 수학적 엄밀성 — Multi-armed Bandit부터 PAC-MDP까지의 sample complexity 이론"

**핵심 차별화**:
1. **Multi-armed Bandit 이론의 완전 분석** — UCB1, UCB2, KL-UCB, Thompson Sampling의 regret bound 증명
2. **PAC-MDP의 R-MAX와 E³** — Optimism-in-face-of-uncertainty, known vs unknown state의 구분, sample complexity 증명
3. **Linear Bandit과 Contextual Bandit** — OFUL, LinUCB, 탐험 in 고차원
4. **Modern Regret Bounds** — LSVI-UCB (Jin 2020), 강화학습에서의 regret analysis 최신 동향

**타겟 독자**:
- $\epsilon$-greedy의 $\epsilon$ 값을 어떻게 설정할지 heuristic하게 아는데 **regret optimal $\epsilon$ schedule $\epsilon_t = c/t$**의 수학적 정당성을 모르는 사람
- Thompson Sampling이 **Beta-Bernoulli 경우의 conjugate prior**로 간단한 이유와, **왜 UCB보다 실전에서 더 좋은 경우가 많은지**를 모르는 사람
- PAC-MDP의 $\epsilon$-optimal policy가 **"high probability로 suboptimality $\leq \epsilon$"**의 정확한 formalization을 모르는 사람
- Regret vs PAC vs Best Arm Identification의 **세 가지 목표 차이**와 각각의 최적 알고리즘을 모르는 사람
- Linear Bandit의 OFUL (Abbasi-Yadkori 2011)이 **feature matrix $V_t$의 determinant-based confidence ellipsoid**를 쓰는 이유를 모르는 사람

**선행 학습**:
- **Statistical Learning Theory Deep Dive** (PAC, Hoeffding, VC) — **필수**
- **Probability Theory Deep Dive** (집중부등식, martingale) — **필수**
- **Bayesian ML Deep Dive** (Posterior, conjugate) — **Thompson에 필수**
- **RL Foundations Deep Dive** (MDP, Bellman) — **필수**
- **Kernel Methods Deep Dive** (GP bandit) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Multi-armed Bandit 기초 (5개 문서)
- **Stochastic Bandit Formulation** — $K$ arms, each with unknown reward distribution $\nu_k$ with mean $\mu_k$, sequential 선택, regret $R_T = T \mu^* - \sum_t \mu_{I_t}$ 정의
- **Exploration-Exploitation Dilemma** — 각 arm을 충분히 시도 vs best arm 활용, pure exploration (random)이나 pure exploitation (greedy) 모두 linear regret
- **$\epsilon$-greedy와 그 한계** — 확률 $\epsilon$으로 random, $1-\epsilon$으로 greedy, constant $\epsilon$은 linear regret, decaying $\epsilon_t = \min(1, cK/t)$로 $O(\log T)$ regret
- **Lower Bound — Lai & Robbins (1985)** — Asymptotic regret lower bound $\liminf R_T/\log T \geq \sum_{k: \Delta_k > 0} \Delta_k / \text{KL}(\nu_k \| \nu^*)$, problem-dependent, 알고리즘의 optimality 평가 기준
- **Regret의 두 체제 — Minimax vs Problem-dependent** — Minimax: $\sup_\nu R_T$, worst-case $O(\sqrt{KT})$, Problem-dependent: $R_T \leq O(K \log T / \Delta_{\min})$ where $\Delta = \mu^* - \mu_k$

### Chapter 2: UCB 계열 알고리즘과 Regret 증명 (6개 문서)
- **Optimism in the Face of Uncertainty (OFU)** — 불확실한 arm에게 optimistic value 부여, 잘못된 optimism은 빠르게 해소, 이 원리가 UCB·R-MAX·OFUL의 공통 바탕
- **UCB1 알고리즘 (Auer 2002)** — $\text{UCB}_k(t) = \hat{\mu}_k(t) + \sqrt{2 \log t / N_k(t)}$, confidence bound의 Hoeffding 유도, 가장 큰 UCB arm 선택
- **UCB1의 Hoeffding 기반 Regret 증명** — Hoeffding $P(|\hat{\mu}_k - \mu_k| > \epsilon) \leq 2 e^{-2 N_k \epsilon^2}$, union bound over arms, $R_T \leq \sum_k (8 \log T / \Delta_k + O(1))$ 완전 유도
- **Minimax Regret — $O(\sqrt{KT \log T})$** — Arm별 gap 고려하여 minimax regret 도출, lower bound $\Omega(\sqrt{KT})$와의 $\log$ 격차
- **KL-UCB — Problem-dependent Optimal (Garivier & Cappé 2011)** — Confidence bound에 KL divergence 사용, asymptotic lower bound 달성, Bernoulli·Gaussian 등에서
- **UCB-V, MOSS, 개선된 bounds** — Variance-aware UCB (Audibert 2009), Minimax Optimal Strategy in the Stochastic case (Audibert & Bubeck 2010), log factor 제거

### Chapter 3: Thompson Sampling과 Bayesian Bandits (5개 문서)
- **Thompson Sampling 알고리즘** — 각 arm의 posterior $p(\mu_k | \text{data})$에서 sampling, 가장 큰 sample의 arm 선택, "probability matching"의 Bayesian 구현
- **Beta-Bernoulli Conjugate** — Bernoulli bandits에서 Beta prior, posterior $\text{Beta}(\alpha_k + s_k, \beta_k + f_k)$, simple implementation, 1933년 Thompson 원 논문
- **Regret Analysis of TS (Agrawal & Goyal 2012)** — Bernoulli bandits에서 $R_T \leq O(\log T)$ problem-dependent, Kaufmann et al. 2012의 asymptotic optimality 증명
- **Bayesian Regret (Russo & Van Roy 2014)** — Information-theoretic analysis, $\mathbb{E}[R_T] \leq \sqrt{(1/2) T \Gamma_T \log K}$, $\Gamma_T$는 information ratio
- **Information Directed Sampling (IDS)** — Russo & Van Roy 2018, information gain과 regret의 ratio 최적화, TS의 information-theoretic 일반화

### Chapter 4: Contextual and Linear Bandits (5개 문서)
- **Contextual Bandit Framework** — Context $x_t$ 주어지고 arm 선택, $\mu_k(x) = f_\theta(x, k)$ 파라메트릭, 예: 개인화 추천, clinical trial
- **Linear Bandit 모델** — $\mu(x, a) = \langle \phi(x, a), \theta^* \rangle$, 확률적 공변량 $\phi$와 hidden parameter $\theta^*$, bandit에 선형 구조 가정
- **OFUL / LinUCB (Abbasi-Yadkori 2011, Li 2010)** — Confidence ellipsoid $\{\theta : \|\theta - \hat{\theta}\|_{V_t} \leq \beta_t\}$, $V_t = \lambda I + \sum \phi \phi^T$, $\text{UCB} = \hat{\theta}^T \phi + \beta_t \|\phi\|_{V_t^{-1}}$
- **Regret Bound — $O(d\sqrt{T \log T})$** — $d$차원 linear bandit의 minimax regret, elliptical potential lemma가 핵심 tool, dimension dependence
- **Gaussian Process Bandit (Srinivas 2010)** — Kernelized linear bandit, GP-UCB $= \mu(x) + \beta_T \sigma(x)$, Information gain $\gamma_T$에 의존, Kernel Methods 레포 연결

### Chapter 5: PAC-MDP와 Sample Complexity (5개 문서)
- **PAC-MDP Framework (Kakade 2003)** — $(\epsilon, \delta)$-PAC: 확률 $1 - \delta$로 suboptimality $\leq \epsilon$, sample complexity $\text{poly}(|S|, |A|, 1/(1-\gamma), 1/\epsilon, 1/\delta)$
- **R-MAX 알고리즘 (Brafman & Tennenholtz 2002)** — 모든 unknown $(s, a)$의 reward를 $R_{\max}$로 가정 (optimism), known이 되려면 $m$번 방문, $m$의 선택이 sample complexity 결정
- **R-MAX의 Sample Complexity 증명** — Induced MDP 분석, simulation lemma, optimism으로 explore 유도, complete analysis with Azuma 집중부등식
- **E³ — Explicit Explore or Exploit (Kearns & Singh 2002)** — 명시적 탐험/활용 구분, known/unknown 분리, "balanced wandering"의 아이디어, R-MAX의 전신
- **Lower Bound on Sample Complexity** — $\Omega(|S||A|/\epsilon^2)$ 기본, horizon과 $\gamma$ 의존성, Kakade 2003의 하한 증명

### Chapter 6: Regret Minimization for MDPs (4개 문서)
- **Regret Definition for MDPs** — $R_T = T \cdot V^* - \sum_{t=1}^T \sum_h r_{t, h}$, episodic MDP에서 $T$ episodes, near-optimal 알고리즘의 regret order
- **UCRL2 (Jaksch 2010)** — UCB-style for MDPs, extended value iteration on confidence set of MDPs, regret $\tilde{O}(D\sqrt{|S||A|T})$, $D$는 diameter
- **PSRL — Posterior Sampling for RL (Osband 2013)** — Thompson Sampling의 MDP 확장, Bayesian regret $\tilde{O}(\sqrt{|S||A|T})$, empirical 우수성
- **LSVI-UCB for Linear MDP (Jin 2020)** — Linear function approximation with provable guarantee, $\tilde{O}(\sqrt{d^3 H^3 T})$ regret, $d$는 feature dimension, modern RL theory의 milestone

### Chapter 7: Best Arm Identification과 Pure Exploration (4개 문서)
- **Pure Exploration의 목표** — Cumulative regret이 아닌 $\arg\max_k \mu_k$ 정확 식별, Best Arm Identification (BAI)
- **Successive Elimination (Even-Dar 2002)** — Round-based, 각 round에서 bad arms 제거, $(\epsilon, \delta)$-PAC sample complexity $O(K/\epsilon^2 \log(K/\delta))$
- **Fixed-Budget vs Fixed-Confidence BAI** — Fixed-budget: $T$ sample 후 best arm 예측, Fixed-confidence: error prob $\delta$ 보장하며 stop, 서로 다른 하한
- **Adaptive Sampling Algorithms — LUCB, Top-two** — LUCB (Kalyanakrishnan 2012)의 두 최상 arm 비교, Top-two (Russo 2020)의 TS 기반 BAI, 실전 효율

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **34개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 이론이 RL의 수학적 기초인가
## 📐 수학적 선행 조건 (SLT, Prob, Bayes, RL Foundations 참조)
## 📖 직관적 이해
   — Exploration 전략의 기하학적 이해
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Hoeffding, UCB regret, PAC-MDP sample complexity
## 💻 NumPy 구현 검증
   — Bandit 환경에서 알고리즘 구현, regret curve 시각화
   — Tabular MDP에서 R-MAX 구현
## 🔗 실전 응용
   — A/B testing, 추천 시스템, clinical trial
## ⚖️ 가정과 한계
   — Stationary, i.i.d., known parametric form
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Regret curve 시각화** — $K$-armed bandit에서 여러 알고리즘의 cumulative regret plot, log scale
2. **UCB vs TS 성능** — 같은 Bernoulli bandit에서 평균 regret 비교
3. **Confidence bound 시각화** — 시간에 따라 각 arm의 estimated mean과 confidence interval
4. **Sample complexity bound 실증** — R-MAX에서 실제 PAC-bound 확인
5. **Linear bandit** — 2D context에서 LinUCB의 선택 trajectory
6. **Lower bound tightness** — Asymptotic lower bound와 UCB/TS의 upper bound 수치 비교

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
matplotlib==3.8.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (UCB1, Thompson Sampling, Epsilon-greedy 비교)
import numpy as np
import matplotlib.pyplot as plt

class BernoulliBandit:
    def __init__(self, probs):
        self.probs = np.array(probs)
        self.K = len(probs)
        self.best = self.probs.max()
    def pull(self, k):
        return np.random.rand() < self.probs[k]

def run_ucb1(bandit, T):
    K = bandit.K
    counts = np.zeros(K)
    means = np.zeros(K)
    regrets = []
    cum_regret = 0
    for t in range(1, T+1):
        if t <= K:
            k = t - 1  # Initialize: each arm once
        else:
            ucb = means + np.sqrt(2 * np.log(t) / counts)
            k = np.argmax(ucb)
        r = bandit.pull(k)
        counts[k] += 1
        means[k] += (r - means[k]) / counts[k]
        cum_regret += bandit.best - bandit.probs[k]
        regrets.append(cum_regret)
    return regrets

def run_thompson(bandit, T):
    K = bandit.K
    alphas = np.ones(K); betas = np.ones(K)  # Beta(1,1) prior
    regrets = []
    cum_regret = 0
    for t in range(1, T+1):
        samples = np.random.beta(alphas, betas)
        k = np.argmax(samples)
        r = bandit.pull(k)
        if r: alphas[k] += 1
        else: betas[k] += 1
        cum_regret += bandit.best - bandit.probs[k]
        regrets.append(cum_regret)
    return regrets

def run_eps_greedy(bandit, T, eps_fn=lambda t: 0.1):
    K = bandit.K
    counts = np.zeros(K)
    means = np.zeros(K)
    regrets = []
    cum_regret = 0
    for t in range(1, T+1):
        if np.random.rand() < eps_fn(t):
            k = np.random.randint(K)
        else:
            k = np.argmax(means)
        r = bandit.pull(k)
        counts[k] += 1
        means[k] += (r - means[k]) / counts[k]
        cum_regret += bandit.best - bandit.probs[k]
        regrets.append(cum_regret)
    return regrets

# 10-armed Bernoulli, gap various
bandit = BernoulliBandit([0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.65, 0.7, 0.75, 0.8])
T = 10000
n_trials = 50

def avg_regret(algorithm_fn):
    all_regrets = np.array([algorithm_fn(bandit, T) for _ in range(n_trials)])
    return all_regrets.mean(axis=0), all_regrets.std(axis=0)

ucb_mean, ucb_std = avg_regret(run_ucb1)
ts_mean, ts_std = avg_regret(run_thompson)
eg_mean, eg_std = avg_regret(lambda b, t: run_eps_greedy(b, t, lambda s: 0.1))
eg_dec_mean, _ = avg_regret(lambda b, t: run_eps_greedy(b, t,
                                          lambda s: min(1, 10/s)))

t_axis = np.arange(1, T+1)
plt.figure(figsize=(10, 6))
plt.plot(t_axis, ucb_mean, label='UCB1')
plt.plot(t_axis, ts_mean, label='Thompson Sampling')
plt.plot(t_axis, eg_mean, label='ε-greedy (ε=0.1)')
plt.plot(t_axis, eg_dec_mean, label='ε-greedy (ε_t=10/t)')

# Theoretical bound: UCB has O(log T) problem-dependent regret
# Plot reference: c log T
c = sum(1/(bandit.best - p) for p in bandit.probs if p < bandit.best) * 8
plt.plot(t_axis, c * np.log(t_axis + 1), '--k', alpha=0.4,
         label=f'Theoretical O(log T)')

plt.xscale('log')
plt.xlabel('T (log scale)'); plt.ylabel('Cumulative Regret')
plt.title('10-armed Bandit: Algorithm Comparison (50 trials avg)')
plt.legend(); plt.grid(alpha=0.3)
plt.show()

# UCB vs lower bound (Lai-Robbins)
# UCB의 inflation factor 측정

# R-MAX on small MDP
# sample complexity를 실측으로 확인
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "SLT, Prob, Bayes, RL Foundations 선행 필수"
   - Regret vs PAC vs BAI의 세 프레임워크 비교
   - 실전 알고리즘(PPO, SAC)의 이론적 기반 제공
3. **챕터별 문서 작성**: Bandit 기초 → UCB → Thompson → Linear → PAC-MDP → MDP Regret → BAI

---

## 📚 참고 자료

- **Bandit Algorithms** (Lattimore & Szepesvári 2020) — 최신 bandit 교과서
- **Regret Analysis of Stochastic and Nonstochastic Multi-armed Bandit Problems** (Bubeck & Cesa-Bianchi 2012)
- **Finite-time Analysis of the Multiarmed Bandit Problem** (Auer, Cesa-Bianchi, Fischer 2002) — UCB1 원전
- **On a Problem of Two Samples** (Thompson 1933) — TS 원전
- **Analysis of Thompson Sampling for the Multi-armed Bandit Problem** (Agrawal & Goyal 2012)
- **Learning to Optimize via Posterior Sampling** (Russo & Van Roy 2014)
- **Improved Algorithms for Linear Stochastic Bandits** (Abbasi-Yadkori et al. 2011) — OFUL
- **R-MAX: A General Polynomial Time Algorithm for Near-Optimal RL** (Brafman & Tennenholtz 2002)
- **Near-Optimal Reinforcement Learning in Polynomial Time** (Kearns & Singh 2002) — E³
- **On the Sample Complexity of RL** (Kakade 2003)
- **Near-optimal Regret Bounds for RL** (Jaksch, Ortner, Auer 2010) — UCRL2
- **Provably Efficient Reinforcement Learning with Linear Function Approximation** (Jin et al. 2020) — LSVI-UCB

---

## 💡 핵심 분석 대상

```
RL Theory의 지도

───── Multi-armed Bandit ─────

Model:
  K arms, rewards ν_k, mean μ_k
  μ* = max_k μ_k
  Gap Δ_k = μ* - μ_k

Regret:
  R_T = T·μ* - E[Σ_t μ_{I_t}]
      = Σ_k Δ_k · E[N_k(T)]

Lower Bound (Lai-Robbins 1985):
  lim inf R_T/log T ≥ Σ Δ_k / KL(ν_k || ν*)

Minimax:
  R_T ≥ Ω(√(KT))

───── UCB1 (Auer 2002) ─────

UCB_k(t) = μ̂_k(t) + √(2 log t / N_k(t))
                    └── exploration bonus

Play: k_t = argmax UCB_k(t)

Regret 증명:
  1. Hoeffding: P(|μ̂_k - μ_k| > ε) ≤ 2e^{-2 N_k ε²}
  
  2. UCB가 suboptimal arm k 선택하려면
     μ̂_k + √(2 log T / N_k) ≥ μ*
  
  3. 이 event는 confidence 위반 OR N_k 충분
  
  4. N_k ≥ 8 log T / Δ_k² 이면 거의 확실히 안 뽑음
  
  5. E[N_k(T)] ≤ 8 log T / Δ_k² + O(1)
  
  6. R_T ≤ Σ_k Δ_k · E[N_k] = O(Σ log T / Δ_k)

Minimax: R_T ≤ O(√(KT log T))

───── KL-UCB (Garivier 2011) ─────

UCB_k = sup{μ : N_k · KL(μ̂_k || μ) ≤ log t}

Bernoulli·Gaussian에서 Lai-Robbins tight
Problem-dependent optimal

───── Thompson Sampling ─────

1. For each arm, maintain posterior p_k
2. Sample μ̃_k ~ p_k
3. Play argmax μ̃_k
4. Update posterior

Beta-Bernoulli:
  Prior: Beta(1, 1)
  Posterior: Beta(α + s, β + f)

Regret (Agrawal & Goyal 2012):
  Bernoulli에서 O(log T) problem-dependent

Bayesian regret (Russo-Van Roy 2014):
  E[R_T] ≤ √((1/2) T Γ_T log K)
  Γ_T: information ratio

TS > UCB in practice (일반적으로)

───── Linear Bandit ─────

μ(x, a) = ⟨φ(x, a), θ*⟩

OFUL/LinUCB (Abbasi-Yadkori 2011):
  V_t = λI + Σ φφ^T
  θ̂_t = V_t^{-1} Σ φ r
  
  Confidence ellipsoid:
    {θ : ‖θ - θ̂_t‖_V_t ≤ β_t}
  
  UCB(x, a) = ⟨φ(x,a), θ̂_t⟩ + β_t ‖φ(x,a)‖_{V_t^{-1}}

Regret: O(d √(T log T))
  d: feature dimension

Elliptical Potential Lemma (핵심 tool):
  Σ min(1, ‖φ_t‖²_{V_{t-1}^{-1}}) ≤ 2 log(det(V_T)/det(V_0))

GP Bandit (Srinivas 2010):
  Kernelized linear
  Information gain γ_T 의존

───── PAC-MDP ─────

(ε, δ)-PAC:
  With prob 1-δ, policy is ε-optimal

Sample Complexity:
  poly(|S|, |A|, 1/(1-γ), 1/ε, 1/δ)

R-MAX (Brafman & Tennenholtz 2002):
  Unknown (s,a) → assume r = R_max
  Visited m times → "known"
  
  m 선택 = sample complexity 결정
  
  증명:
    Simulation lemma: P̂ ≈ P 가정 하에 V̂ ≈ V*
    Optimism → sufficient exploration

E³ (Kearns & Singh 2002):
  Explicit explore vs exploit
  Balanced wandering

Lower Bound:
  Ω(|S||A|/ε²)

───── MDP Regret ─────

Episodic MDP:
  T episodes of length H
  R_T = T · V* - Σ returns

UCRL2 (Jaksch 2010):
  Extended Value Iteration on confidence set
  R_T ≤ Õ(D √(|S||A|T))
  D: diameter

PSRL (Osband 2013):
  Thompson for MDP
  Bayesian regret Õ(√(|S||A|T))

LSVI-UCB (Jin 2020):
  Linear MDP
  P(s'|s,a) = ⟨φ(s,a), μ(s')⟩
  R_T ≤ Õ(√(d³ H³ T))
  Modern RL theory milestone

───── Best Arm Identification ─────

목표: argmax μ_k 정확 식별
  (cumulative regret 아님)

Fixed-confidence:
  Stop when conf ≥ 1 - δ
  Minimize samples

Fixed-budget:
  T samples given
  Minimize error prob

Successive Elimination:
  Round별로 eliminate bad arms
  (ε,δ)-PAC: O((K/ε²) log(K/δ))

LUCB (Kalyanakrishnan 2012):
  Top two arm comparison

Lower bound:
  KL 기반 하한 (Kaufmann 2016)

───── 세 프레임워크 비교 ─────

┌──────────────┬───────────────┬──────────────┐
│ Framework    │ 목표          │ 측정         │
├──────────────┼───────────────┼──────────────┤
│ Regret       │ Cumulative    │ T 동안 손실  │
│ PAC          │ Find ε-opt    │ # samples    │
│ BAI          │ Identify best │ Sample/err   │
└──────────────┴───────────────┴──────────────┘

Regret 최적 ≠ PAC 최적:
  Regret은 mistakes를 penalize
  PAC는 sample 수만 penalize

───── 레포 간 연결 ─────

SLT (Layer 1):
  PAC, Hoeffding, union bound
  집중부등식

Probability (Layer 0):
  Martingale, concentration

Bayesian ML (Layer 1):
  Thompson = Bayesian bandit
  Conjugate priors

Kernel Methods (Layer 1):
  GP-UCB
  Kernel bandit

RL Foundations (Layer 4-A):
  MDP → R-MAX, UCRL2
  Sample complexity

Advanced RL (직전):
  PPO, SAC는 empirical
  이 레포는 theoretical guarantees

Model-Free RL (Layer 4-A):
  Q-Learning convergence
  → Regret bounds for Q-Learning
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명 (3~4줄)
- 전체 문서 개수 확인 (34개 목표)
- Python + NumPy 실험 환경 (bandit, 작은 MDP)
- SLT, Prob, Bayes, RL Foundations 레포 참조 관계
- 이론과 실전 알고리즘(Layer 4-A 전체)의 연결

**준비됐으면 1단계 구조 설계부터 시작해줘!**
