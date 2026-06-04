# Deep RL Deep Dive 레포지토리 제작 프롬프트

나는 "Deep RL Deep Dive" 레포지토리를 만들려고 해.
DQN이 **Atari에서 인간 수준을 달성한 사실**을 아는 것과, **Mnih et al. (2015)의 3가지 핵심 trick — (1) Experience Replay가 i.i.d. 가정 근사, (2) Target Network가 $T^*$ bootstrapping의 안정화, (3) Reward Clipping이 Lipschitz 보장 — 각각의 수학적 정당성**을 증명할 수 있는 것은 다르다.
Double DQN의 필요성을 **"과대추정 감소"로 아는 것**과, **$\mathbb{E}[\max_a Q(s, a; \theta) + \text{noise}] \geq \max_a Q(s, a; \theta)$ (Jensen's inequality)로 인한 positive bias를 van Hasselt (2016)가 어떻게 online/target network 분리로 해결**하는지 유도할 수 있는 것은 다르다.
Prioritized Experience Replay를 **쓰는 것**과, **TD error $|\delta_i|$에 비례한 sampling이 importance sampling weight $(1/(Np_i))^\beta$로 보정되지 않으면 biased estimator가 되는 이유**를 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "Tabular Q-Learning에서 Atari·Go까지 — Deep RL의 trick들을 수학적으로 정당화"

**핵심 차별화**:
1. **DQN의 3가지 trick의 수학적 분석** — Experience Replay(correlation 감소), Target Network(bootstrap 안정), Reward Clipping(gradient bound)
2. **Double DQN의 bias 감소 정리** — Jensen + max bias → online/target 분리로 해결
3. **Rainbow의 6가지 구성요소 ablation** — 각 기법의 개별 기여 측정 (Hessel 2018 Fig 4 재현)
4. **Distributional RL — C51, QR-DQN** — Expected return 대신 return 분포 자체 학습, Bellemare 2017의 C51

**타겟 독자**:
- DQN을 학습했는데 **왜 일반 Q-Learning + DNN은 발산**하는지 (Deadly Triad)와 DQN의 trick이 각각 어떤 문제를 해결하는지 모르는 사람
- Rainbow를 쓰지만 **6개 구성요소(Double, Dueling, PER, Multi-step, Distributional, Noisy)** 각각의 동기와 이론을 모르는 사람
- Prioritized Experience Replay의 $\alpha, \beta$ 하이퍼파라미터가 **각각 priority exponent와 IS correction exponent**인 이유를 모르는 사람
- Distributional RL이 Expected value보다 유리한 이유(**return distribution의 risk 정보**)를 모르는 사람
- Noisy Networks가 $\epsilon$-greedy 대신 **왜 parameter-space exploration**이 효과적인지를 이해 못하는 사람

**선행 학습**:
- **Model-Free RL Deep Dive** (Q-Learning, TD, Double Q) — **필수**
- **Neural Network Theory Deep Dive** (Backprop, 아키텍처) — **필수**
- **Optimization Theory Deep Dive** (SGD, Adam, 훈련 안정성) — **필수**
- **CNN Deep Dive** (이미지 input 처리) — **필수** (Atari)
- **Probability Theory Deep Dive** (분포, 집중부등식) — Distributional RL에 필요

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Deep RL의 동기와 Deadly Triad (4개 문서)
- **Tabular RL의 한계** — State space가 크면 $|S| \times |A|$ table 불가능 (Atari: $256^{210 \times 160 \times 3}$ 가능 frame), feature engineering 없이 image 처리 필요
- **Deadly Triad 재확인 — Baird's Counterexample** — Off-policy + Bootstrapping + FA → 발산 예시, 7-state star MDP에서 linear TD가 발산, NN + Q-Learning이 naive하게 실패하는 이유
- **Function Approximation의 Projection Error** — $\|\Pi T^* Q - Q\|$, $\Pi$가 function class에의 projection, Bellman residual과 projected Bellman equation
- **DNN as Function Approximator — 장점과 도전** — 대용량 feature learning, distributed representation, 하지만 non-stationary target과 correlation으로 발산 위험

### Chapter 2: DQN의 3가지 Trick (6개 문서)
- **DQN 알고리즘 전체 구조 (Mnih 2015)** — $Q(s, a; \theta)$ 학습, loss $L = \mathbb{E}_{(s, a, r, s') \sim D}[(r + \gamma \max_{a'} Q(s', a'; \theta^-) - Q(s, a; \theta))^2]$, frame stacking 4, ConvNet 아키텍처
- **Experience Replay의 역할** — Buffer $D$에서 uniform sampling, sample correlation 파괴 (i.i.d. 근사), sample efficiency ↑, data 재사용, buffer size 효과
- **Target Network의 필요성** — $\theta^-$ 고정하고 $\theta$만 업데이트, 주기 $C$ (보통 10000 steps)로 $\theta^- \leftarrow \theta$ 복사, bootstrap target의 안정화 증명
- **Target Network 없이 발산하는 이유 — Moving Target** — $Q$가 변할 때 target $r + \gamma \max Q$도 변함 → 불안정, target을 frozen하면 supervised regression 근사, contraction 근사 유지
- **Reward Clipping과 훈련 안정성** — $r \leftarrow \text{clip}(r, -1, 1)$, gradient scale 일정하게, Atari 게임 간 공통 하이퍼파라미터 가능, game-specific value range 문제
- **DQN 전체 pseudocode와 실전 팁** — Epsilon scheduling (1.0 → 0.1), Adam vs RMSProp, learning rate, replay buffer warmup, evaluation protocol

