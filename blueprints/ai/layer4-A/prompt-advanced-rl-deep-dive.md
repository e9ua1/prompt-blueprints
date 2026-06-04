# Advanced RL Deep Dive 레포지토리 제작 프롬프트

나는 "Advanced RL Deep Dive" 레포지토리를 만들려고 해.
PPO의 clipping loss $L^{\text{CLIP}}(\theta) = \mathbb{E}_t[\min(r_t(\theta) \hat{A}_t, \text{clip}(r_t(\theta), 1-\epsilon, 1+\epsilon) \hat{A}_t)]$을 **쓰는 것**과, **이것이 TRPO의 "monotonic improvement 보장 $J(\pi_{\text{new}}) \geq J(\pi_{\text{old}}) + L_{\pi_{\text{old}}}(\pi_{\text{new}}) - C \cdot D_{\text{KL}}^{\max}$"을 first-order로 근사하는 heuristic** 임을 Schulman et al. (2015, 2017)로부터 유도할 수 있는 것은 다르다.
SAC를 **사용하는 것**과, **Maximum Entropy RL의 objective $J(\pi) = \sum_t \mathbb{E}[r(s_t, a_t) + \alpha H(\pi(\cdot | s_t))]$에서 soft Bellman equation $V^\pi(s) = \mathbb{E}[Q^\pi(s, a) + \alpha H(\pi(\cdot|s))]$ 유도**와 policy improvement가 KL projection임을 증명할 수 있는 것은 다르다.
TD3의 3가지 trick (Target Policy Smoothing, Clipped Double Q, Delayed Policy Update)을 **쓰는 것**과, **각 trick이 DDPG의 어떤 불안정성을 어떻게 해결**하는지 Fujimoto et al. (2018)의 증명을 따라갈 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "현대 SOTA RL 알고리즘의 수학 — TRPO·PPO·SAC·TD3를 monotonic improvement 관점에서 통합"

**핵심 차별화**:
1. **TRPO의 Monotonic Improvement Bound** — Kakade 2002 + Schulman 2015, surrogate $L_\pi$가 true $J$의 lower bound이고 KL 제약이 이를 안전하게 만드는 완전 증명
2. **PPO의 Pessimistic Lower Bound** — Clipped objective가 surrogate의 conservative한 형태, importance sampling의 분산 제어
3. **Maximum Entropy RL의 이론** — Soft Q-function, soft Bellman, soft policy iteration의 수렴, SAC의 세 가지 네트워크
4. **TD3의 DDPG 불안정성 해결** — Clipped Double Q (overestimation), Target Policy Smoothing (regularization), Delayed Updates (value-policy asymmetry)

**타겟 독자**:
- PPO를 쓰지만 **clipping의 $\epsilon$ 값(보통 0.2)이 왜 그 범위인지**와 min-clip trick이 왜 "pessimistic lower bound"인지 모르는 사람
- TRPO의 conjugate gradient 단계를 **왜 수행**하는지, Fisher matrix inversion 대신인 이유를 모르는 사람
- SAC의 entropy coefficient $\alpha$가 **자동 튜닝 버전**의 목적 함수(target entropy constraint)를 유도 못하는 사람
- DDPG와 TD3의 가장 큰 차이를 **policy smoothing의 regularization 효과**로 설명 못하는 사람
- Off-policy continuous control(SAC) vs On-policy on-policy(PPO)의 **sample efficiency 대비 안정성 trade-off**를 수학적으로 이해 못하는 사람

