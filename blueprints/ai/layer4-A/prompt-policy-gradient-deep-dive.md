# Policy Gradient Deep Dive 레포지토리 제작 프롬프트

나는 "Policy Gradient Deep Dive" 레포지토리를 만들려고 해.
REINFORCE를 **구현하는 것**과, **Log-Derivative Trick $\nabla_\theta \mathbb{E}_{x \sim p_\theta}[f(x)] = \mathbb{E}_{x \sim p_\theta}[f(x) \nabla_\theta \log p_\theta(x)]$이 score function estimator임을 유도**하고 이것이 value-based RL과 근본적으로 다른 접근인 이유를 증명할 수 있는 것은 다르다.
Policy Gradient Theorem (Sutton 1999)을 **인용하는 것**과, **$\nabla_\theta J(\theta) = \mathbb{E}_{s \sim d^\pi, a \sim \pi}[\nabla_\theta \log \pi(a|s) Q^\pi(s, a)]$** 를 performance difference lemma에서 완전히 유도하고, 왜 discounted state distribution $d^\pi$가 등장하는지 증명할 수 있는 것은 다르다.
GAE의 $\lambda$ 파라미터를 **조정하는 것**과, **$\hat{A}_t^{\text{GAE}(\gamma, \lambda)} = \sum_{l=0}^\infty (\gamma \lambda)^l \delta_{t+l}^V$가 $\lambda = 0$일 때 1-step TD advantage, $\lambda = 1$일 때 full Monte Carlo advantage**인 bias-variance trade-off를 유도할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "Policy를 직접 최적화 — Log-Derivative Trick부터 Natural Gradient까지"

**핵심 차별화**:
1. **Log-Derivative Trick의 완전 유도** — Score function estimator, likelihood ratio method, reparam trick과의 차이
2. **Policy Gradient Theorem의 performance difference 유도** — $d^\pi$의 등장, $Q^\pi$와 $A^\pi$의 동등성
3. **Baseline의 Variance Reduction** — Optimal baseline, control variate 이론, state-dependent baseline의 unbiasedness
4. **GAE의 bias-variance 통제** — $\lambda$의 역할, TD(λ)와의 연결, PPO·TRPO의 기반

**타겟 독자**:
- REINFORCE의 $\nabla_\theta \log \pi(a|s)$ 부분을 쓰는데 **왜 이것이 expected reward gradient**인지 유도 못하는 사람
- Baseline을 빼는 효과를 **variance reduction**으로 아는데 **왜 unbiased인지 수학적 증명**을 못하는 사람
- GAE를 쓰지만 **$\lambda$가 0, 0.5, 1일 때 각각 어떤 advantage estimator**가 되는지 유도 못하는 사람
- Actor-Critic에서 Critic이 **왜 bootstrapping되면 bias가 생기고 이것이 수용 가능한 이유**를 모르는 사람
- Natural Policy Gradient (Kakade 2001)의 **Fisher Information Matrix 기반 업데이트**가 Information Geometry 레포의 어느 개념인지 연결 못하는 사람

**선행 학습**:
- **RL Foundations Deep Dive** (MDP, Performance Difference Lemma) — **필수**
- **Model-Free RL Deep Dive** (TD, Actor-Critic 기초) — **필수**
- **Mathematical Statistics Deep Dive** (MLE, Score function) — **필수**
- **Information Geometry Deep Dive** (Fisher, KL) — **NPG에 필수**
- **Optimization Theory Deep Dive** (Gradient, momentum) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Policy 기반 RL의 동기 (4개 문서)
- **Value-based vs Policy-based RL** — Q-Learning은 $Q^*$ 학습 후 greedy policy, Policy-based는 $\pi_\theta$ 직접 최적화, 각각의 장단점 (continuous action, stochastic policy, convergence)
- **Stochastic Policy의 필요성** — 일부 MDP(POMDP, game-theoretic)에서 optimal policy가 stochastic, e.g. Rock-Paper-Scissors, Aliased Gridworld
- **Policy Parameterization** — Discrete: softmax $\pi(a|s) = e^{h_\theta(s, a)}/\sum_{a'} e^{h_\theta(s, a')}$, Continuous: Gaussian $\pi(a|s) = \mathcal{N}(\mu_\theta(s), \sigma_\theta(s)^2)$, 각각의 sampling과 log-density
- **Policy Gradient 목적 함수** — $J(\theta) = \mathbb{E}_{\tau \sim \pi_\theta}[R(\tau)] = \mathbb{E}_{s_0 \sim \rho_0}[V^{\pi_\theta}(s_0)]$, 직접 gradient ascent로 최대화