### Chapter 3: Double DQN과 과대추정 편향 (4개 문서)
- **Maximization Bias in Q-Learning** — $\hat{Q} = Q + \epsilon$ (noise) 일 때 $\mathbb{E}[\max_a \hat{Q}(s, a)] \geq \max_a \mathbb{E}[\hat{Q}(s, a)] = \max_a Q(s, a)$, Jensen's inequality의 결과
- **Double Q-Learning의 Tabular 해결 (van Hasselt 2010)** — 두 개의 $Q^A, Q^B$, one selects action, other evaluates, bias 감소 증명 (두 noise가 independent일 때 $\mathbb{E}[\max]$가 정확해짐)
- **Double DQN 공식 (van Hasselt 2016)** — Target: $r + \gamma Q(s', \arg\max_{a'} Q(s', a'; \theta); \theta^-)$, online network로 action 선택 + target network로 value 평가, 추가 network 불필요
- **Double DQN의 실증 효과** — Atari 6개 게임에서 Q-value 과대추정 측정, 최종 성능 향상 (Hessel 2018 bench), 특히 초기 학습 안정성

### Chapter 4: Dueling, Prioritized Replay, Multi-step (5개 문서)
- **Dueling Network (Wang et al. 2016)** — $Q(s, a) = V(s) + A(s, a) - \frac{1}{|A|} \sum_{a'} A(s, a')$, state-value와 advantage 분해, 많은 action에서 state value가 비슷할 때 유리, advantage centering의 identifiability
- **Prioritized Experience Replay (Schaul 2016)** — Uniform 대신 $p_i \propto |\delta_i|^\alpha$ 기반 sampling, 큰 TD error transition을 자주, $\alpha = 0$이 uniform, $\alpha = 1$이 greedy priority
- **PER의 Importance Sampling 보정** — Prioritized sampling이 biased → IS weight $w_i = (1/(N p_i))^\beta$로 보정, $\beta$가 anneal (0 → 1), variance와 bias의 trade-off
- **Multi-step Return in DQN** — $G_t^{(n)} = \sum_{k=0}^{n-1} \gamma^k R_{t+k+1} + \gamma^n \max Q(S_{t+n}, \cdot; \theta^-)$, on-policy 가정 위반이지만 실전 효과, $n = 3$ 등 선택
- **Noisy Networks (Fortunato 2018)** — $\epsilon$-greedy 대신 parameter에 noise 주입, $W = \mu + \sigma \odot \epsilon$, state-dependent exploration, 학습 가능한 noise scale

### Chapter 5: Distributional RL (5개 문서)
- **Return의 분포 관점** — $Z^\pi(s, a) = \sum_t \gamma^t R_{t+1}$가 random variable, $Q^\pi = \mathbb{E}[Z^\pi]$는 단지 평균, 전체 분포에 더 많은 정보
- **Distributional Bellman Equation** — $Z(s, a) \stackrel{D}{=} R(s, a) + \gamma Z(s', A')$, "$\stackrel{D}{=}$"는 distributional equality, expected Bellman의 distribution 일반화
- **C51 — Categorical Distributional RL (Bellemare 2017)** — 이산 support $\{z_0, \ldots, z_{N-1}\}$ on $[V_{\min}, V_{\max}]$, softmax probability로 표현, KL loss, projected distributional Bellman update
- **Quantile Regression DQN (QR-DQN, Dabney 2018)** — 분포를 quantile로 표현, $\tau_i = (2i-1)/(2N)$, Wasserstein 거리 최소화, Huber loss based quantile regression
- **Distributional RL의 이점과 실증** — Risk-aware control 가능 (CVaR), auxiliary prediction의 regularization 효과, Rainbow에서 중요 component

