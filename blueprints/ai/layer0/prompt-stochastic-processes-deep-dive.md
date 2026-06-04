# Stochastic Processes Deep Dive 레포지토리 제작 프롬프트

나는 "Stochastic Processes Deep Dive" 레포지토리를 만들려고 해.
마르코프 체인 `transition_matrix @ state`를 돌리는 것과, **전이행렬의 스펙트럴 분해로 정상분포의 수렴률을 예측**할 수 있는 것은 다르다.
MCMC를 쓰는 것과, **detailed balance가 정상분포를 보장**하고, **에르고딕성이 평균의 일치를 보장**한다는 것을 증명할 수 있는 것은 다르다.
Brownian motion의 그림을 보는 것과, **연속이지만 어디에서도 미분불가능**하다는 사실을 측도론적으로 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "시간에 따라 진화하는 확률 구조의 수학 — 마르코프 성질·정상성·에르고딕성이 세 축"

**핵심 차별화**:
1. **확률과정의 엄밀한 정의** — 코모고로프 확장정리, 왜 연속시간 과정이 어려운가
2. **마르코프 체인의 스펙트럴 이론** — 전이행렬의 고유값이 수렴 속도를 결정
3. **브라운 운동을 3가지 방식으로 구성** — Random walk limit, Fourier 전개, Wiener 구성
4. **MCMC의 이론적 근거** — Detailed balance, 에르고딕 정리로 왜 MCMC가 동작하는지 증명

**타겟 독자**:
- 마르코프 체인 샘플링을 하지만 정상분포 수렴까지 몇 스텝이 필요한지 모르는 연구자
- Brownian motion이 "어디에서도 미분불가능"이라는 사실을 쓰지만 왜 그런지 모르는 사람
- Poisson 과정의 지수분포 간격과 메모리리스 성질이 왜 같은 말인지 이해 못하는 사람
- MCMC를 Metropolis-Hastings로 쓰지만 acceptance ratio $\min(1, ...)$의 유도를 설명 못하는 사람
- Diffusion Model의 forward process가 왜 "OU process의 이산화"인지 모르는 사람

**선행 학습**:
- **Probability Theory Deep Dive** (측도, 조건부 기댓값, Tower Property, 수렴 이론) — **필수**
- **Linear Algebra Deep Dive** (Spectral Theorem, Perron-Frobenius)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 확률과정의 기초 (5개 문서)
- **확률과정의 엄밀한 정의** — $\{X_t\}_{t \in T}$는 $(\Omega, \mathcal{F}, \mathbb{P})$ 위의 확률변수 가족, Sample path가 $T \to \mathbb{R}$ 함수
- **유한차원 분포와 Kolmogorov 확장정리** — 유한차원 분포족의 일관성 조건, 존재성 정리 (증명 스케치)
- **정상성(Stationarity) — 강·약** — 엄격 정상과 공분산 정상의 정의, 약 정상이 2차 모멘트 불변을 의미
- **필트레이션(Filtration)과 정보 흐름** — $\mathcal{F}_t$가 시각 $t$까지의 정보, 적응(adapted)·예측가능(predictable)의 정의
- **확률과정의 분류** — 이산/연속 시간, 이산/연속 상태, 마르코프/비마르코프, 정상/비정상의 2×2 분류

### Chapter 2: 이산 마르코프 체인 (6개 문서)
- **마르코프 성질과 전이행렬** — $\mathbb{P}(X_{n+1} | X_n, X_{n-1},...) = \mathbb{P}(X_{n+1} | X_n)$, 전이행렬 $P$의 확률 행렬 성질
- **상태의 분류** — 재귀(recurrent) vs 일시(transient), 주기(periodic) vs 비주기, 상호통신 클래스(communicating class)
- **정상분포(Stationary Distribution)** — $\pi P = \pi$ 방정식, Perron-Frobenius로 유일성 증명 (기약·비주기 조건)
- **극한정리와 수렴률** — $P^n \to \mathbf{1}\pi$ 증명, 수렴 속도는 $|\lambda_2|^n$ (제2고유값)
- **Reversibility와 Detailed Balance** — $\pi_i P_{ij} = \pi_j P_{ji}$가 정상분포의 충분조건, 역과정의 정의
- **에르고딕 정리(Ergodic Theorem)** — 시간평균 = 공간평균, $\frac{1}{n}\sum f(X_k) \to \mathbb{E}_\pi[f]$ a.s. 증명