### Chapter 2: Log-Derivative Trick과 REINFORCE (5개 문서)
- **Log-Derivative Trick의 유도** — $\nabla_\theta \mathbb{E}_{p_\theta}[f(x)] = \nabla_\theta \int p_\theta(x) f(x) dx = \int \nabla_\theta p_\theta(x) f(x) dx = \int p_\theta \nabla_\theta \log p_\theta \cdot f dx = \mathbb{E}[\nabla_\theta \log p_\theta(x) \cdot f(x)]$, 핵심은 $\nabla p = p \nabla \log p$
- **Score Function Estimator** — $\nabla \log p$를 score function이라 부름, score의 기댓값이 0 ($\mathbb{E}[\nabla \log p] = 0$) 증명, Fisher Information과의 관계 (Stats 레포 교차)
- **REINFORCE (Williams 1992)** — $\nabla_\theta J(\theta) = \mathbb{E}_{\tau}[\nabla_\theta \log p_\theta(\tau) \cdot R(\tau)] = \mathbb{E}_\tau[\sum_t \nabla_\theta \log \pi(a_t|s_t) \cdot G_t]$, episodic 업데이트
- **REINFORCE의 High Variance 문제** — Return $G_t$의 variance가 trajectory length에 따라 증가, sparse reward에서 특히 심각, convergence 느림
- **Reparameterization Trick과의 대비** — Continuous distribution에서 $x = g(\epsilon; \theta), \epsilon \sim p(\epsilon)$로 reparam, lower variance gradient, 그러나 discrete 불가, PG는 discrete도 가능

### Chapter 3: Policy Gradient Theorem (5개 문서)
- **PG Theorem의 서술 (Sutton 1999)** — $\nabla_\theta J(\theta) = \mathbb{E}_{s \sim d^\pi, a \sim \pi}[\nabla_\theta \log \pi(a|s) Q^\pi(s, a)]$, discounted state distribution $d^\pi(s)$와 $Q^\pi$의 등장
- **Performance Difference Lemma 기반 증명** — $J(\pi') - J(\pi) = (1/(1-\gamma)) \mathbb{E}_{s \sim d^{\pi'}, a \sim \pi'}[A^\pi(s, a)]$, $\pi' = \pi_{\theta + \epsilon}$로 놓고 $\epsilon$에 대한 derivative로 PG theorem 유도
- **Direct Proof via Unrolling** — Bellman equation으로 $V^\pi(s) = \sum_a \pi(a|s) Q^\pi(s, a)$에서 시작, 재귀적으로 $\nabla V^\pi(s_0)$를 전개, geometric sum이 $1/(1-\gamma)$ 및 $d^\pi$로 환원
- **$Q^\pi$를 $A^\pi$로 바꿀 수 있는 이유** — $\mathbb{E}_{a \sim \pi}[\nabla \log \pi(a|s) b(s)] = 0$ for any state-dependent $b$, 증명 ($\int \pi \nabla \log \pi \cdot b \, da = b \nabla \int \pi \, da = b \cdot 0$)
- **Stochastic vs Deterministic Policy Gradient (Silver 2014)** — Stochastic: integrate over actions, DPG: $\nabla J = \mathbb{E}[\nabla_\theta \mu_\theta(s) \cdot \nabla_a Q^\mu(s, a)|_{a = \mu_\theta(s)}]$, continuous action에서의 효율성

