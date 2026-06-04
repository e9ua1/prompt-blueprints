# Optimization Theory Deep Dive 레포지토리 제작 프롬프트

나는 "Optimization Theory Deep Dive" 레포지토리를 만들려고 해.
`torch.optim.SGD(lr=0.01)`을 **쓰는 것**과, **$L$이 $\mu$-강볼록·$L$-smooth일 때 SGD의 수렴률 $\mathbb{E}[f(x_T) - f^*] \leq O(1/T)$** 를 Robbins-Monro 조건 $\sum \eta_t = \infty, \sum \eta_t^2 < \infty$에서 유도할 수 있는 것은 다르다.
`torch.optim.Adam`을 **호출하는 것**과, **Reddi et al. (2018) "On the Convergence of Adam and Beyond"의 반례 — 특정 online convex 문제에서 Adam이 발산하고 AMSGrad가 필요** 한 이유를 증명할 수 있는 것은 다르다.
LR Warmup을 **쓰는 것**과, **"왜 초기 큰 LR이 BatchNorm 없는 ResNet에서 발산을 일으키는지"의 edge-of-stability 이론**(Cohen et al. 2021)을 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "딥러닝 최적화의 수학 — 수렴률·발산 반례·Loss Landscape·실전 튜닝"

**핵심 차별화**:
1. **SGD 수렴 증명의 4가지 세팅** — Convex+smooth, Strongly convex, Nonconvex smooth, Nonsmooth(subgradient), 각각 $O(1/\sqrt{T}), O(1/T), O(1/\sqrt{T}), O(\log T/\sqrt{T})$
2. **Adam의 수렴 반례와 AMSGrad** — Reddi 반례를 NumPy로 재현, Adam의 구조적 문제와 수정
3. **Loss Landscape의 실증·이론** — saddle point 많음, flat minima가 generalize, mode connectivity, Li et al. 2018의 시각화
4. **LR Scheduler의 이론적 정당성** — Cosine decay, linear warmup, cyclical LR의 수렴 관점 분석

**타겟 독자**:
- Adam 기본값 `lr=1e-3, β₁=0.9, β₂=0.999`를 쓰지만 **각 파라미터가 무엇을 제어**하는지 정확히 모르는 사람
- SGD with momentum이 Polyak's heavy ball과 **어떻게 다른지**, Nesterov's accelerated gradient가 **왜 이론상 최적 $O(1/T^2)$**인지 증명 못하는 사람
- Loss landscape에서 "saddle point에 갇혀"를 말하지만 **고차원에서 local min vs saddle 비율**의 random matrix 이론(Dauphin et al. 2014)을 모르는 사람
- Learning Rate Warmup을 쓰지만 **왜 Transformer 훈련에서 warmup이 필수이고 ResNet에서는 덜 중요**한지 모르는 사람
- Grokking 같은 현상에서 **optimizer의 역할**을 궁금해하는 연구자

**선행 학습**:
- **Convex Optimization Deep Dive** (볼록·강볼록, Lipschitz smooth, 쌍대) — **필수**
- **Calculus & Optimization Deep Dive** (Gradient descent, Taylor 정리) — **필수**
- **Probability Theory Deep Dive** (Martingale, 집중부등식) — SGD 분석에 필요
- **Statistical Learning Theory Deep Dive** (Rademacher, 일반화 경계) — 권장
- **Neural Network Theory Deep Dive** (Backprop, Initialization) — 선행

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Gradient Descent의 기초와 수렴 분석 (5개 문서)
- **GD의 정의와 기하학적 직관** — $x_{t+1} = x_t - \eta \nabla f(x_t)$, steepest descent direction, 연속 시간 gradient flow $\dot{x} = -\nabla f(x)$와의 관계
- **Convex Smooth Case 수렴 증명** — $L$-smooth convex $f$에서 $\eta \leq 1/L$이면 $f(x_T) - f^* \leq \frac{\|x_0 - x^*\|^2}{2\eta T}$, $O(1/T)$ 수렴, descent lemma $f(y) \leq f(x) + \nabla f(x)^T(y-x) + \frac{L}{2}\|y-x\|^2$ 핵심
- **Strongly Convex Case 수렴** — $\mu$-strongly convex $f$에서 linear rate $\|x_T - x^*\| \leq (1 - \mu/L)^T \|x_0 - x^*\|$, condition number $\kappa = L/\mu$의 역할, Polyak-Lojasiewicz 조건으로 확장
- **Non-convex Smooth Case** — Saddle point 회피 어려움, 수렴은 $\|\nabla f(x_t)\| \to 0$만 (stationary point), GD는 almost surely saddle 회피 (Ge et al. 2015) 증명 개요
- **Projected Gradient와 Proximal Gradient** — 제약 $x \in C$에서 $x_{t+1} = \Pi_C(x_t - \eta \nabla f)$, Lasso의 ISTA, FISTA의 가속 $O(1/T^2)$, Convex Opt 레포 참조