### Chapter 3: Poisson 과정 (4개 문서)
- **Poisson 과정의 3가지 동치 정의** — 독립증분 + Poisson 분포, 메모리리스 대기 시간, 인피니티시멀 레이트, 세 정의 동치 증명
- **결합·분할과 비균질 과정** — 서로 독립인 Poisson 과정의 합도 Poisson, rate가 시간에 의존하는 비균질 과정
- **복합 Poisson 과정** — $\sum_{k \leq N_t} Y_k$ 형태, 평균·분산·특성함수 유도
- **Queueing 이론 맛보기** — M/M/1 큐의 정상분포, Little의 법칙, Little가 왜 매우 일반적인가

### Chapter 4: 연속시간 마르코프 체인 (4개 문서)
- **생성기(Generator)와 Q-matrix** — 인피니티시멀 생성기 $Q_{ij} = \lim \frac{P_{ij}(t) - \delta_{ij}}{t}$, 행합 0
- **Kolmogorov Forward/Backward 방정식** — $P'(t) = P(t)Q$ (forward), $P'(t) = QP(t)$ (backward), 유도와 해석
- **정상분포와 Detailed Balance (연속)** — $\pi Q = 0$, 상세 균형이 $\pi_i q_{ij} = \pi_j q_{ji}$로
- **Birth-Death 과정** — 상태 $n$에서 $n+1$ 또는 $n-1$로만 이동, 해석적 풀이와 Ising model 등 응용

### Chapter 5: 마팅게일(Martingale) 이론 (5개 문서)
- **마팅게일의 정의** — $\mathbb{E}[X_{n+1} | \mathcal{F}_n] = X_n$, sub/super마팅게일의 정의와 예시(도박 이론 유래)
- **마팅게일 수렴 정리** — Doob의 $L^1$ bounded martingale convergence, 음이 아닌 super마팅게일의 a.s. 수렴
- **Optional Stopping Theorem** — 정지시각 $\tau$에서 $\mathbb{E}[X_\tau] = \mathbb{E}[X_0]$ 성립 조건, 도박장 파산 문제 해결
- **Doob 분해와 이차변분** — 마팅게일의 이차변분 $\langle M \rangle_n$, 확률해석의 기초
- **마팅게일과 ML — Online Learning** — Azuma-Hoeffding 부등식으로 online convex optimization의 regret 경계 유도

### Chapter 6: 브라운 운동(Brownian Motion) (6개 문서)
- **브라운 운동의 공리적 정의** — $B_0 = 0$, 독립증분, 정상증분이 정규분포, 연속 경로, 네 조건의 동치성
- **존재성 — Lévy의 구성** — Haar 기저를 이용한 급수 구성, 균등 수렴으로 연속성 얻기
- **Random Walk Scaling Limit** — $\frac{1}{\sqrt{n}} S_{[nt]} \xrightarrow{d} B_t$ (Donsker 정리 스케치), 이산의 연속 극한
- **경로 성질 — 비미분 가능성** — 거의 확실히 어디에서도 미분 불가능한 증명 (Hausdorff 차원 관점)
- **이차변분 $\langle B \rangle_t = t$** — 브라운 운동의 이차변분이 결정적으로 $t$인 증명, 이토 적분의 근거
- **반사원리(Reflection Principle)와 Maximum** — $M_t = \max_{s \leq t} B_s$의 분포, 반사원리로 유도