### Chapter 4: Baseline과 Variance Reduction (5개 문서)
- **Baseline Subtraction의 Unbiasedness** — $\mathbb{E}[\nabla \log \pi(a|s) (G_t - b(s))] = \mathbb{E}[\nabla \log \pi(a|s) G_t]$ 증명, baseline은 state-dependent만 가능 (action-dependent는 unbiased 깨짐)
- **Optimal Baseline** — Variance $\text{Var}[\nabla \log \pi \cdot (G - b)]$ 최소화, 해 $b^*(s) = \mathbb{E}[(\nabla \log \pi)^2 G] / \mathbb{E}[(\nabla \log \pi)^2]$, 실전에서는 $V^\pi(s)$로 근사
- **Advantage Actor-Critic (A2C/A3C)** — Critic이 $V^\pi(s)$ 학습, advantage $\hat{A}(s, a) = G_t - V^\pi(s)$ (MC) 또는 $r + \gamma V^\pi(s') - V^\pi(s)$ (TD), A3C는 asynchronous multi-worker (Mnih 2016)
- **Control Variate 이론** — Monte Carlo estimation의 variance reduction 기법, baseline = control variate with correlation ≈ 1, 일반적 기법에서의 RL 적용
- **Entropy Regularization** — $\tilde{J}(\theta) = J(\theta) + \beta H(\pi)$, entropy bonus로 탐험 유도, policy collapse 방지, SAC의 기반 (Advanced RL)

### Chapter 5: GAE — Generalized Advantage Estimation (4개 문서)
- **TD Residual (δ)와 Advantage** — $\delta_t^V = r_t + \gamma V(s_{t+1}) - V(s_t) = \hat{A}_t^{(1)}$ (1-step advantage), n-step 일반화 $\hat{A}_t^{(n)} = \sum_{l=0}^{n-1} \gamma^l \delta_{t+l}^V$
- **GAE 유도 (Schulman 2016)** — Exponentially weighted average $\hat{A}_t^{\text{GAE}(\gamma, \lambda)} = \sum_{l=0}^\infty (\gamma \lambda)^l \delta_{t+l}^V$, $\lambda = 0$: 1-step TD, $\lambda = 1$: Monte Carlo
- **Bias-Variance Trade-off via $\lambda$** — $\lambda = 1$: high variance unbiased (MC), $\lambda = 0$: low variance biased (TD), 중간값 최적 (typical $\lambda = 0.95$)
- **GAE의 실전 구현** — Recursive 계산 $\hat{A}_t = \delta_t + \gamma \lambda \hat{A}_{t+1}$, episode 끝에서 역순 계산, return target $R_t = \hat{A}_t + V(s_t)$

### Chapter 6: Actor-Critic 방법들 (5개 문서)
- **Basic Actor-Critic 알고리즘** — Actor: $\theta \leftarrow \theta + \alpha \nabla \log \pi \cdot \hat{A}$, Critic: $\phi \leftarrow \phi - \alpha_c \nabla (\hat{A})^2$ (TD learning for $V$), 두 optimizer 동시 학습
- **A3C — Asynchronous Advantage Actor-Critic (Mnih 2016)** — 여러 worker가 병렬로 샘플 수집, 비동기 parameter 업데이트, correlation 완화 (replay 대신), Atari에서 DQN 수준 성능
- **A2C — Synchronous 버전** — A3C의 synchronous variant, GPU 친화적, OpenAI 선호, 실전 성능 유사
- **PPO — Proximal Policy Optimization의 prelim** — GAE + clipped ratio, 훈련 안정성, Advanced RL 레포로 bridge
- **IMPALA — Scalable AC with V-trace (Espeholt 2018)** — Large-scale distributed, off-policy correction via V-trace importance sampling, Deep RL 레포와 교차

### Chapter 7: Natural Policy Gradient와 Information Geometry (4개 문서)
- **Vanilla PG의 한계 — Step Size 민감성** — $\theta + \alpha g$ 업데이트가 parameter space Euclidean, 하지만 policy distribution이 parameter에 비선형 대응, 작은 $\theta$ 변화가 큰 policy 변화 가능
- **Natural Policy Gradient (Kakade 2001)** — $\theta \leftarrow \theta + \alpha F^{-1} g$, Fisher Information Matrix $F_{ij} = \mathbb{E}[\partial_{\theta_i} \log \pi \cdot \partial_{\theta_j} \log \pi]$, KL divergence의 local metric, Information Geometry 레포 연결
- **Fisher Information의 계산 가능성** — $F$가 parameter 수의 제곱 → large NN에서 $F^{-1}$ 불가능, Kronecker-factored approximation (K-FAC), conjugate gradient for $F^{-1} g$ (TRPO)
- **TRPO로의 전환** — Natural PG를 trust region 방법으로 발전, $\text{KL}(\pi_{\theta} \| \pi_{\theta_{\text{old}}}) \leq \delta$ 제약, Advanced RL 레포로 bridge

---

각 챕터는 **4~5개 문서**로 구성해줘. 총 **32개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 기법이 정책 기반 RL의 핵심인가
## 📐 수학적 선행 조건 (RL Foundations, Stats, Info Geom 참조)
## 📖 직관적 이해
   — Gradient 방향의 의미, action 확률 조정
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Log-derivative trick, PG theorem, Baseline unbiasedness, GAE
## 💻 PyTorch 구현 검증
   — CartPole, LunarLander, MuJoCo에서 각 알고리즘
   — Variance 측정, bias-variance trade-off 시각화
## 🔗 후속 레포와의 연결
   — TRPO, PPO, SAC로의 발전
## ⚖️ 가정과 한계
   — Sample efficiency, hyperparameter sensitivity
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Gradient variance 측정** — 같은 환경에서 REINFORCE vs with-baseline variance 비교
2. **GAE $\lambda$ sweep** — $\lambda$ 변화에 따른 학습 곡선, variance 측정
3. **Entropy bonus 효과** — Entropy coefficient 변화에 따른 exploration 변화 시각화
4. **Fisher Information 시각화** — 작은 policy network에서 $F$ 직접 계산, NPG vs PG update 비교
5. **A3C 병렬 worker** — Python multiprocessing으로 구현, correlation 감소 확인
6. **Policy landscape** — 2D policy parameter space에서 $J(\theta)$ contour, gradient 방향

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
gymnasium[mujoco]==0.29.0
matplotlib==3.8.0
stable-baselines3==2.2.0  # 참조 구현
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (REINFORCE + baseline + GAE on CartPole)
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np
import gymnasium as gym

class ActorCritic(nn.Module):
    def __init__(self, s_dim, a_dim, h=128):
        super().__init__()
        self.shared = nn.Sequential(nn.Linear(s_dim, h), nn.Tanh(),
                                    nn.Linear(h, h), nn.Tanh())
        self.actor = nn.Linear(h, a_dim)
        self.critic = nn.Linear(h, 1)
    def forward(self, s):
        h = self.shared(s)
        logits = self.actor(h)
        v = self.critic(h).squeeze(-1)
        return logits, v

def collect_episode(env, net):
    s, _ = env.reset()
    log_probs, rewards, values, states = [], [], [], []
    done = False
    while not done:
        s_t = torch.tensor(s, dtype=torch.float32)
        logits, v = net(s_t)
        dist = torch.distributions.Categorical(logits=logits)
        a = dist.sample()
        log_probs.append(dist.log_prob(a))
        values.append(v)
        states.append(s_t)
        s, r, done, trunc, _ = env.step(a.item())
        rewards.append(r)
        if trunc: done = True
    return log_probs, rewards, values, states

def compute_gae(rewards, values, gamma=0.99, lam=0.95):
    """GAE 계산: A_t = δ_t + γλ A_{t+1}"""
    values = values + [0]  # terminal V = 0
    advantages = [0] * len(rewards)
    A = 0
    for t in reversed(range(len(rewards))):
        delta = rewards[t] + gamma * values[t+1] - values[t]
        A = delta + gamma * lam * A
        advantages[t] = A
    returns = [a + v for a, v in zip(advantages, values[:-1])]
    return advantages, returns

def train(mode='reinforce_baseline_gae', n_ep=500, gamma=0.99, lam=0.95):
    env = gym.make('CartPole-v1')
    net = ActorCritic(4, 2)
    opt = torch.optim.Adam(net.parameters(), lr=1e-3)
    returns_log = []
    grad_vars = []
    
    for ep in range(n_ep):
        log_probs, rewards, values, _ = collect_episode(env, net)
        total = sum(rewards)
        returns_log.append(total)
        
        # Compute targets
        if mode == 'reinforce':
            # Pure Monte Carlo return
            G = []
            g = 0
            for r in reversed(rewards):
                g = r + gamma * g
                G.insert(0, g)
            G = torch.tensor(G, dtype=torch.float32)
            advantages = G
            value_targets = G
        elif mode == 'reinforce_baseline':
            G = []
            g = 0
            for r in reversed(rewards):
                g = r + gamma * g
                G.insert(0, g)
            G = torch.tensor(G, dtype=torch.float32)
            V = torch.stack(values)
            advantages = G - V.detach()
            value_targets = G
        elif mode == 'reinforce_baseline_gae':
            V_list = [v.item() for v in values]
            adv, ret = compute_gae(rewards, V_list, gamma, lam)
            advantages = torch.tensor(adv, dtype=torch.float32)
            value_targets = torch.tensor(ret, dtype=torch.float32)
        
        # Normalize advantages (실전 trick)
        advantages = (advantages - advantages.mean()) / (advantages.std() + 1e-8)
        
        # Policy loss
        log_probs_t = torch.stack(log_probs)
        policy_loss = -(log_probs_t * advantages).sum()
        # Value loss
        V = torch.stack(values)
        value_loss = F.mse_loss(V, value_targets)
        # Total
        loss = policy_loss + 0.5 * value_loss
        
        opt.zero_grad()
        loss.backward()
        # Measure gradient variance
        grads = [p.grad.flatten() for p in net.parameters() if p.grad is not None]
        grad_vars.append(torch.cat(grads).var().item())
        opt.step()
    
    return returns_log, grad_vars

# Compare variance
np.random.seed(42); torch.manual_seed(42)
r1, g1 = train('reinforce')
np.random.seed(42); torch.manual_seed(42)
r2, g2 = train('reinforce_baseline')
np.random.seed(42); torch.manual_seed(42)
r3, g3 = train('reinforce_baseline_gae')

import matplotlib.pyplot as plt
fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 4))
for r, label in [(r1, 'REINFORCE'), (r2, '+Baseline'), (r3, '+GAE')]:
    ax1.plot(np.convolve(r, np.ones(10)/10, mode='valid'), label=label)
