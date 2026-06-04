# Probability Theory Deep Dive 레포지토리 제작 프롬프트

나는 "Probability Theory Deep Dive" 레포지토리를 만들려고 해.
PMF·PDF에서 시작하는 것과, "확률은 측도(measure)다"라는 **Kolmogorov의 공리적 정의**에서 출발하는 것은 다르다.
VAE의 ELBO를 쓰는 것과, 그것이 **Jensen 부등식 + KL-divergence 비음수성**에서 유도된다는 것을 아는 것은 다르다.
Monte Carlo를 쓰는 것과, 왜 **대수의 법칙**이 $O(1/\sqrt{n})$ 수렴률을 주는지 증명할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "확률은 측도 $\mu$다. 모든 확률 개념은 $(\Omega, \mathcal{F}, \mathbb{P})$에서 출발한다"

**핵심 차별화**:
1. **측도론 기반** — σ-대수에서 시작해 PMF·PDF를 Radon-Nikodym 도함수로 통합
2. **이산과 연속을 쪼개지 않는다** — 기댓값 $\mathbb{E}[X] = \int X \, d\mathbb{P}$ 하나의 정의
3. **수렴 4종을 구분한다** — 확률수렴·거의확실수렴·평균수렴·분포수렴의 강약 관계 완전 증명
4. **AI의 확률 개념을 정밀하게** — ELBO, Evidence Lower Bound가 왜 "Lower" Bound인지 Jensen으로 유도

**타겟 독자**:
- "확률은 확률질량함수"라는 이해로 연속·이산 혼합 분포를 다룰 수 없는 개발자
- Jensen 부등식으로 ELBO를 직접 유도하지 못하는 VAE 사용자
- 대수의 법칙 "강"과 "약"의 차이를 설명 못하는 사람
- 조건부 기댓값을 "조건부 확률의 기댓값"으로만 이해하고, $\sigma$-대수에 대한 함수로 보지 못하는 사람
- 중심극한정리를 암기하지만, 특성함수의 연속 정리로 증명 못하는 사람

**선행 학습**:
- **Linear Algebra Deep Dive** (다변수 정규분포의 공분산 행렬, Spectral Theorem)
- **Calculus & Optimization Deep Dive** (적분·극한·수렴 개념)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 측도론 기초 — 왜 Kolmogorov는 σ-대수를 도입했는가 (6개 문서)
- **확률 공리의 역사** — 고전적·빈도주의·주관주의 확률 한계, Kolmogorov 1933년 공리화의 필요성, Banach-Tarski 역설과 비가측 집합
- **σ-대수(σ-algebra)의 정의** — 가산 합집합으로 닫힌 집합족, Borel σ-대수, 생성된 σ-대수 정의
- **측도(Measure)의 정의** — 가산가법성, Carathéodory 확장정리의 서술, 르베그 측도의 구성
- **확률공간 $(\Omega, \mathcal{F}, \mathbb{P})$** — 샘플공간·사건·확률측도의 3중 구조, $\mathbb{P}(\Omega) = 1$ 정규화
- **르베그 적분의 정의** — 단순함수 → 음이 아닌 함수 → 일반 함수 단계적 구성, 리만 적분과의 차이
- **수렴 정리 — MCT, DCT, Fatou** — 단조수렴·지배수렴·Fatou 보조정리 증명, 언제 $\lim \int = \int \lim$인가

### Chapter 2: 확률변수와 분포 (6개 문서)
- **확률변수의 엄밀한 정의** — $X: \Omega \to \mathbb{R}$이 가측함수라는 조건, 왜 가측이어야 하는가
- **분포함수(CDF)와 Radon-Nikodym** — CDF의 4개 성질 증명, PMF·PDF가 $\mathbb{P}_X$의 Radon-Nikodym 도함수
- **이산과 연속의 통합** — 왜 "연속이면 PDF, 이산이면 PMF"가 불완전한가, 혼합 분포(mixed distribution)의 처리
- **결합분포와 주변분포** — $F_{X,Y}$에서 $F_X$ 추출, 독립성의 정의 $F_{X,Y} = F_X F_Y$와 곱측도
- **변수 변환(Change of Variables)** — $Y = g(X)$의 분포 도출, 야코비안 행렬식이 등장하는 이유의 증명
- **확률분포 갤러리** — Bernoulli, Binomial, Poisson, Normal, Exponential, Gamma, Beta의 유도 관계와 AI 응용