### Chapter 2: SGD의 수학적 분석 (6개 문서)
- **SGD의 정의와 Robbins-Monro 조건** — $x_{t+1} = x_t - \eta_t g_t$, $\mathbb{E}[g_t | x_t] = \nabla f(x_t)$, 수렴을 위한 LR 조건 $\sum \eta_t = \infty, \sum \eta_t^2 < \infty$, 고전적 확률근사 이론
- **Convex Case에서 SGD $O(1/\sqrt{T})$ 증명** — Bounded stochastic gradient $\mathbb{E}\|g_t\|^2 \leq G^2$, $\eta_t = 1/\sqrt{t}$로 $\mathbb{E}[f(\bar{x}_T) - f^*] \leq O(G \cdot D / \sqrt{T})$, averaging의 역할
- **Strongly Convex SGD** — $\eta_t = 1/(\mu t)$로 $\mathbb{E}\|x_T - x^*\|^2 \leq O(1/T)$, Polyak-Ruppert averaging으로 asymptotic optimality
- **Non-convex SGD와 Saddle Points** — $\min_t \mathbb{E}\|\nabla f(x_t)\|^2 \leq O(1/\sqrt{T})$ 수렴, perturbed SGD가 saddle point 탈출 ($\text{poly}(d)/\epsilon^2$ 복잡도, Jin et al. 2017)
- **Mini-batch SGD와 Variance Reduction** — Batch size $B$와 LR $\eta$의 관계 ("linear scaling rule"), SVRG (Johnson & Zhang 2013)의 $O(1/T)$ rate in non-smooth case
- **SGD의 Implicit Regularization** — SGD가 flat minimum으로 편향된다는 실증, SDE 극한 $dx = -\nabla f dt + \sqrt{\eta/B} dW$, noise의 regularization 효과 (Stochastic Processes 레포 연결)

### Chapter 3: Momentum과 가속화 기법 (5개 문서)
- **Polyak's Heavy Ball Method** — $x_{t+1} = x_t - \eta \nabla f(x_t) + \beta (x_t - x_{t-1})$, 물리적 momentum 유추, 이차 수렴 증명, ill-conditioned 문제에서 유리
- **Nesterov's Accelerated Gradient (NAG)** — "lookahead" gradient 계산 $y_t = x_t + \beta(x_t - x_{t-1})$, $x_{t+1} = y_t - \eta \nabla f(y_t)$, convex smooth에서 $O(1/T^2)$ 최적 rate 증명
- **NAG의 ODE 해석** — Continuous limit $\ddot{x} + (3/t) \dot{x} + \nabla f(x) = 0$ (Su, Boyd, Candes 2014), friction 항이 감쇠 제공, 적절한 $\beta$ scheduling
- **Momentum과 Optimum Oscillation** — 큰 momentum이 minimum 근처에서 진동 유발, damping과의 균형, condition number에 따른 optimal momentum
- **SGD with Momentum의 수렴** — Stochastic setting에서 momentum, noise가 heavy ball을 어떻게 바꾸는가, bias-variance trade-off

