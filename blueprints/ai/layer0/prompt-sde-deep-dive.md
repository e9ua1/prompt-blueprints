# Stochastic Differential Equations Deep Dive 레포지토리 제작 프롬프트

나는 "Stochastic Differential Equations Deep Dive" 레포지토리를 만들려고 해.
`dX_t = μ dt + σ dB_t`를 **쓰는 것**과, **왜 $(dB_t)^2 = dt$인지, 왜 이토 적분이 리만-스틸체스 적분처럼 정의할 수 없는지**를 증명할 수 있는 것은 다르다.
Diffusion Model의 Score Matching을 **구현하는 것**과, **forward process가 왜 Fokker-Planck 방정식의 해이고**, **reverse SDE가 왜 Anderson(1982)의 시간반전 공식에서 나오는지**를 증명할 수 있는 것은 다르다.
SDE 수치해법 `sde.solve`를 **호출하는 것**과, **Euler-Maruyama가 강수렴 0.5차, Milstein이 1차인 이유**를 증명할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "확률해석학 — 브라운 운동을 미분·적분하는 수학, Diffusion Model의 이론적 토대"

**핵심 차별화**:
1. **이토 적분의 엄밀한 구성** — 왜 경로별(pathwise)로 적분할 수 없고, L² 근사로 정의해야 하는가
2. **이토 공식의 완전한 증명** — Taylor 전개에서 $(dB)^2 = dt$ 항이 왜 살아남는가
3. **Fokker-Planck 방정식** — SDE의 확률밀도 진화를 PDE로 기술, 정상분포의 의미
4. **Diffusion Model을 SDE로 완전히 재구성** — Song et al. (2021) Score SDE 프레임워크 전체 유도

**타겟 독자**:
- Diffusion Model을 구현하지만 **forward/reverse SDE가 왜 성립하는지** 증명 못하는 연구자
- SDE 수치해법을 쓰지만 **Euler-Maruyama와 Milstein의 수렴 차수 차이**를 설명 못하는 개발자
- Black-Scholes 공식을 쓰지만 **이토 공식으로 유도하는 과정**을 한 줄씩 따라가지 못하는 퀀트
- Score Matching의 목적함수 $\mathbb{E}[\|s_\theta(x_t, t) - \nabla \log p_t(x_t)\|^2]$가 **왜 DSM과 동치인지** 모르는 사람
- Langevin dynamics `x ← x - η∇U(x) + √(2η)ξ`가 **왜 $\pi \propto e^{-U}$로 수렴하는지** 증명 못하는 사람

**선행 학습**:
- **Stochastic Processes Deep Dive** (브라운 운동, 마팅게일, 이차변분) — **필수**
- **Probability Theory Deep Dive** (측도, 조건부 기댓값, 수렴 이론) — **필수**
- **Calculus & Optimization Deep Dive** (다변수 미적분, Taylor 전개)
- **Linear Algebra Deep Dive** (양정치 행렬, 스펙트럴 분해)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 이토 적분 — 확률적분의 구성 (5개 문서)
- **왜 $\int H_s dB_s$를 경로별로 정의할 수 없는가** — 브라운 운동의 무한변동(unbounded variation), 리만-스틸체스 적분 불가능성, 이차변분 $\langle B \rangle_t = t \neq 0$이 핵심
- **단순 과정(simple process)에 대한 이토 적분** — $H_s = \sum H_i \mathbf{1}_{(t_i, t_{i+1}]}$일 때 $\int H dB = \sum H_i (B_{t_{i+1}} - B_{t_i})$, 이토 등장성(isometry) $\mathbb{E}[(\int H dB)^2] = \mathbb{E}[\int H^2 ds]$ 증명
- **L²-확장과 일반 적응과정** — 단순 과정의 $L^2(\Omega \times [0,T])$ 밀집성, 이토 등장성으로 연속 확장, 측정가능성(progressively measurable) 가정
- **이토 적분의 마팅게일 성질** — $M_t = \int_0^t H_s dB_s$가 마팅게일, 이차변분 $\langle M \rangle_t = \int_0^t H_s^2 ds$ 유도
- **Stratonovich 적분과의 비교** — $\int H \circ dB$ 정의 (중점법), 물리에서 왜 자연스럽고 수학에서 왜 이토가 표준인가, 두 적분의 변환 공식