### Chapter 3: 기댓값과 분산 — 적분으로서의 통합 (5개 문서)
- **기댓값의 정의 $\mathbb{E}[X] = \int X \, d\mathbb{P}$** — 단일 정의가 이산·연속을 모두 커버함을 증명
- **Law of the Unconscious Statistician** — $\mathbb{E}[g(X)] = \int g \, d\mathbb{P}_X$의 증명, "g(X)의 기댓값"을 계산하는 두 가지 방법이 일치
- **분산, 공분산, 상관계수의 성질** — Cauchy-Schwarz로 $|\text{Cov}(X,Y)| \leq \sigma_X \sigma_Y$ 증명, 상관계수의 기하학적 의미(두 확률변수의 cos)
- **핵심 부등식들** — Markov, Chebyshev, Jensen, Cauchy-Schwarz 모두 증명 (ML의 많은 정리가 이것들의 응용)
- **적률생성함수(MGF)와 특성함수** — MGF의 미분이 적률을 생성, 특성함수는 항상 존재하고 분포를 유일하게 결정(역정리 진술)

### Chapter 4: 독립성과 조건부 확률 (5개 문서)
- **독립성의 엄밀한 정의** — 사건의 독립, 확률변수의 독립, σ-대수의 독립 3단계, 상호 독립과 쌍독립의 차이(반례)
- **조건부 확률과 Bayes 정리** — $\mathbb{P}(A|B) = \mathbb{P}(A \cap B) / \mathbb{P}(B)$, 조건부 확률도 확률측도임을 증명
- **조건부 기댓값 $\mathbb{E}[X | \mathcal{G}]$** — σ-대수에 대한 조건부 기댓값의 엄밀한 정의 (Kolmogorov), Tower Property 증명
- **조건부 기댓값의 성질** — 선형성, Jensen, Tower Property, Pull-out, 이 성질들이 VAE·EM 알고리즘에서 어떻게 쓰이는가
- **베이즈 추론의 측도론적 기초** — 사전·가능도·사후 분포가 측도 관점에서 어떻게 연결되는가, Bayesian ML의 수학적 기반

### Chapter 5: 수렴 이론 — 4종의 수렴을 구분하기 (6개 문서)
- **수렴의 4종** — 거의확실수렴(a.s.), 확률수렴(in prob.), $L^p$ 수렴, 분포수렴(in distribution)
- **강약 관계 증명** — $\xrightarrow{a.s.} \Rightarrow \xrightarrow{p} \Rightarrow \xrightarrow{d}$, 역이 거짓인 반례들
- **큰 수의 법칙 — 약법칙(WLLN)** — Chebyshev를 이용한 WLLN 완전 증명, $\bar{X}_n \xrightarrow{p} \mu$
- **큰 수의 법칙 — 강법칙(SLLN)** — Kolmogorov의 SLLN 증명 스케치, $\bar{X}_n \xrightarrow{a.s.} \mu$의 의미
- **중심극한정리(CLT)** — 특성함수의 포인트와이즈 수렴 + Lévy 연속정리로 CLT 완전 증명, Lindeberg 조건
- **Monte Carlo의 수학적 근거** — LLN으로 적분 근사, CLT로 오차의 분포 특정, 왜 $O(1/\sqrt{n})$인가

### Chapter 6: 다변수 확률분포와 정규분포 (5개 문서)
- **다변수 정규분포** — $\mathcal{N}(\mu, \Sigma)$의 PDF 유도, 공분산 행렬이 대칭 양의 정부호인 이유
- **Affine 변환과 MVN의 닫힘성** — $Y = AX + b$이면 $Y \sim \mathcal{N}(A\mu + b, A\Sigma A^\top)$ 증명
- **조건부 분포와 Schur Complement** — MVN의 조건부도 MVN인 증명, 조건부 평균·분산의 Schur 보수 공식
- **Gaussian Process 맛보기** — 함수공간의 정규분포, 평균함수와 공분산함수(커널), GP 회귀의 조건부 MVN 유도
- **정보의 기하 — 공분산 행렬의 고유벡터** — 주축(principal axes)과 PCA의 확률적 해석, Mahalanobis 거리의 기하