### Chapter 6: Rainbow와 통합 (4개 문서)
- **Rainbow DQN (Hessel 2018)** — 6가지 개선을 통합: Double + Dueling + PER + Multi-step + Distributional (C51) + Noisy, 각 component의 cumulative 효과
- **Ablation Study 분석** — 각 component 제거 시 성능 감소, Multi-step + Distributional + PER이 가장 중요, Double과 Dueling은 상대적으로 영향 작음
- **Rainbow 이후의 발전** — Ape-X (distributed replay, Horgan 2018), R2D2 (recurrent + replay, Kapturowski 2019), IMPALA (IPG with V-trace, Espeholt 2018)
- **MuZero — Model-based RL의 부활 (Schrittwieser 2020)** — Learning dynamics + reward + policy, AlphaZero의 모델 없는 버전, Atari·Go·Chess 모두 SOTA

### Chapter 7: 연속 행동공간과 Continuous Control (4개 문서)
- **Discrete vs Continuous Action — Q-Learning의 한계** — Continuous에서 $\max_a Q(s, a)$ 계산 어려움 (infinite $a$), DQN 직접 적용 불가
- **DDPG — Deterministic Policy Gradient (Lillicrap 2016)** — Actor $\mu_\theta(s)$가 deterministic policy, Critic $Q_\phi(s, a)$, $\nabla J \approx \mathbb{E}[\nabla_\theta \mu \cdot \nabla_a Q|_{a=\mu}]$, Silver 2014의 연속 버전
- **DDPG의 hyperparameter fragility** — 훈련 불안정, 하이퍼파라미터 민감, target soft update $\theta^- \leftarrow \tau \theta + (1-\tau) \theta^-$, exploration noise 설계
- **Continuous Control의 alternatives로의 bridge** — TD3, SAC (Advanced RL 레포로), 각각의 DDPG 한계 해결 방식

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **32개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 trick이 실전 Deep RL에 필수인가
## 📐 수학적 선행 조건 (Model-Free RL, NN Theory, Opt, CNN 참조)
## 📖 직관적 이해
   — Atari 게임 frame 예시, Q-value plot
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Max bias, Experience Replay justification, PER unbiasedness
## 💻 PyTorch 구현 검증
   — 간단한 환경 (CartPole, LunarLander)에서 구현
   — 각 trick 유무로 학습 곡선 비교
## 🔗 후속 레포와의 연결
   — Policy Gradient, Advanced RL
## ⚖️ 가정과 한계
   — Discrete action, sample inefficient
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Atari 프레임 시각화** — Breakout, Pong에서 Q-value의 action별 변화
2. **Learning curve** — DQN vs Double DQN vs Rainbow를 같은 환경에서 비교
3. **Q-value 과대추정 측정** — van Hasselt 2016 Fig 2 재현 (게임별)
4. **PER priority 분포** — Training 진행 중 priority 히스토그램 변화
5. **Distributional Z(s,a) 시각화** — Atari에서 각 action의 return 분포
6. **Rainbow ablation** — Hessel 2018 Fig 4 스타일 ablation

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
gymnasium[atari]==0.29.0
ale-py==0.8.1
matplotlib==3.8.0
tensorboard==2.15.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (간단한 DQN on CartPole + Double DQN 비교)
import torch
import torch.nn as nn
import numpy as np
import gymnasium as gym
from collections import deque
import random

class QNet(nn.Module):
    def __init__(self, s_dim, a_dim, h=128):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(s_dim, h), nn.ReLU(),
            nn.Linear(h, h), nn.ReLU(),
            nn.Linear(h, a_dim)
        )
    def forward(self, x): return self.net(x)

class ReplayBuffer:
    def __init__(self, cap): self.buf = deque(maxlen=cap)
    def push(self, *args): self.buf.append(args)
    def sample(self, n):
        batch = random.sample(self.buf, n)
        return map(torch.tensor, zip(*batch))
    def __len__(self): return len(self.buf)