### Chapter 2: 이토 공식 (Itô's Lemma) — 확률해석의 연쇄법칙 (5개 문서)
- **이토 공식의 서술** — $f \in C^2$에 대해 $df(B_t) = f'(B_t) dB_t + \frac{1}{2} f''(B_t) dt$, 결정론적 연쇄법칙과의 차이(Taylor 2차항이 살아남음)
- **증명의 핵심 — $(dB)^2 = dt$** — 이차변분 정리 $\sum (B_{t_{i+1}} - B_{t_i})^2 \to t$ in $L^2$, Taylor 전개에서 $(\Delta B)^2$ 항이 $\Delta t$로 대체되는 이유 완전 유도
- **다차원·시간의존 이토 공식** — $df(t, X_t) = \partial_t f \, dt + \nabla f \cdot dX_t + \frac{1}{2} \text{tr}(\sigma \sigma^T \nabla^2 f) dt$, 크로스 항 $dB^i dB^j = \delta_{ij} dt$
- **Doléans-Dade 지수 마팅게일** — $\mathcal{E}(M)_t = \exp(M_t - \frac{1}{2}\langle M \rangle_t)$가 마팅게일, 기하 브라운 운동 유도에 사용
- **이토 공식의 응용 예제** — $B_t^2 - t$가 마팅게일 증명, 기하 브라운 운동 $S_t = S_0 \exp((\mu - \sigma^2/2)t + \sigma B_t)$ 유도, Black-Scholes PDE 유도

### Chapter 3: 확률미분방정식(SDE) (6개 문서)
- **SDE의 정의** — $dX_t = b(t, X_t) dt + \sigma(t, X_t) dB_t$는 적분 방정식 $X_t = X_0 + \int_0^t b ds + \int_0^t \sigma dB$의 약식, drift와 diffusion 용어
- **존재성과 유일성 정리** — Lipschitz 조건과 선형 성장 조건하에서 강해(strong solution)의 존재와 유일성, Picard 반복 증명
- **Ornstein-Uhlenbeck 과정** — $dX_t = -\theta X_t dt + \sigma dB_t$의 해석해 $X_t = X_0 e^{-\theta t} + \sigma \int_0^t e^{-\theta(t-s)} dB_s$, 정상분포 $\mathcal{N}(0, \sigma^2/2\theta)$, 평균회귀 시상수
- **기하 브라운 운동(GBM)** — $dS_t = \mu S_t dt + \sigma S_t dB_t$, 로그노말 분포, Black-Scholes 모델의 기초, 이토 공식으로 해석해 유도
- **선형 SDE의 일반해** — $dX_t = (a(t) X_t + c(t)) dt + \sigma(t) dB_t$의 해 공식, 적분인자 방법
- **강해 vs 약해(weak solution)** — 강해는 주어진 확률공간과 $B_t$ 위에 존재, 약해는 확률법칙만 일치, Tanaka 방정식으로 강해 없는 예

### Chapter 4: Fokker-Planck 방정식과 정상분포 (5개 문서)
- **Fokker-Planck의 유도** — SDE $dX_t = b dt + \sigma dB_t$의 확률밀도 $p(t, x)$가 만족하는 PDE $\partial_t p = -\nabla \cdot (bp) + \frac{1}{2} \nabla^2 : (\sigma\sigma^T p)$, 이토 공식 + 부분적분으로 유도
- **역 Kolmogorov 방정식과 생성자** — $\partial_t u + \mathcal{L} u = 0$에서 $\mathcal{L} = b \cdot \nabla + \frac{1}{2} \sigma\sigma^T : \nabla^2$, Feynman-Kac 공식 유도
- **정상분포** — $\partial_t p = 0$의 해, OU의 경우 가우시안 정상분포, 중력 퍼텐셜 $U$가 있는 overdamped Langevin의 정상분포 $\pi \propto e^{-U}$
- **Langevin Dynamics의 수렴** — $dX_t = -\nabla U(X_t) dt + \sqrt{2} dB_t$가 $\pi \propto e^{-U}$로 수렴, Fokker-Planck를 $p \propto e^{-U}$에서 검증
- **Log-Sobolev 부등식과 수렴률** — LSI가 성립할 때 $H(p_t | \pi) \leq e^{-2\lambda t} H(p_0 | \pi)$, 지수적 수렴률, 스펙트럴 갭