### Chapter 7: AI/ML에서의 확률 이론 (5개 문서)
- **ELBO 유도 완전판** — VAE의 $\log p(x) \geq \mathbb{E}_q[\log p(x,z)] - \mathbb{E}_q[\log q(z|x)]$를 Jensen과 KL로 두 가지 방법으로 유도
- **Reparameterization Trick의 수학** — $z = \mu + \sigma \epsilon$, $\epsilon \sim \mathcal{N}(0,I)$에서 gradient를 흘릴 수 있는 이유 (변수 변환)
- **MLE의 일관성과 CLT** — 최대가능도 추정량이 $\sqrt{n}(\hat\theta_n - \theta_0) \xrightarrow{d} \mathcal{N}(0, I(\theta_0)^{-1})$, Fisher 정보의 역
- **Dropout의 베이지안 해석** — Gal & Ghahramani 2016, Dropout이 근사 베이지안 추론인 이유의 수학적 유도
- **Concentration Inequalities와 PAC 학습** — Hoeffding, McDiarmid 부등식, 샘플 복잡도 상한 유도의 도구

---

각 챕터는 **5~6개 문서**로 구성해줘. 총 **38개 문서** 목표.

**문서 구조** (측도론 기반의 엄밀한 확률):
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 AI에서 중요한가
## 📐 수학적 선행 조건 (LA·Calc 레포 참조)
## 📖 직관적 이해
   — 이산과 연속 양쪽의 예시
## ✏️ 엄밀한 정의
   — σ-대수, 가측함수 등 형식적 정의
## 🔬 정리와 증명
## 💻 NumPy/SciPy 시뮬레이션으로 검증
   — LLN, CLT 등은 반드시 시뮬레이션
   — 이론 수렴률이 실제로 관찰되는지 확인
## 🔗 AI/ML 연결
   — VAE, GAN, Bayesian NN, MCMC 등
## ⚖️ 가정과 한계
   — 독립성, 유한 분산 등 가정이 깨지면?
   — Cauchy 분포처럼 기댓값이 없는 경우
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **이산/연속 구분하지 않는다** — 기댓값은 항상 $\int X \, d\mathbb{P}$, PMF/PDF는 Radon-Nikodym 도함수
2. **시뮬레이션 필수** — LLN·CLT는 실제로 몬테카를로로 재현해 수렴률 관찰
3. **반례 제시** — "수렴의 강약 관계"는 반드시 반례로 역이 성립하지 않음을 보임
4. **측도 표기의 일관성** — $(\Omega, \mathcal{F}, \mathbb{P})$ 명시, $X \in \mathcal{F}$가 아닌 "$X$는 $\mathcal{F}$-가측"
5. **AI 응용은 "왜 확률 모델이 이렇게 생겼는가"를 측도로 답함** — VAE의 latent variable이 연속 확률변수라는 것의 의미

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0      # 통계분포, stats
matplotlib==3.8.0
seaborn==0.13.0    # 분포 시각화
sympy==1.12
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (CLT 시각화)
import numpy as np
import matplotlib.pyplot as plt
from scipy import stats

# 임의의 비정규 분포 (예: 지수분포)에서 표본평균의 분포
def clt_demo(sample_size, n_trials=10000, dist_fn=np.random.exponential):
    means = [dist_fn(size=sample_size).mean() for _ in range(n_trials)]
    return np.array(means)

# n이 커질수록 정규분포에 수렴
fig, axes = plt.subplots(1, 4, figsize=(16, 3))
for i, n in enumerate([2, 10, 50, 500]):
    means = clt_demo(n)
    # 정규분포와 비교 (평균 1, 분산 1/n — 지수분포의 경우)
    axes[i].hist(means, bins=50, density=True, alpha=0.6)
    x = np.linspace(means.min(), means.max(), 100)
    axes[i].plot(x, stats.norm.pdf(x, loc=1.0, scale=1/np.sqrt(n)), 'r-')
    axes[i].set_title(f'n = {n}')
plt.tight_layout(); plt.show()
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Linear Algebra, Calculus 선행" 명시
   - "Mathematical Statistics, Information Theory, Stochastic Processes로 이어짐" 명시
   - 측도론 기반 접근의 차별점 강조