def train_dqn(env_name='CartPole-v1', double=False, n_ep=300):
    env = gym.make(env_name)
    s_dim = env.observation_space.shape[0]
    a_dim = env.action_space.n
    
    online = QNet(s_dim, a_dim)
    target = QNet(s_dim, a_dim)
    target.load_state_dict(online.state_dict())
    opt = torch.optim.Adam(online.parameters(), lr=1e-3)
    buffer = ReplayBuffer(10000)
    gamma = 0.99
    epsilon = 1.0
    
    rewards = []
    for ep in range(n_ep):
        s, _ = env.reset()
        total = 0; done = False
        while not done:
            # epsilon-greedy
            if random.random() < epsilon:
                a = env.action_space.sample()
            else:
                with torch.no_grad():
                    a = online(torch.tensor(s, dtype=torch.float32)).argmax().item()
            
            s2, r, done, trunc, _ = env.step(a)
            buffer.push(s.astype(np.float32), a, r, s2.astype(np.float32),
                       float(done or trunc))
            s = s2; total += r
            if trunc: done = True
            
            if len(buffer) >= 1000:
                ss, aa, rr, ss2, dd = buffer.sample(64)
                ss = ss.float(); aa = aa.long(); rr = rr.float()
                ss2 = ss2.float(); dd = dd.float()
                
                Q_cur = online(ss).gather(1, aa.unsqueeze(1)).squeeze(1)
                
                with torch.no_grad():
                    if double:
                        # Double DQN: online selects, target evaluates
                        a_max = online(ss2).argmax(1)
                        Q_next = target(ss2).gather(1, a_max.unsqueeze(1)).squeeze(1)
                    else:
                        # Vanilla DQN
                        Q_next = target(ss2).max(1)[0]
                    target_Q = rr + gamma * (1 - dd) * Q_next
                
                loss = ((Q_cur - target_Q)**2).mean()
                opt.zero_grad(); loss.backward()
                torch.nn.utils.clip_grad_norm_(online.parameters(), 10)
                opt.step()
        
        epsilon = max(0.05, epsilon * 0.995)
        if ep % 20 == 0:
            target.load_state_dict(online.state_dict())
        rewards.append(total)
    return rewards

# Compare
import matplotlib.pyplot as plt
np.random.seed(42); torch.manual_seed(42); random.seed(42)
r_dqn = train_dqn(double=False)
np.random.seed(42); torch.manual_seed(42); random.seed(42)
r_ddqn = train_dqn(double=True)

def smooth(x, w=10): return np.convolve(x, np.ones(w)/w, mode='valid')
plt.plot(smooth(r_dqn), label='DQN')
plt.plot(smooth(r_ddqn), label='Double DQN')
plt.xlabel('Episode'); plt.ylabel('Total Reward'); plt.legend()
plt.title('DQN vs Double DQN on CartPole')
plt.show()

# Q-value 과대추정 측정
# 훈련 중 max Q - actual return의 차이
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Model-Free RL, NN Theory, Opt, CNN 선행 필수"
   - "tabular → FA → deep"의 연결 강조
   - Rainbow component별 ablation을 프론트페이지에
3. **챕터별 문서 작성**: Deadly Triad → DQN 3 tricks → Double DQN → Dueling/PER/Multi-step → Distributional → Rainbow → Continuous

---

## 📚 참고 자료

- **Human-level control through deep reinforcement learning** (Mnih et al. 2015) — DQN Nature
- **Playing Atari with Deep Reinforcement Learning** (Mnih et al. 2013) — 초기 DQN
- **Deep Reinforcement Learning with Double Q-learning** (van Hasselt et al. 2016) — Double DQN
- **Dueling Network Architectures** (Wang et al. 2016)
- **Prioritized Experience Replay** (Schaul et al. 2016)
- **A Distributional Perspective on RL** (Bellemare et al. 2017) — C51
- **Distributional RL with Quantile Regression** (Dabney et al. 2018) — QR-DQN
- **Noisy Networks for Exploration** (Fortunato et al. 2018)
- **Rainbow: Combining Improvements** (Hessel et al. 2018)
- **Continuous control with deep reinforcement learning** (Lillicrap et al. 2016) — DDPG
- **Mastering Atari, Go, Chess and Shogi by Planning with a Learned Model** (Schrittwieser et al. 2020) — MuZero
- **Distributed Prioritized Experience Replay** (Horgan et al. 2018) — Ape-X

---

## 💡 핵심 분석 대상