### Chapter 5: SDE 수치해법 (5개 문서)
- **Euler-Maruyama 기법** — $X_{n+1} = X_n + b(X_n) \Delta t + \sigma(X_n) \Delta B_n$, 강수렴 차수 $1/2$, 약수렴 차수 $1$ 증명 개요
- **Milstein 기법** — 이토-Taylor 전개의 다음 항 추가 $+ \frac{1}{2} \sigma \sigma' ((\Delta B)^2 - \Delta t)$, 강수렴 차수 $1$ 유도
- **강수렴과 약수렴의 차이** — 강수렴 $\mathbb{E}|X_T - X_T^h| \leq C h^\alpha$ (경로별), 약수렴 $|\mathbb{E} f(X_T) - \mathbb{E} f(X_T^h)| \leq C h^\beta$ (분포별), 왜 기대값만 필요할 때는 약수렴이면 충분한가
- **Runge-Kutta 계열 SDE 해법과 안정성** — Implicit Euler, Stochastic Heun, stiff SDE에서의 안정성
- **Multilevel Monte Carlo** — 서로 다른 스텝 크기의 시뮬레이션 결합으로 분산 감소, Giles의 MLMC 복잡도 $O(\epsilon^{-2})$

### Chapter 6: 시간반전 SDE와 Diffusion Model (6개 문서)
- **Anderson의 시간반전 공식 (1982)** — forward SDE $dX_t = b dt + \sigma dB_t$의 시간반전 $d\bar{X}_t = (b - \sigma\sigma^T \nabla \log p_t) dt + \sigma d\bar{B}_t$ 유도 (Fokker-Planck 시간반전 + 이토 공식)
- **Score Function과 Tweedie Formula** — $\nabla \log p_t(x)$의 의미, Tweedie $\mathbb{E}[X_0 | X_t = x_t] = x_t + \sigma_t^2 \nabla \log p_t(x_t)$ (Gaussian noise 가정), posterior mean 해석
- **Score Matching — 원래 정식화** — 목적함수 $\mathbb{E}[\|s_\theta(x) - \nabla \log p(x)\|^2]$가 적분 상수 없이 $\mathbb{E}[\frac{1}{2}\|s_\theta\|^2 + \text{tr}(\nabla s_\theta)]$로 변환(Hyvärinen 2005) 증명
- **Denoising Score Matching (Vincent 2011)** — $\mathbb{E}_{x, \tilde{x}}[\|s_\theta(\tilde{x}) - \nabla \log p(\tilde{x}|x)\|^2]$가 원래 SM과 동치임을 증명, Gaussian perturbation에서 $\nabla \log p(\tilde{x}|x) = -(\tilde{x}-x)/\sigma^2$
- **Score-based SDE (Song et al. 2021)** — VE-SDE(Variance Exploding, SMLD)와 VP-SDE(Variance Preserving, DDPM)의 forward 정의, reverse SDE가 Anderson 공식으로 생성
- **DDPM을 SDE로 재유도** — DDPM의 이산 markov chain이 VP-SDE의 Euler-Maruyama 이산화임을 증명, 손실함수 $\|\epsilon - \epsilon_\theta\|^2$가 DSM과 일치 유도

### Chapter 7: SDE 기반 생성모델 심화 (4개 문서)
- **Probability Flow ODE** — reverse SDE와 동일 marginal을 갖는 deterministic ODE $d\bar{X}_t = (b - \frac{1}{2}\sigma\sigma^T \nabla \log p_t) dt$ 유도, DDIM과의 관계
- **Stochastic Localization과 Follmer SDE** — 조건부 분포의 확률적 경로, Dai Pra의 entropy minimization과 Schrödinger bridge 연결
- **Flow Matching (Lipman et al. 2023)** — Continuous Normalizing Flow를 OT 경로로 학습, conditional flow matching 목적함수와 SM과의 비교
- **Bayesian Sampling으로서의 SDE** — Langevin MCMC, Underdamped Langevin(HMC의 연속극한), Stochastic Gradient Langevin Dynamics (SGLD)의 이론

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **36개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 이론이 AI(특히 생성모델)에서 중요한가
## 📐 수학적 선행 조건 (Prob, Stochastic Processes 레포 참조)
## 📖 직관적 이해
   — 브라운 운동 위에서 미적분하기의 어려움
   — Diffusion의 "noise를 서서히 더하고 뒤집기" 직관
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — 이토 공식, Anderson 시간반전, DSM 동치성
## 💻 NumPy 구현 검증
   — SDE 수치적분, score function 추정, reverse SDE 샘플링