### Chapter 7: MCMC — 마르코프 체인 몬테카를로 (5개 문서)
- **MCMC의 아이디어** — 원하는 분포 $\pi$를 정상분포로 갖는 마르코프 체인을 설계 → 샘플링
- **Metropolis-Hastings 알고리즘** — proposal + acceptance $\alpha = \min(1, \frac{\pi(y)q(x|y)}{\pi(x)q(y|x)})$가 detailed balance를 만족함을 증명
- **Gibbs Sampler** — 조건부 분포로 업데이트, Metropolis-Hastings의 특수 경우로 유도
- **Hamiltonian Monte Carlo(HMC)** — 해밀토니안 역학 + leapfrog integrator, 왜 효율적인가 (gradient 활용)
- **MCMC의 수렴 진단과 혼합 시간(Mixing Time)** — $t_{\text{mix}}(\epsilon)$의 정의, 수렴 진단(Gelman-Rubin $\hat{R}$, ESS)

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **35개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 과정이 AI에서 중요한가
## 📐 수학적 선행 조건 (LA, Prob 레포 참조)
## 📖 직관적 이해
   — 랜덤 워크·줄서기 등 물리적 비유
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — 에르고딕 정리, 마팅게일 수렴, 존재성 등
## 💻 시뮬레이션으로 검증
   — 마르코프 체인 수렴, 브라운 운동 궤적
   — MCMC로 실제 샘플링 & 수렴 진단
## 🔗 AI/ML 연결
   — MCMC, HMC, Diffusion Model, RL Q-learning
## ⚖️ 가정과 한계
   — 기약성·비주기·에르고딕 가정이 깨지면?
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **모든 정의는 $(\Omega, \mathcal{F}, \mathbb{P}, \{\mathcal{F}_t\})$로 명시** — 필트레이션까지 항상 함께
2. **샘플 경로 시각화** — 브라운 운동, Poisson 과정, 마르코프 체인의 샘플 궤적 plot 필수
3. **스펙트럴 접근** — 마르코프 체인 수렴을 전이행렬 고유값 분해로 보여줌
4. **이산→연속 연결** — 랜덤 워크의 스케일링 극한이 브라운 운동, 이항분포의 극한이 Poisson 등
5. **MCMC는 실전 예제와 함께** — 복잡한 사후분포에서 MH, Gibbs, HMC 모두 비교 구현

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
matplotlib==3.8.0
networkx==3.2.0    # 마르코프 체인 그래프
arviz==0.17.0      # MCMC 진단
pymc==5.10.0       # MCMC 비교
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (마르코프 체인 수렴률 관찰)
import numpy as np
import matplotlib.pyplot as plt

# 3-state transition matrix (기약·비주기)
P = np.array([
    [0.7, 0.2, 0.1],
    [0.3, 0.4, 0.3],
    [0.2, 0.3, 0.5],
])

# 정상분포: πP = π
eigvals, eigvecs = np.linalg.eig(P.T)
idx = np.argmax(np.abs(eigvals))
pi = np.real(eigvecs[:, idx])
pi = pi / pi.sum()
print(f'정상분포 π = {pi}')
print(f'제2고유값 |λ_2| = {sorted(np.abs(eigvals))[-2]:.4f}')  # 수렴률

# 초기 분포에서 시작해 P^n 적용
mu = np.array([1.0, 0.0, 0.0])
dists = [mu.copy()]
for _ in range(50):
    mu = mu @ P
    dists.append(mu.copy())
dists = np.array(dists)

# TV 거리 수렴 확인 — |λ_2|^n 속도
tv = 0.5 * np.sum(np.abs(dists - pi), axis=1)
plt.semilogy(tv, label='TV distance to π')
plt.semilogy(np.abs(sorted(np.abs(eigvals))[-2])**np.arange(len(tv)), 
             '--', label='|λ_2|^n bound')
plt.legend(); plt.xlabel('n'); plt.ylabel('distance (log)')
plt.show()
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Probability Theory 필수 선행, LA의 Spectral Theorem 활용" 명시
   - "SDE Deep Dive로 이어짐" 언급
   - MCMC·HMC·Diffusion Model과의 연결 강조
3. **챕터별 문서 작성**: 기초 → 이산 MC → Poisson → 연속 MC → 마팅게일 → 브라운 → MCMC

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Probability: Theory and Examples** (Durrett) — 확률과정 챕터
- **Markov Chains and Mixing Times** (Levin & Peres) — MC와 MCMC 수렴
- **Brownian Motion and Stochastic Calculus** (Karatzas & Shreve) — BM 고전
- **Monte Carlo Statistical Methods** (Robert & Casella) — MCMC 표준
- **An Introduction to MCMC for Machine Learning** (Andrieu et al.) — ML 관점
- **Handbook of MCMC** (Brooks et al.)

---

## 💡 핵심 분석 대상