### Chapter 4: 적응적 학습률 (Adaptive Methods) (6개 문서)
- **AdaGrad (Duchi et al. 2011)** — 좌표별 누적 gradient $G_t = \sum_s g_s \odot g_s$, $x_{t+1} = x_t - \eta g_t / \sqrt{G_t + \epsilon}$, sparse data에서 강력, regret $O(\sqrt{T})$ 증명
- **RMSProp와 이동평균** — AdaGrad의 LR 감소 문제 해결, $v_t = \beta v_{t-1} + (1-\beta) g_t^2$, Hinton의 Coursera 강의에서 유래, Adam의 기반
- **Adam (Kingma & Ba 2014)** — $m_t$ (1차 moment), $v_t$ (2차 moment), bias correction $\hat{m}_t = m_t/(1-\beta_1^t)$, 현대 딥러닝의 기본값
- **Adam의 수렴 반례 (Reddi et al. 2018)** — Online convex 문제에서 Adam이 발산하는 구체적 구성, $v_t$의 non-monotonic 업데이트가 핵심, AMSGrad의 수정 ($\hat{v}_t = \max(\hat{v}_{t-1}, v_t)$)
- **AdamW — Weight Decay 분리 (Loshchilov & Hutter 2019)** — L2 regularization과 weight decay의 차이, Adam에서 이들이 왜 다른지 ($v_t$의 정규화가 L2를 왜곡), AdamW의 분리 이유
- **Lion, Sophia, 최신 Optimizer** — Lion (Chen et al. 2023)의 sign-based update, Sophia의 2차 정보 활용, 각각의 수렴 분석과 실전 성능

### Chapter 5: Loss Landscape 이론 (5개 문서)
- **High-dimensional Saddle Points** — Dauphin et al. 2014: 고차원 non-convex에서 saddle이 local min보다 훨씬 많다, random matrix의 Wigner 반원 법칙으로 eigenvalue 분포 분석
- **Flat vs Sharp Minima 논쟁** — Hochreiter & Schmidhuber 1997의 flat minima 가설, Keskar et al. 2017의 large batch → sharp minima, Dinh et al. 2017의 reparameterization 반론
- **Loss Landscape 시각화 (Li et al. 2018)** — 2D projection 기법, filter normalization, ResNet의 smoother landscape 관찰, BatchNorm의 landscape smoothing 효과
- **Mode Connectivity (Garipov et al. 2018)** — 서로 다른 훈련 결과가 low-loss path로 연결됨, SGD로 도달한 두 minimum이 고원을 통해 이어짐, 일반화에 대한 함의
- **Neural Tangent Kernel과 Lazy Regime** — 무한폭 NN에서 훈련이 kernel regression으로 환원, $f(x; \theta_t) - f(x; \theta_0) \approx \nabla_\theta f(x; \theta_0)^T (\theta_t - \theta_0)$, NN이 linearize되는 조건 (Chizat et al. 2019)

### Chapter 6: Learning Rate Scheduling (4개 문서)
- **고정 LR vs 감소 LR** — Convex에서 $\eta_t = 1/\sqrt{t}$ vs $\eta_t = 1/(\mu t)$, non-convex에서 step decay vs continuous, 각각의 수렴률
- **Warmup의 이론과 실전** — Goyal et al. 2017 "Large Minibatch SGD", Transformer에서의 warmup 필수성, 초기 큰 LR의 발산 문제, gradient noise scale 기반 설명
- **Cosine Annealing과 SGDR (Loshchilov & Hutter 2017)** — $\eta_t = \eta_{\min} + \frac{1}{2}(\eta_{\max} - \eta_{\min})(1 + \cos(t\pi/T))$, warm restart로 여러 minimum 탐색, snapshot ensemble
- **One-Cycle Policy (Smith 2018)** — LR을 증가→감소하는 curve, super-convergence 주장, momentum도 반대로 scheduling, 실전 튜닝 recipe