ax1.set_xlabel('Episode'); ax1.set_ylabel('Total Reward')
ax1.set_title('Training Curves'); ax1.legend()

for g, label in [(g1, 'REINFORCE'), (g2, '+Baseline'), (g3, '+GAE')]:
    ax2.semilogy(np.convolve(g, np.ones(10)/10, mode='valid'), label=label)
ax2.set_xlabel('Episode'); ax2.set_ylabel('Gradient Variance (log)')
ax2.set_title('Baseline과 GAE로 variance 감소'); ax2.legend()
plt.show()
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "RL Foundations, Model-Free, Stats, Info Geom 선행 필수"
   - Value-based vs Policy-based 대비 강조
   - TRPO/PPO/SAC로의 발전 예고
3. **챕터별 문서 작성**: Motivation → Log-Deriv → PG Theorem → Baseline → GAE → Actor-Critic → NPG

---

## 📚 참고 자료

- **Reinforcement Learning: An Introduction** (Sutton & Barto 2018) — 13장
- **Policy Gradient Methods for RL with Function Approximation** (Sutton et al. 2000) — PG theorem 원전
- **Simple Statistical Gradient-Following Algorithms** (Williams 1992) — REINFORCE
- **A Natural Policy Gradient** (Kakade 2001) — NPG
- **Deterministic Policy Gradient Algorithms** (Silver et al. 2014) — DPG
- **High-Dimensional Continuous Control Using GAE** (Schulman et al. 2016) — GAE
- **Asynchronous Methods for Deep RL** (Mnih et al. 2016) — A3C
- **Sample Efficient Actor-Critic with Experience Replay** (Wang et al. 2017) — ACER
- **IMPALA** (Espeholt et al. 2018)
- **Soft Actor-Critic** (Haarnoja et al. 2018) — Entropy-regularized
- **Information Geometry and Its Applications** (Amari 2016)