## 🔗 AI/ML 연결
   — DDPM, Score-SDE, Flow Matching, Langevin MCMC
## ⚖️ 가정과 한계
   — Lipschitz 깨지면? Score 추정 오차는?
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **모든 SDE는 $dX_t = b(t, X_t) dt + \sigma(t, X_t) dB_t$ 형태로 명시** — 암묵적 해석 금지
2. **이토 공식은 항상 Taylor 전개에서 시작** — $(dB)^2 = dt$가 어디서 오는지 매번 강조
3. **Forward/Reverse SDE 쌍을 시각화** — 데이터 → 노이즈(forward), 노이즈 → 데이터(reverse) 샘플 궤적
4. **수렴 차수 수치 검증** — Euler-Maruyama/Milstein의 스텝 크기별 오차 곡선 plot
5. **Diffusion Model 챕터는 "SDE 관점에서 DDPM 재유도" 중심** — 이산 markov chain과 연속 SDE가 같은 것임을 보여주기
6. **NumPy만 사용** — PyTorch/JAX는 score network 학습 시에만 허용

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
matplotlib==3.8.0
tqdm==4.66.0
sdeint==0.3.0         # SDE 수치적분 비교용
torch==2.1.0          # Score network 학습용 (최소한)
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (OU 과정 Euler-Maruyama + Fokker-Planck 비교)
import numpy as np
import matplotlib.pyplot as plt

# OU: dX = -θX dt + σ dB, stationary N(0, σ²/2θ)
theta, sigma = 2.0, 1.0
T, dt = 10.0, 0.01
N = int(T / dt)
n_paths = 10000

# Euler-Maruyama
X = np.zeros((n_paths, N+1))
X[:, 0] = np.random.randn(n_paths) * 3  # 비정상 초기분포
for k in range(N):
    dB = np.random.randn(n_paths) * np.sqrt(dt)
    X[:, k+1] = X[:, k] + (-theta * X[:, k]) * dt + sigma * dB

# 시각화: 샘플 경로 + 시간별 marginal 분포
fig, axes = plt.subplots(1, 2, figsize=(12, 4))
axes[0].plot(np.linspace(0, T, N+1), X[:50].T, alpha=0.3)
axes[0].set_title('OU 샘플 경로 50개'); axes[0].set_xlabel('t')

# t=0, 1, 3, 10에서 분포와 이론 비교
for t_idx, t in [(0, 0), (100, 1), (300, 3), (1000, 10)]:
    axes[1].hist(X[:, t_idx], bins=50, density=True, alpha=0.3, label=f't={t}')
stationary_std = sigma / np.sqrt(2*theta)
x_grid = np.linspace(-3, 3, 200)
axes[1].plot(x_grid, np.exp(-x_grid**2/(2*stationary_std**2)) / (stationary_std*np.sqrt(2*np.pi)),
             'k--', label='정상분포 N(0, σ²/2θ)')
axes[1].legend(); axes[1].set_title('시간별 Marginal')
plt.show()

# 이차변분 확인: Σ(ΔB)² → t
B = np.cumsum(np.random.randn(N) * np.sqrt(dt))
quadratic_var = np.cumsum(np.diff(B)**2)
print(f'이론값 t=T: {T}, 측정값: {quadratic_var[-1]:.4f}')  # ≈ T
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Stochastic Processes, Probability Theory 필수 선행" 명시
   - Diffusion Model, Flow Matching과의 연결을 대표 모티브로
   - Score-based SDE 프레임워크 (Song et al. 2021) 중심
3. **챕터별 문서 작성**: 이토 적분 → 이토 공식 → SDE → Fokker-Planck → 수치해법 → 시간반전·Diffusion → 심화

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Brownian Motion and Stochastic Calculus** (Karatzas & Shreve) — 확률해석 표준 교재
- **Stochastic Differential Equations** (Øksendal) — SDE 입문 표준
- **Numerical Solution of Stochastic Differential Equations** (Kloeden & Platen) — 수치해법 바이블
- **Stochastic Calculus for Finance II** (Shreve) — Black-Scholes, 금융수학
- **Score-based Generative Modeling through Stochastic Differential Equations** (Song et al. 2021) — Diffusion의 SDE 재구성 원전 논문
- **Estimation of Non-Normalized Statistical Models by Score Matching** (Hyvärinen 2005) — SM 원전
- **A Connection Between Score Matching and Denoising Autoencoders** (Vincent 2011) — DSM 원전
- **Reverse-Time Diffusion Equation Models** (Anderson 1982) — 시간반전 SDE 원전