### Chapter 7: Edge-of-Stability와 2차 방법 (4개 문서)
- **Edge-of-Stability (Cohen et al. 2021)** — GD의 LR이 $2/L$ 경계에서 진동하며 훈련, Sharpness가 이 경계에 도달할 때까지 증가, 왜 theoretical bound보다 큰 LR로 훈련 가능한가
- **Gradient Noise Scale (GNS)** — $B^* = \text{tr}(\Sigma) / \|g\|^2$, 최적 batch size 추정, LR과 batch size의 교환법칙 (Smith et al. 2018)
- **2차 방법 — Newton, Gauss-Newton, K-FAC** — Newton: $x_{t+1} = x_t - H^{-1} \nabla f$, 빠른 수렴이지만 $H$ 계산 $O(d^2)$ 메모리, Gauss-Newton의 positive-semidefinite 근사, K-FAC의 Kronecker factored approximation (Martens & Grosse 2015)
- **Natural Gradient Descent (Amari 1998)** — Fisher Information 메트릭 사용, $\theta_{t+1} = \theta_t - \eta F^{-1} \nabla L$, Information Geometry 레포와 연결, KL divergence가 지역적으로 Fisher

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **35개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 최적화 기법이 딥러닝에 중요한가
## 📐 수학적 선행 조건 (Convex Opt, Calc, Prob 레포 참조)
## 📖 직관적 이해
   — 물리학적 비유 (공, 골짜기, momentum, damping)
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — 수렴률 증명, Adam 반례 구성, Edge-of-stability
## 💻 NumPy 구현 검증
   — 최적화 알고리즘 바닥부터, toy problem에서 수렴 속도 비교
   — Adam 반례 재현, Loss landscape 시각화
## 🔗 실전 활용
   — 언제 어느 optimizer를 쓸 것인가, 하이퍼파라미터 튜닝 recipe
## ⚖️ 가정과 한계
   — 가정이 깨지는 경우 (non-smooth, heavy-tailed gradient)
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Optimizer별 수렴 curve plot** — 같은 toy problem에서 SGD, Momentum, Adam, AMSGrad 비교
2. **Adam 반례 재현 필수** — Reddi et al.의 예제를 직접 구현해 발산 확인
3. **Loss landscape 2D 시각화** — Li et al. 방식으로 간단한 NN의 landscape 그리기
4. **Saddle point vs local min 확인 실험** — 고차원 random function에서 critical point 분류
5. **LR sweep 그래프** — LR 변화에 따른 loss curve, edge-of-stability 관찰
6. **Natural gradient vs standard gradient 비교** — 이차분포 문제에서 수렴 비교

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
matplotlib==3.8.0
torch==2.1.0         # 대규모 실험
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (SGD, Momentum, Adam 비교 + Adam 반례 재현)
import numpy as np
import matplotlib.pyplot as plt

# 1. 기본 optimizer들 바닥부터
def sgd(grad_fn, x0, lr=0.01, T=500):
    x = x0.copy(); history = [x.copy()]
    for t in range(T):
        x -= lr * grad_fn(x)
        history.append(x.copy())
    return np.array(history)

def momentum(grad_fn, x0, lr=0.01, beta=0.9, T=500):
    x = x0.copy(); v = np.zeros_like(x); history = [x.copy()]
    for t in range(T):
        v = beta * v + grad_fn(x)
        x -= lr * v
        history.append(x.copy())
    return np.array(history)

def adam(grad_fn, x0, lr=0.001, b1=0.9, b2=0.999, eps=1e-8, T=500):
    x = x0.copy(); m = np.zeros_like(x); v = np.zeros_like(x)
    history = [x.copy()]
    for t in range(1, T+1):
        g = grad_fn(x)
        m = b1*m + (1-b1)*g
        v = b2*v + (1-b2)*g*g
        m_hat = m / (1 - b1**t)
        v_hat = v / (1 - b2**t)
        x -= lr * m_hat / (np.sqrt(v_hat) + eps)
        history.append(x.copy())
    return np.array(history)

# Rosenbrock 함수 (ill-conditioned)
def rosen(x): return (1-x[0])**2 + 100*(x[1]-x[0]**2)**2
def rosen_grad(x):
    return np.array([
        -2*(1-x[0]) - 400*x[0]*(x[1] - x[0]**2),
        200*(x[1] - x[0]**2)
    ])

x0 = np.array([-1.5, 1.5])
h_sgd = sgd(rosen_grad, x0, lr=0.002)
h_mom = momentum(rosen_grad, x0, lr=0.001, beta=0.9)
h_adam = adam(rosen_grad, x0, lr=0.1)