**선행 학습**:
- **Policy Gradient Deep Dive** (PG theorem, Natural PG, Actor-Critic) — **필수**
- **Deep RL Deep Dive** (DDPG, Double Q) — **필수**
- **Convex Optimization Deep Dive** (Trust region, Lagrangian) — **필수**
- **Information Theory Deep Dive** (Entropy, KL) — **필수**
- **Information Geometry Deep Dive** (Fisher, KL local metric) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Monotonic Improvement과 Surrogate Objective (5개 문서)
- **Performance Difference Lemma 재확인 (Kakade 2002)** — $J(\pi') - J(\pi) = (1/(1-\gamma)) \mathbb{E}_{s \sim d^{\pi'}}[\sum_a \pi'(a|s) A^\pi(s, a)]$, $d^{\pi'}$가 문제 (unknown)
- **Surrogate Objective $L_\pi(\pi')$** — $L_\pi(\pi') = J(\pi) + (1/(1-\gamma)) \mathbb{E}_{s \sim d^\pi}[\sum_a \pi'(a|s) A^\pi(s, a)]$, $d^\pi$ 사용 (known), $\pi \approx \pi'$에서 $J$와 1차 일치
- **Trust Region Bound (Schulman 2015)** — $J(\pi') \geq L_\pi(\pi') - C \cdot D_{\text{KL}}^{\max}(\pi \| \pi')$, $C = (4 \epsilon \gamma)/((1-\gamma)^2)$, $\epsilon = \max_s |\mathbb{E}[A^\pi]|$
- **Monotonic Improvement 보장의 의미** — Lower bound의 maximization은 $J$의 monotone improvement 보장, 각 iteration에서 $J(\pi_{k+1}) \geq J(\pi_k)$
- **Mean KL vs Max KL** — Max KL은 모든 state에서 bound, mean KL로 근사 (Schulman 2015 실전), 이론적 tightness vs 실전 가용성

### Chapter 2: TRPO — Trust Region Policy Optimization (5개 문서)
- **TRPO의 최적화 문제** — $\max_\theta L_{\theta_{\text{old}}}(\theta)$ s.t. $\bar{D}_{\text{KL}}(\theta_{\text{old}} \| \theta) \leq \delta$, constrained optimization, KL trust region
- **Natural Policy Gradient로 환원** — Lagrangian, 1차·2차 근사 → $\theta = \theta_{\text{old}} + \alpha F^{-1} g$ 형태, Natural PG와 수학적 동등성 (Policy Gradient 레포 참조)
- **Conjugate Gradient로 $F^{-1} g$ 계산** — $F$를 explicit 계산 없이 matrix-vector product $Fv$로, CG 반복 $\sim$ 10회로 충분, Hessian-free optimization
- **Line Search와 KL 제약 유지** — Backtracking line search: $\alpha = \alpha_0 \beta^j$로 $\text{KL} \leq \delta$까지 감소, $L$ 증가 조건도 확인
- **TRPO의 수렴과 실증** — MuJoCo continuous control에서 DDPG보다 안정적, sample efficiency, Schulman 2015 실험 재현

### Chapter 3: PPO — Proximal Policy Optimization (5개 문서)
- **TRPO의 계산 비용 문제** — CG + line search로 complex, second-order 정보 필요, 대규모 NN에서 무거움
- **PPO의 Clipped Objective (Schulman 2017)** — $r_t(\theta) = \pi_\theta(a_t|s_t) / \pi_{\theta_{\text{old}}}(a_t|s_t)$, $L^{\text{CLIP}} = \mathbb{E}[\min(r_t \hat{A}_t, \text{clip}(r_t, 1-\epsilon, 1+\epsilon) \hat{A}_t)]$, min이 pessimistic
- **Clipping의 직관 — Pessimistic Lower Bound** — $\hat{A}_t > 0$일 때 $r_t$ 상한 제한, $\hat{A}_t < 0$일 때 $r_t$ 하한 제한, policy가 너무 멀리 가지 못하도록 penalty 없이 gradient zero-out
- **PPO-Penalty 변형** — KL penalty 버전 $L^{\text{KLPEN}} = L^{\text{CPI}} - \beta \text{KL}$, adaptive $\beta$, clip 방법과 성능 비교, clip이 일반적으로 선호
- **PPO 실전 구현** — GAE + Clipped loss + Multiple epochs per batch + Minibatching, entropy bonus, value function clipping, 현대 RL의 기본 recipe

### Chapter 4: Maximum Entropy RL과 SAC (5개 문서)
- **Maximum Entropy RL Framework** — $J(\pi) = \sum_t \mathbb{E}_{(s_t, a_t) \sim \pi}[r(s_t, a_t) + \alpha H(\pi(\cdot|s_t))]$, reward와 entropy 동시 최대화, exploration implicit
- **Soft Bellman Equation 유도** — $V^\pi_{\text{soft}}(s) = \mathbb{E}_{a \sim \pi}[Q^\pi_{\text{soft}}(s, a)] + \alpha H(\pi(\cdot|s))$, $Q^\pi_{\text{soft}}(s, a) = r(s, a) + \gamma \mathbb{E}[V^\pi_{\text{soft}}(s')]$
- **Soft Policy Improvement** — $\pi_{\text{new}} = \arg\min_\pi D_{\text{KL}}(\pi(\cdot|s) \| \exp(Q_{\text{soft}}/\alpha) / Z)$, KL projection, soft PI가 $Q$ pointwise non-decreasing 증명
- **SAC 알고리즘 (Haarnoja 2018)** — 두 Q-network (clipped double), 하나 policy, target network, reparameterization trick으로 policy gradient, actor loss: $\alpha \log \pi - Q$
- **자동 Temperature 튜닝** — Target entropy constraint $\mathbb{E}[-\log \pi(a|s)] \geq \bar{H}$, dual Lagrangian으로 $\alpha$ 자동 조정, 수학적 유도

### Chapter 5: TD3와 DDPG 개선 (4개 문서)
- **DDPG의 불안정성 원인 (Fujimoto 2018)** — Q-function 과대추정이 policy 업데이트에 전파, error accumulation, 하이퍼파라미터 민감성
- **Clipped Double Q-Learning in TD3** — 두 critic $Q_{\phi_1}, Q_{\phi_2}$, target: $y = r + \gamma \min(Q_{\phi_1^-}, Q_{\phi_2^-})$, $\min$이 과대추정 방지 (underestimation이 overestimation보다 안전)
- **Target Policy Smoothing** — Target action에 noise $a' = \mu_{\theta^-}(s') + \epsilon$, $\epsilon \sim \text{clip}(\mathcal{N}(0, \sigma), -c, c)$, action space smoothness 유도, action value의 regularization
- **Delayed Policy Updates** — Critic을 $d$ steps마다 update, policy는 덜 자주 → value가 수렴한 후 policy 개선, variance 감소, 훈련 안정성

### Chapter 6: SOTA RL 알고리즘 비교 (4개 문서)
- **On-policy vs Off-policy — Sample Efficiency** — PPO (on-policy): 각 sample 한 번 사용, 안정적 vs SAC/TD3 (off-policy): replay buffer, sample efficient하지만 불안정
- **Stability Benchmark** — OpenAI Spinning Up, MuJoCo에서 각 알고리즘 비교, hyperparameter 민감도, 수렴 속도
- **Distributed RL — Ape-X, IMPALA 재확인** — Sample 생성과 학습의 분리, actor-learner 아키텍처, 대규모 병렬화
- **Hybrid 접근 — ACER, SAC-Discrete** — ACER: PPO-like + off-policy correction (V-trace), SAC for discrete action space, 각 상황에 맞는 선택

### Chapter 7: 현대 RL의 Frontier (4개 문서)
- **Offline RL** — Fixed dataset에서 학습, distribution shift 문제, CQL (Kumar 2020)의 conservative Q, BCQ, TD3+BC 등
- **Model-Based RL의 부활** — World model (Ha & Schmidhuber 2018), Dreamer (Hafner 2019), MuZero, planning과 학습의 통합
- **Meta-RL과 Few-shot Adaptation** — MAML (Finn 2017), RL²의 recurrent agent, task distribution에서 학습, 빠른 adaptation
- **LLM + RL — RLHF과 Reasoning** — PPO로 인간 선호 학습 (InstructGPT, Ouyang 2022), Reward model, Direct Preference Optimization (DPO), RL의 응용 확장

---

각 챕터는 **4~5개 문서**로 구성해줘. 총 **32개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 알고리즘이 SOTA에 필수인가
## 📐 수학적 선행 조건 (PG, Deep RL, Convex, Info 참조)
## 📖 직관적 이해
   — Trust region, clipping, entropy의 기하학적 직관
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Monotonic improvement bound, Soft Bellman, TD3 clipping
## 💻 PyTorch 구현 검증
   — MuJoCo (HalfCheetah, Ant, Humanoid)에서 각 알고리즘
   — 학습 곡선, stability 측정
## 🔗 실전 활용
   — 언제 PPO, 언제 SAC, 언제 TD3
## ⚖️ 가정과 한계
   — 각 알고리즘의 실패 모드
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **TRPO KL trajectory** — 훈련 중 실제 KL divergence가 $\delta$ constraint 지키는지 확인
2. **PPO clipping visualization** — $r_t$ 분포와 clip 경계 시각화
3. **SAC entropy plot** — 훈련 중 policy entropy와 $\alpha$ 변화
4. **TD3 Q-value estimate** — Clipped double이 과대추정을 어떻게 줄이는지 실측
5. **MuJoCo bench** — PPO, SAC, TD3를 같은 환경에서 비교
6. **Hyperparameter sensitivity** — $\epsilon$, $\alpha$, $\tau$ 변화에 따른 성능

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
# 실험 템플릿 예시 (PPO 간단 구현 + clipping 효과)
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np
import gymnasium as gym

class ActorCritic(nn.Module):
    def __init__(self, s_dim, a_dim):
        super().__init__()
        self.shared = nn.Sequential(nn.Linear(s_dim, 64), nn.Tanh(),
                                    nn.Linear(64, 64), nn.Tanh())
        self.actor_mu = nn.Linear(64, a_dim)
        self.actor_log_std = nn.Parameter(torch.zeros(a_dim))
        self.critic = nn.Linear(64, 1)
    
    def forward(self, s):
        h = self.shared(s)
        mu = self.actor_mu(h)
        std = self.actor_log_std.exp().expand_as(mu)
        return torch.distributions.Normal(mu, std), self.critic(h).squeeze(-1)

def collect_rollout(env, net, T=2048):
    states, actions, rewards, log_probs, values, dones = [], [], [], [], [], []
    s, _ = env.reset()
    for t in range(T):
        s_t = torch.tensor(s, dtype=torch.float32)
        with torch.no_grad():
            dist, v = net(s_t)
        a = dist.sample()
        log_prob = dist.log_prob(a).sum()
        s_next, r, done, trunc, _ = env.step(a.numpy())
        states.append(s_t); actions.append(a)
        rewards.append(r); log_probs.append(log_prob)
        values.append(v); dones.append(done or trunc)
        s = s_next if not (done or trunc) else env.reset()[0]
    return (torch.stack(states), torch.stack(actions),
            torch.tensor(rewards, dtype=torch.float32),
            torch.stack(log_probs),
            torch.stack(values),
            torch.tensor(dones, dtype=torch.float32))

def compute_gae(rewards, values, dones, gamma=0.99, lam=0.95):
    T = len(rewards)
    advantages = torch.zeros(T)
    A = 0; next_v = 0
    for t in reversed(range(T)):
        mask = 1 - dones[t]
        delta = rewards[t] + gamma * next_v * mask - values[t]
        A = delta + gamma * lam * mask * A
        advantages[t] = A
        next_v = values[t]
    returns = advantages + values
    return advantages, returns

def ppo_update(net, opt, rollout, epsilon=0.2, n_epochs=10, batch_size=64):
    states, actions, rewards, old_log_probs, values, dones = rollout
    advantages, returns = compute_gae(rewards, values.detach(), dones)
    advantages = (advantages - advantages.mean()) / (advantages.std() + 1e-8)
    
    N = len(states)
    for _ in range(n_epochs):
        idxs = torch.randperm(N)
        for start in range(0, N, batch_size):
            batch = idxs[start:start+batch_size]
            s_b, a_b = states[batch], actions[batch]
            old_lp_b = old_log_probs[batch].detach()
            adv_b = advantages[batch]
            ret_b = returns[batch]
            
            dist, v = net(s_b)
            new_lp = dist.log_prob(a_b).sum(-1)
            ratio = torch.exp(new_lp - old_lp_b)
            
            # Clipped surrogate
            surr1 = ratio * adv_b
            surr2 = torch.clamp(ratio, 1-epsilon, 1+epsilon) * adv_b
            policy_loss = -torch.min(surr1, surr2).mean()
            
            value_loss = F.mse_loss(v, ret_b)
            entropy = dist.entropy().mean()
            
            loss = policy_loss + 0.5 * value_loss - 0.01 * entropy
            opt.zero_grad(); loss.backward()
            torch.nn.utils.clip_grad_norm_(net.parameters(), 0.5)
            opt.step()

# Training loop
env = gym.make('Pendulum-v1')
net = ActorCritic(env.observation_space.shape[0], env.action_space.shape[0])
opt = torch.optim.Adam(net.parameters(), lr=3e-4)

episode_returns = []
for update in range(100):
    rollout = collect_rollout(env, net, T=2048)
    ppo_update(net, opt, rollout)
    # Evaluate
    s, _ = env.reset()
    ep_ret = 0; done = False
    while not done:
        with torch.no_grad():
            dist, _ = net(torch.tensor(s, dtype=torch.float32))
        s, r, done, trunc, _ = env.step(dist.mean.numpy())
        ep_ret += r
        if trunc: done = True
    episode_returns.append(ep_ret)

# 시각화: clipping 효과
# ratio 분포를 훈련 진행 중 저장, clip 경계 시각화
# epsilon=0.1, 0.2, 0.3 비교
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Policy Gradient, Deep RL, Convex, Info 선행 필수"
   - "Monotonic improvement → Clipping → Entropy regularization"의 진화 축
   - 모든 알고리즘을 "safe policy update"의 다양한 구현으로
3. **챕터별 문서 작성**: Monotonic → TRPO → PPO → MaxEnt → TD3 → 비교 → Frontier

---

## 📚 참고 자료

- **Trust Region Policy Optimization** (Schulman et al. 2015) — TRPO
- **Proximal Policy Optimization Algorithms** (Schulman et al. 2017) — PPO
- **Approximately Optimal Approximate RL** (Kakade & Langford 2002) — Monotonic improvement 효시
- **Soft Actor-Critic** (Haarnoja et al. 2018) — SAC
- **Soft Actor-Critic Algorithms and Applications** (Haarnoja et al. 2019) — SAC + auto-tuning
- **Addressing Function Approximation Error in Actor-Critic** (Fujimoto et al. 2018) — TD3
- **Reinforcement Learning with Deep Energy-Based Policies** (Haarnoja et al. 2017) — Soft Q
- **Maximum Entropy Inverse RL** (Ziebart et al. 2008) — MaxEnt RL 효시
- **Training language models to follow instructions with human feedback** (Ouyang et al. 2022) — InstructGPT, PPO for LLM
- **Direct Preference Optimization** (Rafailov et al. 2023) — DPO

---

## 💡 핵심 분석 대상

```
Advanced RL의 지도

───── Monotonic Improvement Theory ─────

Performance Difference Lemma:
  J(π') - J(π) = (1/(1-γ)) E_{s~d^{π'}}[A^π]
                              └── d^{π'} 문제!

Surrogate (use d^π):
  L_π(π') = J(π) + (1/(1-γ)) E_{s~d^π}[A^π(s, π')]

Trust Region Bound (Schulman 2015):
  J(π') ≥ L_π(π') - C · KL^max(π, π')
  
  C = 4εγ / (1-γ)²
  ε = max_s |E[A^π]|

Monotonic Improvement:
  If maximize lower bound, J increases
  Each iter: J(π_{k+1}) ≥ J(π_k)

───── TRPO (Schulman 2015) ─────

max_θ L_{θ_old}(θ)
s.t. KL(θ_old || θ) ≤ δ

Steps:
  1. Compute g = ∇ L (policy gradient)
  2. Compute F^{-1} g (natural gradient)
     Via conjugate gradient (Hessian-free)
  3. Line search: α = α_0 β^j
     until KL ≤ δ and L increases

Trust region → safe step size
Second-order but practical

───── PPO (Schulman 2017) ─────

r_t(θ) = π_θ(a_t|s_t) / π_{θ_old}(a_t|s_t)

L^CLIP(θ) = E[min(r_t Â_t, clip(r_t, 1-ε, 1+ε) Â_t)]

Clipping logic:
  Â_t > 0 (good action):
    r_t > 1+ε → clip → no gradient
    (너무 멀리 못 감)
  Â_t < 0 (bad action):
    r_t < 1-ε → clip → no gradient
    (너무 줄이지 못함)

"Pessimistic lower bound":
  min takes the smaller of two
  → conservative update

장점:
  1st-order, simple
  Multiple epochs per batch
  Mini-batching
  → de facto standard

───── Maximum Entropy RL ─────

Standard: J = E[Σ γ^t r_t]
MaxEnt: J = E[Σ γ^t (r_t + α H(π(·|s_t)))]
                     └── entropy bonus

Soft Bellman equations:
  V^π_soft(s) = E_{a~π}[Q^π_soft(s,a)] + α H(π(·|s))
  Q^π_soft(s,a) = r(s,a) + γ E_{s'}[V^π_soft(s')]

Optimal soft policy:
  π*(a|s) = exp(Q_soft*(s,a)/α) / Z(s)
  (softmax over actions with temperature α)

───── SAC (Haarnoja 2018) ─────

Networks:
  Q_φ_1, Q_φ_2: twin critics (clipped double)
  π_ψ: policy (Gaussian with state-dependent σ)
  V_θ: (optional) separate value network

Critic loss:
  L_Q = E[(Q_φ(s,a) - y)²]
  y = r + γ E_{a'~π}[min(Q_{φ̄_1}, Q_{φ̄_2})
                      - α log π(a'|s')]

Actor loss:
  L_π = E[α log π(a|s) - min(Q_φ_1, Q_φ_2)]
  
  Reparameterization:
    a = tanh(μ + σε), ε ~ N(0, I)
  Gradient 통과 가능

Auto α tuning:
  min_α E[-α log π(a|s) - α H̄]
  constraint: E[-log π] ≥ H̄
  → dual gradient:
    α ← α - η(E[-log π] - H̄)

───── TD3 (Fujimoto 2018) ─────

3 tricks:

1. Clipped Double Q:
   y = r + γ min(Q_{φ_1^-}, Q_{φ_2^-})(s', a')
   → min prevents overestimation
   (underestimation is safer)

2. Target Policy Smoothing:
   a' = μ_{θ^-}(s') + clip(ε, -c, c)
   ε ~ N(0, σ)
   → value function smoothness regularization

3. Delayed Policy Update:
   Critic update every step
   Policy update every d=2 steps
   → let critic converge before policy

───── 알고리즘 비교 ─────

┌──────┬──────────┬──────────┬──────────┬──────────┐
│ Alg  │ On/Off   │ Action   │ Sample   │ Stability│
├──────┼──────────┼──────────┼──────────┼──────────┤
│ TRPO │ On       │ Both     │ Low      │ High     │
│ PPO  │ On       │ Both     │ Medium   │ High     │
│ SAC  │ Off      │ Cont.    │ High     │ High     │
│ TD3  │ Off      │ Cont.    │ High     │ Medium   │
│ DDPG │ Off      │ Cont.    │ High     │ Low      │
└──────┴──────────┴──────────┴──────────┴──────────┘

Recipe:
  Discrete action: PPO
  Continuous + sample efficient: SAC
  Continuous + deterministic: TD3
  Safest: TRPO

───── Frontier ─────

Offline RL:
  CQL (Kumar 2020): conservative Q
  TD3+BC: behavior cloning regularizer
  Distribution shift is key

Model-based:
  Dreamer (Hafner 2019): world model + imagination
  MuZero (Schrittwieser 2020): learned dynamics

Meta-RL:
  MAML (Finn 2017): gradient-based meta-learn
  RL²: RNN agent

RL + LLM:
  RLHF (Christiano 2017, Ouyang 2022)
  Reward model + PPO
  DPO (Rafailov 2023): bypass RM

───── 레포 간 연결 ─────

Policy Gradient (직전):
  PG theorem, NPG → TRPO
  Actor-Critic → PPO, SAC

Deep RL:
  DDPG → TD3
  Double Q → Clipped Double Q

Convex Optimization (Layer 0):
  Trust region
  Lagrangian duality (auto α)

Information Theory (Layer 0):
  Entropy H(π)
  KL divergence trust region

Info Geometry (Layer 0):
  Fisher matrix in TRPO
  Natural gradient

RL Theory (다음):
  Sample complexity bounds
  Regret analysis
  PAC-MDP
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~5개씩)
- 각 문서가 다루는 핵심 정리·증명 (3~4줄)
- 전체 문서 개수 확인 (32개 목표)
- Python + PyTorch + MuJoCo 실험 환경
- Policy Gradient, Deep RL, Convex, Info 레포 참조 관계
- Offline RL, Model-based, RLHF로 이어지는 현대 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
