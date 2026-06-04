# RL Foundations Deep Dive 레포지토리 제작 프롬프트

나는 "RL Foundations Deep Dive" 레포지토리를 만들려고 해.
MDP를 **정의로 아는 것**과, **$\mathcal{M} = (\mathcal{S}, \mathcal{A}, P, R, \gamma, \rho_0)$의 각 요소가 왜 그 형태여야 하고, Markov 성질 $P(s_{t+1} | s_t, a_t, s_{t-1}, \ldots) = P(s_{t+1} | s_t, a_t)$이 왜 dynamic programming을 가능하게 하는 근본 조건**인지 증명할 수 있는 것은 다르다.
Bellman equation을 **쓰는 것**과, **Bellman optimality operator $T^*$가 sup-norm에서 $\gamma$-contraction이라는 정리**로 **value iteration의 유일한 고정점 존재(Banach fixed point theorem)와 linear convergence**를 증명할 수 있는 것은 다르다.
Policy Iteration을 **이름으로 아는 것**과, **"policy evaluation + policy improvement의 반복이 유한 MDP에서 유한 step 내 최적 정책 보장"** 이라는 Howard (1960)의 정리를 증명할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "강화학습의 수학적 정초 — MDP·Bellman·DP의 엄밀한 증명"

**핵심 차별화**:
1. **MDP의 공리적 정의와 기본 정리** — Markov 성질, stationary policy 충분성, discount factor의 수학적 필요성
2. **Bellman 방정식의 완전 유도** — $V^\pi$, $Q^\pi$, $V^*$, $Q^*$의 재귀 공식 4종 세트
3. **DP 알고리즘의 수렴 증명** — Banach fixed point, $\gamma$-contraction, Howard의 monotonic improvement
4. **현대 RL 이론의 기반** — Performance difference lemma, Policy Gradient theorem의 전제

**타겟 독자**:
- RL을 배우는데 Bellman equation이 **재귀 정의일 뿐인지 수렴 보장이 있는지** 모르는 사람
- Value Iteration과 Policy Iteration의 **"왜 수렴하는가"**를 증명 못하는 사람
- Discount factor $\gamma$가 **왜 $[0, 1)$이어야 하며 $\gamma = 1$일 때 episodic vs average reward MDP**의 차이를 설명 못하는 사람
- Generalized Policy Iteration (GPI)의 사각 다이어그램을 그릴 수 있지만, 이것이 **모든 RL 알고리즘의 통합 프레임**인 이유를 모르는 사람