# contour + trajectory
x, y = np.linspace(-2, 2, 100), np.linspace(-1, 3, 100)
X, Y = np.meshgrid(x, y)
Z = (1-X)**2 + 100*(Y-X**2)**2
plt.contour(X, Y, Z, levels=np.logspace(-2, 3.5, 20), cmap='gray', alpha=0.4)
for h, name in [(h_sgd, 'SGD'), (h_mom, 'Momentum'), (h_adam, 'Adam')]:
    plt.plot(h[:, 0], h[:, 1], '-o', markersize=2, label=name)
plt.scatter([1], [1], c='r', s=100, marker='*', label='optimum')
plt.legend(); plt.title('Rosenbrock 위 optimizer 궤적'); plt.show()

# 2. Adam 반례 재현 (Reddi 2018, Theorem 3)
# f_t(x) = C x for t mod 3 == 1 (확률 1)
#       = -1 * x for t mod 3 != 1 (확률 1)
# True optimum: x = -1 (because average gradient is negative)
# But Adam converges to +1!

def adam_reddi(T=10000, lr=0.001, b1=0.9, b2=0.999, eps=1e-8):
    C = 1010
    x, m, v = 0.0, 0.0, 0.0
    history = []
    for t in range(1, T+1):
        # 주기 3마다 한 번 큰 긍정 gradient, 두 번 작은 부정 gradient
        if t % 3 == 1:
            g = C  # positive big
        else:
            g = -1  # negative small
        m = b1*m + (1-b1)*g
        v = b2*v + (1-b2)*g*g
        m_hat = m / (1 - b1**t)
        v_hat = v / (1 - b2**t)
        x -= lr * m_hat / (np.sqrt(v_hat) + eps)
        x = np.clip(x, -1, 1)  # projection
        history.append(x)
    return np.array(history)

x_hist = adam_reddi()
plt.plot(x_hist); plt.axhline(-1, color='r', linestyle='--', label='true optimum')
plt.title('Adam이 gradient 평균 방향과 반대로 수렴 (Reddi 반례)')
plt.xlabel('step'); plt.ylabel('x'); plt.legend(); plt.show()
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Convex Opt, Calc, Prob 선행 필수" 명시
   - NN Theory (Backprop)과 Generalization Theory 사이의 위치
   - Optimizer = "실전 딥러닝의 거의 모든 것"
3. **챕터별 문서 작성**: GD → SGD → Momentum → Adaptive → Landscape → LR → Edge-of-Stability

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Convex Optimization** (Boyd & Vandenberghe) — 표준 교과서
- **Introductory Lectures on Convex Optimization** (Nesterov) — Nesterov 본인
- **First-Order Methods in Optimization** (Beck) — 1차 방법 현대 정리
- **Adam: A Method for Stochastic Optimization** (Kingma & Ba 2014)
- **On the Convergence of Adam and Beyond** (Reddi et al. 2018) — AMSGrad
- **Decoupled Weight Decay Regularization** (Loshchilov & Hutter 2019) — AdamW
- **Visualizing the Loss Landscape of Neural Nets** (Li et al. 2018)
- **Gradient Descent on Neural Networks Typically Occurs at the Edge of Stability** (Cohen et al. 2021)
- **Identifying and Attacking the Saddle Point Problem** (Dauphin et al. 2014)
- **The Marginal Value of Adaptive Gradient Methods** (Wilson et al. 2017)
- **SGDR: Stochastic Gradient Descent with Warm Restarts** (Loshchilov & Hutter 2017)

---

## 💡 핵심 분석 대상