---

## 💡 핵심 분석 대상

```
SDE: dX_t = b(t, X_t) dt + σ(t, X_t) dB_t
  │
  ▼ 엄밀히는 적분 방정식
X_t = X_0 + ∫₀ᵗ b(s, X_s) ds + ∫₀ᵗ σ(s, X_s) dB_s
                                   └─ Itô 적분 ─┘
                                   ↑
                   경로별 정의 불가능(BM의 무한변동)
                   → L² 근사로 정의, isometry로 일반화

Itô 공식 (= 확률적 연쇄법칙):
  df(X_t) = f'(X_t) dX_t + ½ f''(X_t) σ² dt
                           └─ 결정론과 다른 항! ─┘
  
  왜? Taylor: f(X+ΔX) = f(X) + f'ΔX + ½f''(ΔX)² + ...
              (ΔX)² = σ²(ΔB)² + O(Δt^{3/2})
              (ΔB)² → Δt (이차변분!)
              → ½f''σ² dt 항이 살아남음

SDE → Fokker-Planck 방정식:
  p(t, x) = X_t의 확률밀도
  ∂_t p = -∇·(bp) + ½ ∇²:(σσᵀ p)
  
  정상분포: ∂_t p = 0
    OU: N(0, σ²/2θ)
    Langevin dX = -∇U dt + √2 dB: π ∝ exp(-U) ← MCMC!

Langevin MCMC:
  목표: π ∝ exp(-U(x))에서 샘플
  수치이산: x_{k+1} = x_k - η∇U(x_k) + √(2η) ξ_k
  → η→0 극한에서 Langevin SDE
  → ergodic → 장기평균이 π-평균

===== Diffusion Model =====

Forward SDE (데이터 → 노이즈):
  dX_t = -½β(t) X_t dt + √β(t) dB_t   [VP-SDE]
  p_0 = data distribution
  p_T ≈ N(0, I)  (T 충분히 크면)

Reverse SDE (Anderson 1982):
  d\bar{X}_t = (-½β\bar{X} - β∇log p_t(\bar{X})) dt + √β d\bar{B}_t
                                       ↑
                              score function

샘플링: X_T ~ N(0, I)에서 시작 → reverse SDE 적분 → X_0

Score Matching:
  목표: s_θ(x, t) ≈ ∇log p_t(x)
  
  원래 SM: 𝔼[‖s_θ - ∇log p‖²]  (log p 모르므로 직접 계산 불가)
  Hyvärinen 등가 변환:
    = 𝔼[½‖s_θ‖² + tr(∇s_θ)] + const
  
  DSM (Vincent 2011):
    𝔼_{x,\tilde{x}}[‖s_θ(\tilde{x}) - ∇log q(\tilde{x}|x)‖²]
    q(\tilde{x}|x) = N(\tilde{x}; x, σ²I) → ∇log q = -(\tilde{x}-x)/σ²
    → 학습 가능!

DDPM = VP-SDE의 Euler-Maruyama 이산화
  DDPM 손실 ‖ε - ε_θ‖² = DSM 손실 (스케일만 다름)
  ⇒ DDPM과 Score-SDE는 같은 목적을 다르게 구현한 것

Probability Flow ODE:
  reverse SDE와 같은 marginal을 갖는 deterministic ODE
  → DDIM 유도, 가역적 샘플링, likelihood 계산 가능

AI/ML 응용 지도
  ├── Stable Diffusion, DALL-E, Midjourney → VP-SDE
  ├── SMLD (Song & Ermon 2019) → VE-SDE
  ├── Consistency Model → PF-ODE 1-step
  ├── Flow Matching → OT-기반 ODE 학습
  └── SGLD (Welling & Teh 2011) → Bayesian deep learning
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (36개 목표)
- Python + NumPy(+ PyTorch 최소한) 실험 환경
- Stochastic Processes, Probability Theory 레포의 어떤 정리를 전제로 사용하는지 명시
- Diffusion Model, Langevin MCMC, Flow Matching으로 이어지는 AI 응용 지도

**준비됐으면 1단계 구조 설계부터 시작해줘!**