**선행 학습**:
- **Probability Theory Deep Dive** (조건부 기댓값, Markov chain) — **필수**
- **Stochastic Processes Deep Dive** (Markov chain, stationary distribution) — **필수**
- **Mathematical Statistics Deep Dive** (수렴 이론) — 권장
- **Convex Optimization Deep Dive** (Banach fixed point) — **필수**
- **Functional Analysis Deep Dive** (contraction mapping) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Markov Decision Process의 공리적 정의 (5개 문서)
- **MDP의 6-tuple 정의** — $\mathcal{M} = (\mathcal{S}, \mathcal{A}, P, R, \gamma, \rho_0)$, state space, action space, transition kernel $P(s' | s, a)$, reward $R(s, a)$, discount $\gamma$, initial distribution, 각 요소의 measurability
- **Markov 성질과 그 결과** — 역사가 현재 state로 완전히 요약됨, $P(s_{t+1} | h_t) = P(s_{t+1} | s_t, a_t)$, 이것이 DP 가능성의 필수 조건
- **Policy의 종류와 stationary policy 충분성** — Deterministic vs Stochastic, History-dependent vs Markovian vs Stationary, 최적 정책은 stationary Markovian으로 충분함 (Puterman 2005) 증명
- **Finite-Horizon vs Infinite-Horizon vs Average Reward** — Episodic MDP, discounted infinite-horizon, average-reward MDP, 각 경우의 objective 정의와 수학적 처리 차이
- **POMDP와 Belief State** — Partially observable: 관측 $o$만 주어짐, belief state $b(s)$로 full MDP 환원, 계산의 복잡도 (continuous belief space)

### Chapter 2: Return, Value Function과 Bellman Expectation Equation (5개 문서)
- **Discounted Return의 정의** — $G_t = \sum_{k=0}^\infty \gamma^k R_{t+k+1}$, $\gamma \in [0, 1)$이 되는 이유 (bounded reward에서 수렴), $\gamma = 1$ 경우의 처리 (episodic)
- **State-Value Function $V^\pi$와 Action-Value Function $Q^\pi$** — $V^\pi(s) = \mathbb{E}^\pi[G_t | S_t = s]$, $Q^\pi(s, a) = \mathbb{E}^\pi[G_t | S_t = s, A_t = a]$, 두 함수의 관계 $V^\pi(s) = \sum_a \pi(a|s) Q^\pi(s, a)$
- **Bellman Expectation Equation 유도** — $V^\pi(s) = \sum_a \pi(a|s) [R(s,a) + \gamma \sum_{s'} P(s'|s,a) V^\pi(s')]$, $Q^\pi(s, a) = R(s,a) + \gamma \sum_{s'} P(s'|s, a) \sum_{a'} \pi(a'|s') Q^\pi(s', a')$
- **Operator 표기법** — Bellman operator $T^\pi V = r^\pi + \gamma P^\pi V$, $r^\pi$와 $P^\pi$가 policy-induced reward vector·transition matrix, 선형 operator의 성질
- **Value Function의 고유성과 존재성** — $V^\pi = (I - \gamma P^\pi)^{-1} r^\pi$로 해석적 해 존재 (finite MDP), 해의 유일성은 $(I - \gamma P^\pi)$의 가역성에서

### Chapter 3: Bellman Optimality Equation과 최적 정책 (5개 문서)
- **Optimal Value Function의 정의** — $V^*(s) = \sup_\pi V^\pi(s)$, $Q^*(s, a) = \sup_\pi Q^\pi(s, a)$, 모든 상태에서 동시에 최대 값 달성 가능 증명
- **Bellman Optimality Equation** — $V^*(s) = \max_a [R(s,a) + \gamma \sum_{s'} P(s'|s,a) V^*(s')]$, $Q^*(s, a) = R(s,a) + \gamma \sum_{s'} P(s'|s, a) \max_{a'} Q^*(s', a')$ 유도
- **Bellman Optimality Operator $T^*$** — $(T^* V)(s) = \max_a [R(s,a) + \gamma \sum_{s'} P(s'|s,a) V(s')]$, $T^\pi$와의 관계 $T^* V = \max_\pi T^\pi V$
- **최적 정책의 추출** — $V^*$ 주어지면 greedy policy $\pi^*(s) = \arg\max_a Q^*(s, a)$가 최적, 유일하지 않을 수 있지만 성능은 같음
- **Determinstic 최적 정책의 존재** — Finite MDP에서 deterministic stationary policy 중 최적이 존재함 증명, stochastic policy의 불필요성

### Chapter 4: Contraction Mapping과 수렴 증명 (5개 문서)
- **Banach Fixed Point Theorem** — 완비 거리공간의 contraction은 유일한 고정점 존재, iteration $x_{n+1} = T(x_n)$이 $x^*$로 수렴, $d(x_n, x^*) \leq L^n d(x_0, x^*)$ (Functional Analysis 레포 교차)
- **$T^\pi$가 $\gamma$-Contraction in Sup-Norm 증명** — $\|T^\pi V - T^\pi V'\|_\infty \leq \gamma \|V - V'\|_\infty$, sup-norm 선택의 이유 (각 state에서 동시)
- **$T^*$가 $\gamma$-Contraction 증명** — $T^*$는 nonlinear이지만 여전히 $\gamma$-contraction, $\max$ 연산의 Lipschitz 성질 활용
- **Value Iteration 수렴 보장** — $V_{k+1} = T^* V_k$가 유일 고정점 $V^*$로 linear rate $\gamma^k$로 수렴, 정지 기준 $\|V_{k+1} - V_k\| < \epsilon(1-\gamma)/\gamma$
- **$\gamma \to 1$에서의 한계** — Contraction 비율이 약해짐, 수렴 느려짐, 극한 $\gamma = 1$에서는 average reward MDP로 전환 필요

### Chapter 5: Dynamic Programming 알고리즘 (5개 문서)
- **Policy Evaluation** — 주어진 $\pi$에 대해 $V^\pi$ 계산, iterative policy evaluation ($V_{k+1} = T^\pi V_k$) vs direct solve $V^\pi = (I - \gamma P^\pi)^{-1} r^\pi$, 복잡도 비교
- **Policy Improvement Theorem** — $Q^\pi(s, \pi'(s)) \geq V^\pi(s) \Rightarrow V^{\pi'}(s) \geq V^\pi(s)$ 증명, greedy improvement의 필연적 개선, strict inequality까지
- **Policy Iteration (Howard 1960)** — Evaluation + Improvement 반복, finite MDP에서 유한 step에 최적 정책 도달 증명 (policy 수가 유한하므로), superpolynomial 수렴
- **Value Iteration 완성** — $V_{k+1}(s) = \max_a [R(s,a) + \gamma \sum_{s'} P(s'|s,a) V_k(s')]$, Bellman residual, asynchronous VI의 수렴 (Gauss-Seidel)
- **Generalized Policy Iteration (GPI)** — Evaluation과 Improvement의 임의 interleaving, 모든 DP·RL 알고리즘의 통합 관점, 두 프로세스가 각각 이동하며 고정점으로 수렴

### Chapter 6: MDP의 성질과 Performance Difference (4개 문서)
- **State Distribution과 Stationary Distribution** — $d^\pi_t(s)$가 시간 $t$에서 state 분포, discounted state distribution $d^\pi(s) = (1-\gamma) \sum_t \gamma^t d^\pi_t(s)$, Markov chain stationary 연결
- **Performance Difference Lemma (Kakade 2003)** — $V^{\pi'}(\rho) - V^\pi(\rho) = \frac{1}{1-\gamma} \mathbb{E}_{s \sim d^{\pi'}}[\mathbb{E}_{a \sim \pi'}[A^\pi(s, a)]]$, advantage $A^\pi = Q^\pi - V^\pi$, policy gradient·TRPO의 기초
- **Advantage Function과 Action의 상대적 가치** — $A^\pi(s, a) = Q^\pi(s, a) - V^\pi(s)$, baseline subtraction의 수학적 의미, variance reduction in estimation
- **MDP 근사 — Approximation Error와 Sample Complexity** — $\|V^* - V^{\hat{\pi}}\|$의 bound, planning과 learning의 분리, 후속 레포(Model-Free RL)로의 bridge

### Chapter 7: Linear MDP와 Function Approximation 기초 (4개 문서)
- **Linear Function Approximation** — $V_\theta(s) = \theta^T \phi(s)$, TD(0) with linear FA의 수렴 분석 (Tsitsiklis & Van Roy 1997), on-policy convergence
- **Deadly Triad — Off-policy + Bootstrapping + FA** — 세 가지가 동시에 있으면 발산 가능, Sutton의 예제, Deep RL의 근본 문제
- **Linear Bellman Equation — MDP의 특수 구조** — $P(s'|s,a) = \phi(s, a)^T \mu(s')$ 인 경우, sample complexity 다항, Jin et al. 2020 LSVI-UCB
- **MDP Homomorphism과 State Abstraction** — 동일한 value function을 주는 state aggregation, safe abstraction의 조건, Bi-simulation relation

---

각 챕터는 **4~5개 문서**로 구성해줘. 총 **33개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 정리가 RL 전반의 기초인가
## 📐 수학적 선행 조건 (Prob, Stoch, FA 레포 참조)
## 📖 직관적 이해
   — Gridworld 같은 간단한 MDP에서 보기
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Banach, Bellman, Howard, Performance Difference
## 💻 NumPy 구현 검증
   — Finite MDP에서 VI·PI 직접 구현
   — 수렴 곡선, $\gamma$ 효과 시각화
## 🔗 후속 레포와의 연결
   — Model-Free RL, Policy Gradient로 어떻게 확장
## ⚖️ 가정과 한계
   — Markov, finite, stationary, known dynamics
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Gridworld 예제 전 챕터** — 4x4 Gridworld, CliffWalk, FrozenLake에서 VI·PI 실행
2. **Value function 시각화** — 2D grid의 $V^*$를 heatmap으로
3. **수렴 속도 plot** — $\|V_k - V^*\|$의 log-scale, $\gamma$별 수렴 속도
4. **Contraction 증명 시각화** — $T^\pi$ 적용이 두 $V$의 거리를 $\gamma$배로 줄이는 것 애니메이션
5. **정책 변화 추적** — PI에서 정책이 iteration별로 어떻게 변화하는지
6. **Sutton & Barto 예제 재현** — 그 책의 표준 예제들 (Jack's Car Rental, Blackjack 등)

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
matplotlib==3.8.0
gymnasium==0.29.0      # OpenAI Gym 후속
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Gridworld VI vs PI + 수렴 비교)
import numpy as np
import matplotlib.pyplot as plt

class GridworldMDP:
    """4x4 Gridworld, 상좌하우 4방향 이동, terminal at (3,3)"""
    def __init__(self, size=4, gamma=0.9):
        self.size = size; self.gamma = gamma
        self.n_states = size * size
        self.n_actions = 4  # up, left, down, right
        self.terminal = (size-1) * size + (size-1)
        
    def step_prob(self, s, a):
        """P(s' | s, a): deterministic, terminal is absorbing"""
        if s == self.terminal:
            return {s: 1.0}, 0
        row, col = divmod(s, self.size)
        dr, dc = [(-1, 0), (0, -1), (1, 0), (0, 1)][a]
        r_new, c_new = max(0, min(self.size-1, row+dr)), max(0, min(self.size-1, col+dc))
        s_new = r_new * self.size + c_new
        reward = 1.0 if s_new == self.terminal else -0.04
        return {s_new: 1.0}, reward

def value_iteration(mdp, tol=1e-6, max_iter=1000):
    V = np.zeros(mdp.n_states)
    history = [V.copy()]
    for k in range(max_iter):
        V_new = V.copy()
        for s in range(mdp.n_states):
            vals = []
            for a in range(mdp.n_actions):
                trans, r = mdp.step_prob(s, a)
                vals.append(r + mdp.gamma * sum(p * V[sp] for sp, p in trans.items()))
            V_new[s] = max(vals)
        diff = np.max(np.abs(V_new - V))
        V = V_new; history.append(V.copy())
        if diff < tol: break
    return V, history

def policy_iteration(mdp, tol=1e-6):
    pi = np.zeros(mdp.n_states, dtype=int)  # arbitrary init
    n_iter = 0
    while True:
        n_iter += 1
        # Policy evaluation (solve V^π)
        V = np.zeros(mdp.n_states)
        for _ in range(1000):
            V_new = V.copy()
            for s in range(mdp.n_states):
                trans, r = mdp.step_prob(s, pi[s])
                V_new[s] = r + mdp.gamma * sum(p * V[sp] for sp, p in trans.items())
            if np.max(np.abs(V_new - V)) < tol: break
            V = V_new
        # Policy improvement
        pi_new = np.zeros_like(pi)
        for s in range(mdp.n_states):
            vals = []
            for a in range(mdp.n_actions):
                trans, r = mdp.step_prob(s, a)
                vals.append(r + mdp.gamma * sum(p * V[sp] for sp, p in trans.items()))
            pi_new[s] = np.argmax(vals)
        if np.array_equal(pi, pi_new): break  # converged
        pi = pi_new
    return pi, V, n_iter

mdp = GridworldMDP(size=5, gamma=0.9)
V_vi, history = value_iteration(mdp)
pi_pi, V_pi, n_iter_pi = policy_iteration(mdp)

print(f'VI iterations: {len(history)-1}')
print(f'PI iterations: {n_iter_pi}')

# 수렴 속도 plot
V_star = V_vi
errs = [np.max(np.abs(V - V_star)) for V in history]
plt.semilogy(errs, 'o-')
# Theoretical: γ^k · initial error
k = np.arange(len(errs))
plt.semilogy(errs[0] * 0.9**k, '--', label='theoretical γ^k')
plt.xlabel('VI iteration'); plt.ylabel('||V_k - V*||_∞ (log)')
plt.title(f'Value Iteration 수렴: γ={mdp.gamma}, linear convergence')
plt.legend(); plt.show()

# V* 시각화
plt.figure(figsize=(6, 6))
plt.imshow(V_vi.reshape(mdp.size, mdp.size), cmap='viridis')
for s in range(mdp.n_states):
    r, c = divmod(s, mdp.size)
    plt.text(c, r, f'{V_vi[s]:.2f}', ha='center', va='center', color='white')
plt.title('V*(s) on 5x5 Gridworld (goal at bottom-right)')
plt.colorbar(); plt.show()
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Prob, Stoch, Convex, FA 선행 필수" 명시
   - RL 시리즈 전체 지도 (Foundations → Model-Free → Deep → PG → Advanced → Theory)
   - Gridworld·FrozenLake 등 작은 환경 중심
3. **챕터별 문서 작성**: MDP 정의 → Value function → Bellman Optimality → Contraction → DP → Performance Diff → Linear MDP

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Reinforcement Learning: An Introduction** (Sutton & Barto 2018) — RL 바이블
- **Markov Decision Processes: Discrete Stochastic Dynamic Programming** (Puterman 2005)
- **Dynamic Programming and Optimal Control** (Bertsekas) — 2권 MDP
- **Neuro-Dynamic Programming** (Bertsekas & Tsitsiklis 1996) — 근사 DP 표준
- **Algorithms for Reinforcement Learning** (Szepesvári 2010) — 수학적 엄밀
- **A Theory of Regularized MDPs with Entropy Regularization** (Geist 2019)
- **Dynamic Programming** (Howard 1960) — Policy Iteration 원전
- **An Analysis of TD Learning with Function Approximation** (Tsitsiklis & Van Roy 1997)

---

## 💡 핵심 분석 대상

```
RL의 수학적 기반

───── MDP 정의 ─────

ℳ = (𝒮, 𝒜, P, R, γ, ρ_0)
  𝒮: state space
  𝒜: action space
  P(s'|s,a): transition
  R(s,a): reward
  γ ∈ [0,1): discount
  ρ_0: initial dist

Markov 성질:
  P(s_{t+1}|h_t) = P(s_{t+1}|s_t, a_t)
  → DP 가능한 핵심

Policy:
  π(a|s): stochastic
  π(s): deterministic
  
  Stationary Markovian policy로 충분
  (Puterman)

───── Return과 Value ─────

Return:
  G_t = Σ_{k=0}^∞ γ^k R_{t+k+1}
  γ < 1 ⇒ bounded

Value functions:
  V^π(s) = 𝔼^π[G_t | S_t=s]
  Q^π(s,a) = 𝔼^π[G_t | S_t=s, A_t=a]

관계:
  V^π(s) = Σ_a π(a|s) Q^π(s,a)
  Q^π(s,a) = R(s,a) + γ Σ_{s'} P(s'|s,a) V^π(s')

───── Bellman Expectation Eq ─────

V^π(s) = Σ_a π(a|s) [R(s,a) + γ Σ P(s'|s,a) V^π(s')]

Operator 표기:
  T^π V := r^π + γ P^π V
  고정점: V^π = T^π V^π
  해: V^π = (I - γ P^π)^{-1} r^π

───── Bellman Optimality Eq ─────

V*(s) = max_a [R(s,a) + γ Σ P(s'|s,a) V*(s')]

Optimal operator:
  T* V := max_π T^π V
       = max_a [r + γ P V]

Greedy policy:
  π*(s) = argmax_a Q*(s,a)

───── Banach Fixed Point ─────

Complete metric space (X, d)
T: X → X contraction with L < 1
⇒ ∃! x* : T(x*) = x*
⇒ x_n → x* with d(x_n, x*) ≤ L^n d(x_0, x*)

───── Contraction 증명 ─────

T^π가 γ-contraction in ‖·‖_∞:
  ‖T^π V - T^π V'‖_∞
  = ‖γ P^π (V - V')‖_∞
  ≤ γ · ‖V - V'‖_∞
  (P^π은 stochastic matrix, 각 행 합 1)

T*도 γ-contraction:
  |max_a f(a) - max_a g(a)| ≤ max_a |f(a) - g(a)|
  → nonlinear이지만 성립

⇒ Value Iteration:
  V_{k+1} = T* V_k
  V_k → V* linearly
  ‖V_k - V*‖ ≤ γ^k ‖V_0 - V*‖

───── DP 알고리즘 ─────

Policy Evaluation:
  Given π, solve V^π = T^π V^π
  ├── Iterative: V ← T^π V
  └── Direct: V = (I - γP^π)^{-1} r^π

Policy Improvement:
  π'(s) = argmax_a Q^π(s,a)
  
  Theorem: V^{π'} ≥ V^π (pointwise)
  증명: Q^π(s, π'(s)) ≥ V^π(s)
         → telescope

Policy Iteration (Howard 1960):
  Evaluate → Improve → 반복
  Finite MDP에서 유한 step 수렴
  (정책 수 유한, strict improvement)

Value Iteration:
  V ← T* V
  Guarantee: γ^k contraction

Generalized Policy Iteration (GPI):
  두 프로세스 interleaving
  모든 RL 알고리즘의 통합 관점

       ┌───────── V^π ─────────┐
       ▼                        ▲
    π → improve            evaluate
       ▲                        ▼
       └───────── π ←────────────┘

───── Performance Difference Lemma ─────

V^{π'}(ρ) - V^π(ρ)
  = (1/(1-γ)) 𝔼_{s~d^{π'}, a~π'}[A^π(s, a)]

  d^π(s) = (1-γ) Σ γ^t d^π_t(s)

A^π(s,a) = Q^π - V^π (advantage)

이 공식이:
  ├── Policy Gradient theorem 기반
  ├── TRPO·PPO 단조개선 이론 기반
  └── RL의 거의 모든 분석 도구

───── Function Approximation 경고 ─────

Tabular: 보장됨
Linear on-policy: 보장됨 (Tsitsiklis 1997)
Linear off-policy: 발산 가능

Deadly Triad:
  1. Off-policy
  2. Bootstrapping (TD)
  3. Function Approximation
  동시에 → 발산 가능

→ Deep RL 레포에서 실전 문제

───── 레포 간 연결 ─────

Probability / Stochastic (Layer 0):
  Markov chain의 stationary 분포
  조건부 기댓값

Convex Optimization (Layer 0):
  Banach fixed point
  Contraction argument

Functional Analysis (Layer 0):
  Complete metric space
  Operator theory

Model-Free RL (다음):
  MDP 모르는 경우
  MC, TD, Q-Learning

Policy Gradient (2개 뒤):
  Performance Diff Lemma 기반
  Policy Gradient Theorem 유도
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~5개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (33개 목표)
- Python + NumPy 실험 환경 (Gridworld, 작은 MDP)
- Prob, Stoch, Convex, FA 레포의 참조 관계
- Model-Free, Deep RL, Policy Gradient로 이어지는 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