```
딥러닝 최적화의 지도

┌─── Gradient Descent 계보 ───┐
│                               │
│ GD: x_{t+1} = x_t - η∇f(x_t)  │
│  └── Convex smooth: O(1/T)   │
│  └── Strongly convex: linear  │
│  └── Non-convex: O(1/√T)     │
│      (stationary point)       │
│                               │
│ SGD: g_t 대신 E[g_t] = ∇f     │
│  Robbins-Monro: Σ η = ∞,      │
│                 Σ η² < ∞       │
│  └── Convex: O(1/√T)         │
│  └── Strongly convex: O(1/T)  │
│  └── Non-convex: O(1/√T)     │
│                               │
│ Momentum (Polyak heavy ball): │
│   v = β v + ∇f                │
│   x = x - η v                 │
│  └── Accelerated 수렴        │
│                               │
│ Nesterov (NAG):              │
│   Lookahead gradient          │
│   Convex smooth: O(1/T²)      │
│   (optimal for 1차 방법)     │
└───────────────────────────────┘

┌── Adaptive Methods ──┐
│                        │
│ AdaGrad:              │
│   G_t = Σ g_s² (coord-wise)│
│   x -= η g / √G       │
│   Sparse-friendly     │
│   LR decay too fast   │
│                        │
│ RMSProp:              │
│   v = β v + (1-β) g²  │
│   Fixed LR-like       │
│                        │
│ Adam:                 │
│   m = β₁m + (1-β₁)g   │
│   v = β₂v + (1-β₂)g²  │
│   m̂ = m/(1-β₁ᵗ)       │
│   v̂ = v/(1-β₂ᵗ)       │
│   x -= η m̂/√(v̂+ε)     │
│                        │
│ Adam 반례 (Reddi):    │
│   특정 convex 문제에서│
│   x → wrong direction │
│                        │
│ AMSGrad:              │
│   v̂_t = max(v̂_{t-1}, v_t)│
│   Monotone → 수렴 보장 │
│                        │
│ AdamW:                │
│   Weight decay를 분리  │
│   x -= η (m̂/√v̂ + λw)  │
│   L2 ≠ weight decay in Adam│
└────────────────────────┘

┌── Loss Landscape ──┐
│                       │
│ 고차원 critical pts: │
│   local min ≪ saddle │
│   (random matrix)    │
│                       │
│ Flat vs Sharp:       │
│   Flat = better gen?  │
│   (논쟁 중)          │
│                       │
│ Mode Connectivity:   │
│   SGD 해들이          │
│   low-loss path로 연결 │
│                       │
│ NTK / Lazy Regime:   │
│   무한폭 → kernel reg │
│   θ_t ≈ θ_0 + linear  │
│                       │
│ Edge of Stability:   │
│   Sharpness → 2/η     │
│   진동하며 학습       │
│   (Cohen 2021)        │
└───────────────────────┘

┌── LR Scheduling ──┐
│                       │
│ Warmup:              │
│   0 → η over T_w      │
│   Transformer 필수    │
│                       │
│ Cosine Annealing:    │
│   η_min + ½(η_max-η_min)│
│     × (1 + cos(πt/T)) │
│                       │
│ SGDR (Warm Restart):│
│   여러 cosine cycle   │
│   snapshot ensemble   │
│                       │
│ One-Cycle (Smith):   │
│   up then down        │
│   super-convergence   │
└───────────────────────┘

┌── 2차 방법 ──┐
│                │
│ Newton:       │
│   x -= H⁻¹∇f   │
│   O(d³) cost  │
│                │
│ Gauss-Newton: │
│   H ≈ J^TJ (PSD)│
│                │
│ K-FAC (Martens):│
│   H ≈ A ⊗ B   │
│   Kronecker factored│
│                │
│ Natural Grad (Amari):│
│   x -= η F⁻¹∇f │
│   F = Fisher    │
│   → Info Geometry 레포│
└────────────────┘

───── NN Theory (앞)와 분업 ─────

NN Theory 레포:
  Backprop = reverse-mode AD
  Xavier/He 초기화 분산 보존
  UAT, depth/width

Optimization Theory 레포 (이것):
  학습 알고리즘 수렴률
  Optimizer 비교 및 반례
  Landscape 이론
  LR, Warmup, Scheduler

Generalization Theory 레포 (다음):
  왜 over-parameterized도 일반화?
  Double Descent, NTK, Grokking
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (35개 목표)
- Python + NumPy + PyTorch 실험 환경 (toy problem부터 대규모까지)
- Convex Opt, Calc, Prob, SLT, NN Theory 레포와의 참조 관계
- Generalization Theory로 이어지는 자연스러운 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