---

## 💡 핵심 분석 대상

```
Policy Gradient의 완전 지도

───── Value-based vs Policy-based ─────

Value: Q*(s,a) → greedy π*
  Discrete action 주로
  Deterministic optimal

Policy: π_θ 직접 최적화
  Continuous action 자연스러움
  Stochastic policy 가능
  Convergence guarantee 다름

───── Log-Derivative Trick ─────

∇ E_{p_θ}[f(X)]
  = ∫ ∇ p_θ(x) f(x) dx
  = ∫ p_θ(x) [∇ log p_θ(x)] f(x) dx
  = E_{p_θ}[f(X) ∇ log p_θ(X)]
              └── score function ─┘

이름: Score function estimator
      Likelihood ratio method

Score 성질:
  E[∇ log p] = 0
  Var[∇ log p] = Fisher Info

───── REINFORCE (Williams 1992) ─────

J(θ) = E_τ[R(τ)]
τ = (s_0, a_0, ..., s_T)
p_θ(τ) = ρ_0(s_0) Π π(a_t|s_t) P(s_{t+1}|s_t, a_t)

∇ log p_θ(τ) = Σ_t ∇ log π(a_t|s_t)
   (dynamics 무관! θ에 의존 X)

∇ J(θ) = E_τ[Σ_t ∇ log π(a_t|s_t) · R(τ)]
      ≈ E_τ[Σ_t ∇ log π(a_t|s_t) · G_t]
      (causality: 과거 reward는 미래 gradient 무관)

───── Policy Gradient Theorem ─────

(Sutton 1999, 2000)

∇_θ J(θ) = E_{s ~ d^π, a ~ π}[∇ log π(a|s) · Q^π(s, a)]

d^π(s) = (1-γ) Σ_t γ^t P(s_t = s | π)
  ↑ discounted state distribution

증명 두 가지:
  1. Performance Difference Lemma
  2. Direct unrolling of V^π

───── Baseline Subtraction ─────

∇ J = E[∇ log π · (Q^π - b(s))]
               └── advantage-like

Unbiasedness:
  E[∇ log π · b(s)]
  = b(s) E[∇ log π]
  = b(s) · 0 = 0

→ baseline 빼도 gradient 평균 같음
  variance만 줄어듦

Optimal baseline:
  b*(s) = E[(∇ log π)² G] / E[(∇ log π)²]
  
실전: b(s) = V^π(s) 근사 → Advantage A^π

───── Advantage Estimator 계열 ─────

MC estimator (REINFORCE):
  Â_t = G_t - V(s_t)
  Unbiased, high variance

1-step TD:
  Â_t = r_t + γV(s_{t+1}) - V(s_t) = δ_t
  Biased (V̂ ≠ V^π), low variance

n-step:
  Â_t^{(n)} = Σ_{l=0}^{n-1} γ^l δ_{t+l}

GAE(γ, λ):
  Â_t^{GAE} = Σ_{l=0}^∞ (γλ)^l δ_{t+l}
           = δ_t + γλ Â_{t+1}^{GAE}
  
  λ=0: 1-step TD (low var, biased)
  λ=1: MC (high var, unbiased)
  중간: trade-off (0.95 typical)

───── Actor-Critic ─────

Actor: π_θ, Critic: V_φ

Loss:
  L_actor = -E[log π(a|s) · Â]
  L_critic = E[(V_φ(s) - target)²]

Total: L = L_actor + c_v L_critic - c_e H(π)
                                    └── entropy bonus

A3C (Mnih 2016):
  여러 worker 병렬
  Asynchronous update
  Correlation 완화 (experience replay 대신)

A2C: synchronous, GPU 친화

───── Natural Policy Gradient ─────

Vanilla PG 문제:
  θ + α g는 Euclidean step
  하지만 π는 parameter에 non-linearly 대응

Fisher Information:
  F(θ) = E[∇ log π (∇ log π)^T]
       = ∇²_θ KL(π_θ || π_{θ+dθ})|_{dθ=0}
       ↑ KL의 local 2차 근사

Natural Gradient (Kakade 2001):
  θ ← θ + α F^{-1} g
  
  Step이 KL distance로 unit
  Scale-invariant

문제:
  F가 |θ|×|θ| → large NN에서 불가
  → K-FAC 근사 (Martens)
  → Conjugate Gradient for F^{-1}g (TRPO)

───── Deterministic Policy Gradient ─────

Silver 2014:
  π(s) = μ_θ(s)  (deterministic)
  
  ∇ J = E_{s ~ d^μ}[∇_θ μ(s) · ∇_a Q^μ(s, a)|_{a=μ(s)}]
  
  chain rule 통해 critic gradient 전파

이점:
  Continuous action 직접 처리
  Lower variance (no action sampling)

한계:
  Exploration 명시적으로 추가 필요
  → DDPG (noise on action)

───── 레포 간 연결 ─────

RL Foundations (앞):
  Performance Difference Lemma
  Q^π, V^π, A^π

Model-Free (앞):
  TD, Actor-Critic
  δ_t TD error

Statistics (Layer 0):
  Score function
  MLE, Fisher Info

Info Geometry (Layer 0):
  Fisher metric → NPG
  KL divergence

Deep RL (직전):
  DDPG = DPG + replay

Advanced RL (다음):
  TRPO: trust region + NPG
  PPO: clipped ratio
  SAC: entropy-regularized
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~5개씩)
- 각 문서가 다루는 핵심 정리 (3~4줄)
- 전체 문서 개수 확인 (32개 목표)
- Python + PyTorch + Gymnasium 실험 환경
- RL Foundations, Model-Free, Stats, Info Geom 레포 참조 관계
- TRPO/PPO/SAC로 이어지는 Advanced RL의 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