```
확률과정 {X_t}_{t∈T} on (Ω, ℱ, ℙ, {ℱ_t})
  │
  ▼
Markov Property:
  ℙ(X_{t+s} ∈ A | ℱ_t) = ℙ(X_{t+s} ∈ A | X_t)
  "미래는 과거와 독립, 현재가 주어지면"
  │
  ▼
이산 마르코프 체인
  전이행렬 P_{ij} = ℙ(X_{n+1} = j | X_n = i)
  
  정상분포: πP = π
  수렴: P^n → 1π^T (기약·비주기)
  수렴률: ‖μ_n - π‖_TV ≤ C·|λ_2|^n
  ↑ 제2고유값이 mixing time 결정
  
  Detailed Balance: π_i P_{ij} = π_j P_{ji}
    ⇒ π가 정상분포 (자동 만족)
  
  에르고딕 정리:
    (1/n) Σ f(X_k) →^{a.s.} 𝔼_π[f]
  ↑ 시간 평균 = 공간 평균 (MCMC의 이론적 근거)

Poisson 과정 (rate λ)
  세 가지 동치 정의:
  1. 독립증분, N_t - N_s ~ Poisson(λ(t-s))
  2. 간격시간 독립, ~ Exp(λ) (memoryless)
  3. ℙ(N_{t+h} - N_t = 1) = λh + o(h)
        ℙ(N_{t+h} - N_t ≥ 2) = o(h)

마팅게일
  𝔼[M_{n+1} | ℱ_n] = M_n  (공정한 게임)
  
  ├── Doob 수렴 정리: L¹ bounded → a.s. 수렴
  ├── Optional Stopping: 𝔼[M_τ] = 𝔼[M_0]
  ├── Azuma-Hoeffding: bounded differences → concentration
  └── 이차변분 ⟨M⟩: 확률해석의 기초

Brownian Motion B_t
  공리: B_0 = 0, 독립증분, B_t - B_s ~ N(0, t-s), 연속 경로
  
  성질:
  ├── a.s. 연속, but a.s. 어디에서도 미분불가능
  ├── ⟨B⟩_t = t (이차변분이 결정적!)
  ├── Random walk scaling limit (Donsker)
  ├── 반사원리: ℙ(M_t ≥ a) = 2ℙ(B_t ≥ a)
  └── 마팅게일: B_t, B_t² - t, exp(λB_t - λ²t/2) 모두 마팅게일

MCMC (Markov Chain Monte Carlo)
  목표: π에서 샘플 뽑기 (π는 복잡한 분포)
  아이디어: π가 정상분포인 MC 설계

  Metropolis-Hastings:
    x → y ~ q(y|x) (proposal)
    accept with α = min(1, π(y)q(x|y) / π(x)q(y|x))
    
    ⇒ Detailed Balance 만족: π(x)P(x,y) = π(y)P(y,x)
    ⇒ π가 정상분포
    
  Gibbs Sampler:
    조건부 분포 p(x_i | x_{-i})에서 순차 샘플
    MH with α = 1 (특수 경우)
  
  HMC (Hamiltonian Monte Carlo):
    Hamiltonian H(x,p) = U(x) + K(p), U = -log π
    Leapfrog로 해밀토니안 역학 적분
    gradient 활용 → 효율적 제안

  수렴 진단:
  ├── ESS (Effective Sample Size)
  ├── Gelman-Rubin R̂: 여러 체인의 일치도
  └── Autocorrelation 감소 속도 = mixing time

AI/ML 응용
  ├── Bayesian NN: MCMC로 가중치 사후분포 샘플
  ├── Diffusion Model:
  │     Forward = OU process 이산화 (다음 레포)
  │     Reverse SDE로 생성
  ├── RL Q-learning: 정책의 마르코프 체인 수렴 (벨만 연산자)
  ├── LSTM의 시계열 = 조건부 마르코프 과정
  └── Transformer self-attention = non-Markovian (전체 시퀀스 의존)
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (35개 목표)
- Python + NumPy + SciPy 실험 환경
- Probability Theory, Linear Algebra 레포의 어떤 정리를 전제로 사용하는지 명시
- 후속 레포(SDE Deep Dive, Diffusion Model, Bayesian ML, RL Foundations)와의 연결점

**준비됐으면 1단계 구조 설계부터 시작해줘!**
