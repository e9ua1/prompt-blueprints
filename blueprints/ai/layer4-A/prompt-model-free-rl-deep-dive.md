# Model-Free RL Deep Dive 레포지토리 제작 프롬프트

나는 "Model-Free RL Deep Dive" 레포지토리를 만들려고 해.
Q-Learning을 **코드로 짜는 것**과, **$Q_{t+1}(s, a) = Q_t(s, a) + \alpha_t [R + \gamma \max_{a'} Q_t(s', a') - Q_t(s, a)]$가 Robbins-Monro 조건 $\sum_t \alpha_t = \infty$, $\sum_t \alpha_t^2 < \infty$ 하에서 $Q^*$에 almost surely 수렴**한다는 Watkins (1989), Jaakkola et al. (1994)의 증명을 따라갈 수 있는 것은 다르다.
TD(0)와 Monte Carlo의 차이를 **"bootstrapping 여부"로 아는 것**과, **둘의 bias-variance trade-off**를 수학적으로 분석하고 **TD(λ)의 eligibility trace가 forward view와 backward view로 동치**임을 증명할 수 있는 것은 다르다.
SARSA가 **on-policy**인 것과 Q-Learning이 **off-policy**인 것을 **용어로 아는 것**과, **Cliff Walking 환경에서 왜 SARSA는 안전 경로를, Q-Learning은 최적이지만 위험한 경로를 배우는지**를 수학적으로 설명할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "MDP를 모르는 상태에서의 학습 — Monte Carlo, TD, Q-Learning 수렴 증명의 완전 분해"

**핵심 차별화**:
1. **Robbins-Monro 확률근사 정리** — Stochastic approximation의 일반 이론으로 Q-Learning·TD 수렴을 한 번에 증명
2. **Bias-Variance Trade-off in Bootstrap** — MC(unbiased, high variance) vs TD(biased, low variance)의 수학적 분석
3. **On-policy vs Off-policy의 엄밀한 구분** — Behavior policy와 target policy의 분리, Importance sampling의 역할
4. **n-step, TD(λ), Eligibility Trace** — 유한 n-step에서 무한 추적으로, forward-backward equivalence 증명

**타겟 독자**:
- Q-Learning을 구현하는데 **$\alpha$를 어떻게 스케줄해야 수렴 보장**되는지 (Robbins-Monro 조건)를 모르는 사람
- MC vs TD를 개념으로 아는데 **어느 쪽이 언제 더 효율적**인지 bias-variance로 설명 못하는 사람
- TD(λ)의 $\lambda$ 파라미터가 **eligibility trace의 감쇠율**인 이유와 $\lambda = 0$이 TD(0), $\lambda = 1$이 MC인 이유를 유도 못하는 사람
- Expected SARSA와 일반 SARSA의 차이가 **policy expectation을 analytical하게 계산 vs sample**인 것을 모르는 사람
- Double Q-Learning이 **왜 max bias를 줄이는지** (van Hasselt 2010의 증명)를 모르는 사람

**선행 학습**:
- **RL Foundations Deep Dive** (MDP, Bellman, DP) — **필수**
- **Probability Theory Deep Dive** (집중부등식, martingale) — **필수**
- **Mathematical Statistics Deep Dive** (수렴 이론) — **필수**
- **Stochastic Processes Deep Dive** (Markov chain ergodicity) — **필수**

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Model-Free Setting의 정의 (4개 문서)
- **Model-based vs Model-free 패러다임 비교** — $P, R$ 알려짐 vs 샘플 $(s, a, r, s')$만 주어짐, planning vs learning, sample complexity trade-off
- **Online vs Offline Learning** — Interactive environment (on-line, on-policy) vs 저장된 데이터 (offline, batch), each의 도전
- **Generalized Policy Iteration 재확인 — 샘플 기반 버전** — RL Foundations의 GPI를 evaluation·improvement가 샘플로 되는 방식으로 확장, 모든 MF 알고리즘의 틀
- **Exploration-Exploitation Dilemma** — $\epsilon$-greedy, Boltzmann, UCB의 기본 형식, 충분한 탐험 필요성, 후속(RL Theory) 레포와 연결

### Chapter 2: Monte Carlo Methods (5개 문서)
- **First-Visit vs Every-Visit MC** — $V^\pi(s) = \mathbb{E}[G_t | S_t = s]$의 sample mean 추정, first-visit vs every-visit의 편향 분석 (first-visit은 unbiased, every-visit은 asymptotically unbiased)
- **MC Convergence 증명** — Law of Large Numbers로 $\hat{V}(s) \to V^\pi(s)$ a.s., variance는 return $G_t$의 $\sigma^2$에 비례, 수렴률 $O(1/\sqrt{N})$
- **MC Control with Exploring Starts** — 모든 $(s, a)$가 무한번 시작, $Q(s, a)$를 직접 추정, greedy improvement, 수렴 조건(Sutton & Barto 5.3)
- **On-Policy MC — $\epsilon$-soft Policy** — $\epsilon$-greedy로 충분한 탐험 보장, $\epsilon$-greedy에서의 수렴 (단, $\epsilon$-optimal policy로)
- **Off-Policy MC — Importance Sampling** — Behavior $b$, target $\pi$의 분리, IS ratio $\rho_t = \prod_{k} \pi(a_k|s_k) / b(a_k|s_k)$, ordinary vs weighted IS, variance 문제

### Chapter 3: Temporal Difference Learning (5개 문서)
- **TD(0) 업데이트와 직관** — $V(s) \leftarrow V(s) + \alpha[R + \gamma V(s') - V(s)]$, MC(전체 return)와 DP(bootstrapping) 사이의 타협, TD error $\delta = R + \gamma V(s') - V(s)$의 역할
- **TD(0) Convergence (Tsitsiklis & Van Roy 1997)** — Tabular TD(0)이 $V^\pi$로 a.s. 수렴 증명, Robbins-Monro 조건, stochastic approximation의 일반 결과 적용
- **SARSA — On-Policy TD Control** — $Q(s, a) \leftarrow Q(s, a) + \alpha[R + \gamma Q(s', a') - Q(s, a)]$, $a'$가 현재 policy에서 선택, on-policy 성격, 수렴 조건
- **Expected SARSA** — $Q(s, a) \leftarrow Q + \alpha[R + \gamma \sum_{a'} \pi(a'|s') Q(s', a') - Q]$, $\max$ 대신 expectation, variance 감소 (explicit policy expectation)
- **TD(0) vs MC — Bias-Variance Trade-off** — MC는 unbiased·high variance, TD(0)는 biased·low variance, $V^\pi$가 incorrect일 때 TD는 wrong target으로 bootstrap, 각 시점의 예시

### Chapter 4: Q-Learning과 수렴 증명 (6개 문서)
- **Q-Learning 알고리즘 (Watkins 1989)** — $Q(s, a) \leftarrow Q + \alpha[R + \gamma \max_{a'} Q(s', a') - Q]$, off-policy 성격 (max가 current behavior와 독립), Behavior policy는 $\epsilon$-greedy 가능
- **Q-Learning이 Bellman Optimality의 확률 근사** — Fixed point가 $Q^*$, $T^* Q = Q^*$를 noisy update로 추적, 기댓값이 정확히 $T^* Q$
- **Robbins-Monro Stochastic Approximation Theorem** — $x_{t+1} = x_t + \alpha_t [F(x_t) + w_t]$의 수렴, $\sum \alpha_t = \infty$, $\sum \alpha_t^2 < \infty$, $F$가 contraction mapping일 때 수렴 보장
- **Watkins-Dayan의 Q-Learning Convergence 증명 (1992)** — 모든 $(s, a)$가 infinitely often 방문, RM 조건, bounded reward 하에서 $Q_t \to Q^*$ a.s., 증명의 핵심 ($T^*$의 contraction 활용)
- **Jaakkola-Jordan-Singh의 일반화 증명 (1994)** — 더 일반적인 stochastic approximation framework, Q-Learning·TD·MC를 통합 관점에서, variance bound 포함
- **Maximization Bias and Double Q-Learning (van Hasselt 2010)** — $\mathbb{E}[\max Q] \geq \max \mathbb{E}[Q]$ (Jensen), 과대추정의 수학적 원인, 두 개의 $Q^A, Q^B$로 bias 감소 증명

### Chapter 5: n-step TD와 Eligibility Trace (5개 문서)
- **n-step Return의 정의** — $G_{t:t+n} = R_{t+1} + \gamma R_{t+2} + \cdots + \gamma^{n-1} R_{t+n} + \gamma^n V(S_{t+n})$, MC($n = \infty$)와 TD(0)($n = 1$) 사이의 연속체
- **n-step TD의 bias-variance** — $n$이 클수록 bias ↓, variance ↑ (MC 극한), 최적 $n$은 환경·task dependent
- **TD(λ)와 λ-return** — $G_t^\lambda = (1-\lambda) \sum_{n=1}^\infty \lambda^{n-1} G_{t:t+n}$, geometric weighting, $\lambda = 0$은 TD(0), $\lambda = 1$은 MC
- **Forward View vs Backward View of TD(λ)** — Forward: 각 $t$에서 미래의 $\lambda$-return 업데이트 (episodic에서만), Backward: eligibility trace $e_t$로 온라인 업데이트, 두 관점의 동치성 증명
- **Eligibility Trace 구현 — Accumulating, Replacing, Dutch** — $e_t(s) = \gamma \lambda e_{t-1}(s) + \mathbb{1}[S_t = s]$, replacing trace ($\max$), dutch trace의 정확 등가성

### Chapter 6: Actor-Critic의 시작 (4개 문서)
- **Actor-Critic 프레임워크** — Actor(policy)와 Critic(value function)을 분리, DP·MC·TD의 통합 관점, GPI의 continuous 버전
- **Policy Evaluation as Critic** — TD(0)이나 MC로 $V^\pi, Q^\pi$ 추정, baseline으로 사용, variance reduction 기반
- **Policy Improvement as Actor** — $\epsilon$-greedy나 softmax policy, critic 기반 개선, Policy Gradient 레포로 이어지는 기반
- **Linear Function Approximation과 Semi-gradient** — $\hat{V}(s; \theta) = \theta^T \phi(s)$, semi-gradient TD: 일반 gradient가 아님 ($\nabla \hat{V}$만 사용, target은 fixed), on-policy 수렴 (Tsitsiklis-VanRoy)

### Chapter 7: 실전 Model-Free RL 이슈 (4개 문서)
- **Function Approximation의 Deadly Triad** — Off-policy + Bootstrapping + FA → 발산 가능, Baird's counterexample, tabular과 linear on-policy만 보장, Deep RL의 근본 문제 (다음 레포)
- **Experience Replay와 Off-policy 학습 준비** — Replay buffer로 i.i.d. 근사, correlation 완화, sample efficiency ↑, DQN으로 이어지는 기술
- **Reward Shaping과 Potential-based** — $F(s, s') = \gamma \Phi(s') - \Phi(s)$ 형식의 shaping이 optimal policy 불변 (Ng 1999), 실전 수렴 가속
- **Model-Free Control의 수렴 조건 요약** — Tabular + RM + 충분한 탐험 ⇒ 수렴, linear on-policy ⇒ 수렴, 그 외 발산 가능, 다음 레포(Deep RL)로 bridge

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **33개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 알고리즘이 필수 baseline인가
## 📐 수학적 선행 조건 (RL Foundations, Prob, Stats 참조)
## 📖 직관적 이해
   — Toy MDP에서 update step 추적
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Robbins-Monro, Q-Learning convergence, Double Q-Learning bias
## 💻 NumPy 구현 검증
   — Gridworld, FrozenLake, CliffWalk에서 모든 알고리즘
   — 수렴 곡선, bias-variance 시각화
## 🔗 후속 레포와의 연결
   — Deep RL, Policy Gradient
## ⚖️ 가정과 한계
   — Tabular, finite MDP, i.i.d.
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Cliff Walking 예제** — Sutton & Barto의 대표 예제, SARSA vs Q-Learning 비교 실험
2. **수렴 curve** — $\|Q_t - Q^*\|$를 sample 수 대비, log-scale
3. **Bias-variance plot** — MC vs TD(0) vs n-step의 MSE 분해
4. **Eligibility trace 시각화** — 각 state의 $e_t$가 시간에 따라 어떻게 업데이트되는지
5. **Double Q-Learning vs Q-Learning** — Roulette 예제 (van Hasselt 2010 Fig 1)에서 과대추정 비교
6. **Deadly Triad 재현** — Baird's counterexample로 linear FA off-policy 발산 시각화

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
matplotlib==3.8.0
gymnasium==0.29.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Q-Learning, SARSA, Double Q-Learning 비교 on Cliff Walking)
import numpy as np
import matplotlib.pyplot as plt
import gymnasium as gym

def epsilon_greedy(Q, s, epsilon, n_a):
    if np.random.rand() < epsilon:
        return np.random.randint(n_a)
    return np.argmax(Q[s])

def q_learning(env, n_episodes=500, alpha=0.5, gamma=1.0, epsilon=0.1):
    Q = np.zeros((env.observation_space.n, env.action_space.n))
    rewards = []
    for ep in range(n_episodes):
        s, _ = env.reset()
        total = 0; done = False
        while not done:
            a = epsilon_greedy(Q, s, epsilon, env.action_space.n)
            s_next, r, done, trunc, _ = env.step(a)
            # Q-Learning: off-policy, target uses max
            Q[s, a] += alpha * (r + gamma * np.max(Q[s_next]) - Q[s, a])
            s = s_next; total += r
            if trunc: done = True
        rewards.append(total)
    return Q, rewards

def sarsa(env, n_episodes=500, alpha=0.5, gamma=1.0, epsilon=0.1):
    Q = np.zeros((env.observation_space.n, env.action_space.n))
    rewards = []
    for ep in range(n_episodes):
        s, _ = env.reset()
        a = epsilon_greedy(Q, s, epsilon, env.action_space.n)
        total = 0; done = False
        while not done:
            s_next, r, done, trunc, _ = env.step(a)
            a_next = epsilon_greedy(Q, s_next, epsilon, env.action_space.n)
            # SARSA: on-policy, target uses actual next action
            Q[s, a] += alpha * (r + gamma * Q[s_next, a_next] - Q[s, a])
            s, a = s_next, a_next; total += r
            if trunc: done = True
        rewards.append(total)
    return Q, rewards

def double_q_learning(env, n_episodes=500, alpha=0.5, gamma=1.0, epsilon=0.1):
    QA = np.zeros((env.observation_space.n, env.action_space.n))
    QB = np.zeros_like(QA)
    rewards = []
    for ep in range(n_episodes):
        s, _ = env.reset()
        total = 0; done = False
        while not done:
            # Behavior: epsilon-greedy on QA+QB
            a = epsilon_greedy(QA + QB, s, epsilon, env.action_space.n)
            s_next, r, done, trunc, _ = env.step(a)
            if np.random.rand() < 0.5:
                # Update QA using argmax from QA, value from QB
                a_star = np.argmax(QA[s_next])
                QA[s, a] += alpha * (r + gamma * QB[s_next, a_star] - QA[s, a])
            else:
                a_star = np.argmax(QB[s_next])
                QB[s, a] += alpha * (r + gamma * QA[s_next, a_star] - QB[s, a])
            s = s_next; total += r
            if trunc: done = True
        rewards.append(total)
    return (QA + QB) / 2, rewards

# Compare on CliffWalking
env = gym.make('CliffWalking-v0')
np.random.seed(42)
_, r_q = q_learning(env)
np.random.seed(42)
_, r_s = sarsa(env)
np.random.seed(42)
_, r_dq = double_q_learning(env)

# Sliding window average
def smooth(x, w=10): return np.convolve(x, np.ones(w)/w, mode='valid')

plt.figure(figsize=(10, 5))
plt.plot(smooth(r_q), label='Q-Learning')
plt.plot(smooth(r_s), label='SARSA')
plt.plot(smooth(r_dq), label='Double Q-Learning')
plt.xlabel('Episode'); plt.ylabel('Total Reward (smoothed)')
plt.title('Cliff Walking: SARSA takes safer path (higher average), Q-Learning learns optimal but risky')
plt.legend(); plt.ylim(-100, 0); plt.show()

# Robbins-Monro conditions: alpha_t = 1/t satisfies both
# Demonstrate convergence rate
alphas = [('const 0.1', 0.1), ('1/t', None), ('1/sqrt(t)', None)]
# Plot Q[0, 0] trajectory for each schedule
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "RL Foundations, Prob, Stats, Stoch 선행 필수"
   - "Tabular → Linear FA → Deep (다음)의 진행"
   - Robbins-Monro를 RL의 핵심 수학으로 강조
3. **챕터별 문서 작성**: Setup → MC → TD → Q-Learning → n-step·Trace → Actor-Critic → 실전

---

## 📚 참고 자료

- **Reinforcement Learning: An Introduction** (Sutton & Barto 2018) — 5~12장
- **Stochastic Approximation** (Benveniste, Métivier, Priouret 1990)
- **Learning from Delayed Rewards** (Watkins 1989) — Q-Learning 박사논문
- **Q-Learning** (Watkins & Dayan 1992) — 수렴 증명
- **Convergence Results for Single-Step On-Policy Reinforcement-Learning Algorithms** (Singh et al. 2000)
- **On the Convergence of Stochastic Iterative Dynamic Programming Algorithms** (Jaakkola, Jordan, Singh 1994)
- **Double Q-Learning** (van Hasselt 2010)
- **An Analysis of TD(0) with Function Approximation** (Tsitsiklis & Van Roy 1997)
- **Policy Invariance Under Reward Transformations** (Ng, Harada, Russell 1999)

---

## 💡 핵심 분석 대상

```
Model-Free RL의 지도

───── Setting ─────

Model-based: P, R 알려짐 → DP
Model-free: 샘플 (s,a,r,s')만

Exploration vs Exploitation:
  ε-greedy, Boltzmann, UCB

───── Monte Carlo ─────

V^π(s) ≈ (1/N) Σ G_t over visits to s

Variants:
  First-visit: first time only
  Every-visit: all occurrences

Convergence: LLN
  Rate: O(1/√N)
  Unbiased, high variance

Control with ES (Exploring Starts):
  모든 (s,a) 시작 → 수렴

Off-policy:
  Importance Sampling ratio
  ρ = Π π/b
  ├── Ordinary IS: unbiased, high variance
  └── Weighted IS: biased, low variance

───── TD(0) ─────

V(s) ← V(s) + α[R + γV(s') - V(s)]
                  └── TD error δ ─┘

MC vs TD:
  MC: biased=0, var=σ²(G_t)
  TD: bias≠0, var ↓ (bootstrap)

Convergence (Tsitsiklis 1997):
  Tabular: ✓
  Linear on-policy: ✓
  Linear off-policy: ✗ 가능

───── SARSA (On-policy) ─────

Q(s,a) ← Q + α[R + γQ(s',a') - Q]
  a' from current policy

Cliff Walking:
  "안전" 경로 학습
  (ε-greedy의 탐험 고려)

───── Q-Learning (Off-policy) ─────

Q(s,a) ← Q + α[R + γ max_{a'} Q(s',a') - Q]
                      └── behavior 무관 ┘

Target: Q*(s,a) = r + γ max Q*
        ↑ Bellman Optimality

Convergence (Watkins-Dayan 1992):
  조건:
    1. 모든 (s,a) infinitely often 방문
    2. Σα = ∞, Σα² < ∞ (Robbins-Monro)
    3. Bounded reward
  ⇒ Q_t → Q* a.s.

Proof sketch:
  T* is γ-contraction (RL Foundations)
  Stochastic approximation:
    Q_{t+1} = (1-α) Q_t + α(T* Q_t + noise)
  → converges to T* 고정점

Cliff Walking:
  "최적" 경로 학습
  (하지만 ε-greedy에서 위험)

───── Maximization Bias ─────

𝔼[max X_i] ≥ max 𝔼[X_i]
  Jensen's inequality

→ Q-Learning이 Q-value 과대추정
  Estimation noise + max → positive bias

Double Q-Learning (van Hasselt 2010):
  Q^A, Q^B 두 개
  a* = argmax Q^A(s', ·)
  target = Q^B(s', a*)
  → argmax과 value 분리
  → bias 감소

───── n-step TD ─────

n-step return:
  G_{t:t+n} = R_{t+1} + γR_{t+2} + ... + γ^{n-1}R_{t+n} + γ^n V(S_{t+n})

n=1: TD(0)
n=∞: MC

Bias-variance:
  n ↑ → bias ↓, variance ↑

───── TD(λ) ─────

λ-return:
  G_t^λ = (1-λ) Σ_{n=1}^∞ λ^{n-1} G_{t:t+n}
  λ=0: TD(0)
  λ=1: MC

Forward view:
  각 시점에서 미래 G_t^λ 업데이트
  (episodic only)

Backward view (online):
  eligibility trace e_t(s):
    e_t(s) = γλ e_{t-1}(s) + 𝟙[S_t=s]
  
  V(s) ← V(s) + α δ_t e_t(s)  (all s)

Forward ≡ Backward (증명 필요!)

Trace types:
  Accumulating: += 1
  Replacing: := 1
  Dutch: 정확한 equivalence

───── Actor-Critic 시작 ─────

Actor: policy π_θ
Critic: V or Q

Actor-Critic 업데이트:
  Critic: TD(0) on V^π
  Actor: policy gradient
  
→ Policy Gradient 레포로

───── 실전 이슈 ─────

Deadly Triad (Baird 1995):
  1. Off-policy
  2. Bootstrapping
  3. FA
  동시 → 발산 가능

Experience Replay:
  Buffer에서 i.i.d. sample
  → Deep RL의 핵심

Reward Shaping:
  F(s,s') = γΦ(s') - Φ(s)
  → optimal policy 불변 (Ng 1999)

───── 레포 간 연결 ─────

RL Foundations (앞):
  Bellman, DP, Contraction

Probability (Layer 0):
  Martingale convergence
  집중부등식

Stochastic Proc (Layer 0):
  Markov chain ergodicity
  Stationary distribution

Stats (Layer 0):
  Stochastic approximation
  Robbins-Monro

Deep RL (다음):
  Experience Replay + DQN
  FA로 scale-up

Policy Gradient (2개 뒤):
  Actor-Critic 심화
  Policy Gradient theorem
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명 (3~4줄)
- 전체 문서 개수 확인 (33개 목표)
- Python + NumPy + Gymnasium 실험 환경
- RL Foundations, Prob, Stats, Stoch 레포의 참조 관계
- Deep RL로 이어지는 Experience Replay 개념

**준비됐으면 1단계 구조 설계부터 시작해줘!**