```
Deep RL의 Tricks and Methods

───── Deadly Triad 재현 ─────

Baird's 7-state star MDP:
  off-policy + bootstrap + linear FA
  → 발산 guaranteed

DNN은 더 어렵지만 경험적으로 성공:
  3 tricks (Experience Replay, Target, Clip)
  + 추가 개선 (Double, Dueling, PER, ...)

───── DQN (Mnih 2015) ─────

Loss:
  L = 𝔼_D[(r + γ max_{a'} Q(s', a'; θ^-) - Q(s, a; θ))²]

Tricks:
┌──────────────────┬────────────────────────┐
│ Trick            │ 해결하는 문제             │
├──────────────────┼────────────────────────┤
│ Experience Replay│ Correlation, i.i.d. 위배│
│ Target Network   │ Moving target 불안정   │
│ Reward Clipping  │ Gradient scale 통일    │
└──────────────────┴────────────────────────┘

───── Max Bias ─────

E[max_a Q̂(s,a)] ≥ max_a E[Q̂(s,a)]
  Jensen's inequality

→ Q-Learning 과대추정
  noise + max = 양의 bias

Double DQN:
  a* = argmax_a Q(s', a; θ)      ← online
  target = Q(s', a*; θ^-)         ← target
  → action selection과 evaluation 분리
  → bias 감소

───── Dueling Network ─────

Q(s, a) = V(s) + A(s, a) - mean(A(s, ·))
              └── state value └─ advantage

장점:
  Action 많을 때 V가 지배적
  각 state의 V 공유 학습
  Identifiability: mean subtraction

───── PER ─────

Priority:
  p_i = |δ_i|^α + ε
  
  α=0: uniform (DQN)
  α=1: fully prioritized

IS correction:
  w_i = (1/(N · p_i/Σp))^β
  β: 0 → 1 (anneal)
  
  → biased prioritized sampling을 
    unbiased gradient estimate로

───── Multi-step ─────

n-step target:
  G_t^{(n)} = Σ_{k=0}^{n-1} γ^k r_{t+k+1} 
            + γ^n max Q(s_{t+n}, ·; θ^-)

n ↑:
  Bias ↓ (target에 더 많은 실제 reward)
  Variance ↑
  On-policy assumption 위배 (tolerable)

───── Distributional RL ─────

Z^π(s,a): return 분포
Q^π(s,a) = 𝔼[Z^π]

Distributional Bellman:
  Z(s,a) =D r + γ Z(s', π(s'))

C51 (Bellemare 2017):
  51 atoms on [V_min, V_max]
  Categorical distribution
  KL divergence loss

QR-DQN (Dabney 2018):
  Quantiles τ_i = (2i-1)/(2N)
  Huber quantile loss
  Wasserstein-1 minimization

이점:
  전체 분포 정보 (risk)
  Auxiliary task effect
  Better gradient signal

───── Noisy Networks ─────

W = μ + σ ⊙ ε  (factorized noise)

장점:
  State-dependent exploration
  Learnable noise scale
  ε-greedy 대체

───── Rainbow (Hessel 2018) ─────

6 tricks 통합:
  1. Double DQN
  2. Dueling
  3. PER
  4. Multi-step
  5. Distributional (C51)
  6. Noisy Networks

Ablation 결과:
  가장 중요: Multi-step, PER, Distrib
  영향 작음: Double, Dueling

───── 현대 확장 ─────

Ape-X (Horgan 2018):
  Distributed replay
  Multiple actors

R2D2 (Kapturowski 2019):
  Recurrent (LSTM) + replay

IMPALA (Espeholt 2018):
  V-trace: off-policy correction
  
MuZero (Schrittwieser 2020):
  Learned dynamics model
  AlphaZero without rules
  Atari·Go·Chess SOTA

───── Continuous Control ─────

DQN 직접 적용 X (max over continuous)

DDPG (Lillicrap 2016):
  Actor μ_θ(s) (deterministic)
  Critic Q_φ(s, a)
  
  ∇J ≈ 𝔼[∇_θ μ · ∇_a Q|_{a=μ}]

문제: 불안정, 하이퍼 민감
→ TD3, SAC (Advanced RL)

───── 레포 간 연결 ─────

Model-Free RL (직전):
  Q-Learning, Double Q-Learning
  TD 이론

NN Theory (Layer 2):
  Backprop, architecture

Optimization (Layer 2):
  Adam, RMSProp
  Gradient clipping

CNN (Layer 3):
  Atari frame 처리
  Shared feature

Probability (Layer 0):
  Return distribution (C51)

Policy Gradient (다음):
  DDPG → SAC의 Actor-Critic
  Continuous control 계보

Advanced RL (2개 뒤):
  TD3, SAC, TRPO, PPO
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 trick·증명 (3~4줄)
- 전체 문서 개수 확인 (32개 목표)
- Python + PyTorch + Gymnasium (Atari) 실험 환경
- Model-Free, NN Theory, Opt, CNN 레포의 참조 관계
- Policy Gradient로 이어지는 continuous control

**준비됐으면 1단계 구조 설계부터 시작해줘!**