3. **챕터별 문서 작성**: Chapter 1부터 순서대로 (측도 → 확률변수 → 수렴)

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Probability: Theory and Examples** (Rick Durrett) — 측도론 기반 확률 표준
- **A Probability Path** (Sidney Resnick) — 입문자용 측도론 확률
- **Probability and Measure** (Patrick Billingsley) — 측도론 고전
- **Pattern Recognition and Machine Learning** (Bishop) — ML의 확률 기반
- **Information Theory, Inference, and Learning Algorithms** (MacKay)
- **Kolmogorov, "Foundations of the Theory of Probability" (1933)** — 공리화 원전

---

## 💡 핵심 분석 대상

```
Kolmogorov 공리 (1933)
  │
  ▼
(Ω, ℱ, ℙ) — 확률공간
  Ω: 샘플공간
  ℱ: σ-대수 (사건의 집합)
  ℙ: 확률측도, ℙ(Ω) = 1, 가산가법성
  │
  ▼
확률변수 X: Ω → ℝ (ℱ-가측)
  │
  ├── 분포 ℙ_X (ℝ 위의 측도, push-forward)
  ├── CDF F_X(x) = ℙ(X ≤ x)
  └── PDF/PMF = dℙ_X/dμ (Radon-Nikodym, μ는 Lebesgue/counting 측도)
        │
        ▼
기댓값 𝔼[X] = ∫_Ω X dℙ = ∫_ℝ x dℙ_X(x) = ∫_ℝ x f_X(x) dx
  │
  ▼
독립성, 조건부 확률, 조건부 기댓값 𝔼[X|𝒢]
  ├── Tower: 𝔼[𝔼[X|𝒢]] = 𝔼[X]
  ├── Pull-out: 𝔼[YX|𝒢] = Y·𝔼[X|𝒢] if Y ∈ 𝒢-가측
  └── Jensen: φ convex ⇒ 𝔼[φ(X)|𝒢] ≥ φ(𝔼[X|𝒢])
        │
        ▼
수렴 이론
  a.s.  ────┐
            ├── LLN (Law of Large Numbers)
  in prob. ─┤       (약법칙 + 강법칙)
            │
  L^p    ───┘
  
  in dist. ← CLT (Central Limit Theorem)
           ← 특성함수 방법 + Lévy 연속정리

AI/ML 응용:
  ├── VAE: log p(x) ≥ ELBO = 𝔼_q[log p(x,z)] - 𝔼_q[log q(z|x)]
  │        ⇒ Jensen 부등식 + KL(q ‖ p)
  ├── GAN: minimax 게임 = JS-divergence 최소화 (원 이론)
  ├── MLE 점근: √n(θ̂ - θ₀) →^d 𝒩(0, I(θ₀)^{-1})  ⇒ CLT
  ├── Bayesian NN: 사후 p(w|D) ∝ p(D|w) p(w)  ⇒ Bayes 정리
  ├── MCMC: 에르고딕 정리 + 마르코프 체인의 정상분포 수렴
  └── Dropout: Bernoulli 확률변수로 근사 베이지안 추론 (Gal 2016)

측도론이 왜 필요한가 — 구체적 반례:
1. Cauchy 분포: 기댓값이 없음 (∫ |x| dℙ = ∞)
   → "평균"이라는 개념을 단순히 ∑ x·P(X=x)로 정의 불가
2. Brownian Motion의 전체경로공간: 비가산 Ω, 비자명 측도론 필요
3. 혼합 분포 (이산 + 연속): PMF 또는 PDF 한쪽으로만 못 씀
4. 조건부 확률: ℙ(B) = 0인 B에 대해서는 "단순 정의"가 실패
   → Radon-Nikodym으로 조건부 기댓값을 재정의
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~6개씩)
- 각 문서가 다루는 핵심 정리·증명·AI 응용 (3~4줄)
- 전체 문서 개수 확인 (38개 목표)
- Python + NumPy + SciPy 실험 환경
- Linear Algebra, Calculus 레포의 어떤 정리를 전제로 사용하는지 명시
- 후속 레포(Mathematical Statistics, Information Theory, Stochastic Processes, SDE)와의 연결점

**준비됐으면 1단계 구조 설계부터 시작해줘!**
