# Mathematical Statistics Deep Dive 레포지토리 제작 프롬프트

나는 "Mathematical Statistics Deep Dive" 레포지토리를 만들려고 해.
확률 문제를 푸는 것과, "**관측으로부터 파라미터를 역추정하는 구조**"의 수학을 아는 것은 다르다.
`scipy.stats.ttest_ind`를 호출하는 것과, **귀무가설 하에서 통계량이 왜 t-분포를 따르는가**를 증명할 수 있는 것은 다르다.
MLE를 쓰는 것과, MLE가 **점근적으로 정규분포를 따르고 그 분산이 Fisher 정보의 역**이라는 Cramér-Rao 경계를 증명할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "통계학은 '관측 데이터로 파라미터를 거꾸로 추론하는 수학'이다. 추정량의 성질을 분포 이론으로 증명한다"

**핵심 차별화**:
1. **추정량을 확률변수로 본다** — $\hat\theta$는 데이터의 함수이자 확률변수, 그 분포를 유도
2. **점근 이론에 집중** — MLE의 일관성·정규성·효율성을 완전 증명 (CLT와 델타 방법)
3. **가설검정의 수학** — 검정력·유의수준의 엄밀한 정의, Neyman-Pearson 보조정리 증명
4. **베이즈와 빈도주의 모두** — 두 관점의 철학적·수학적 차이를 명확히 서술

**타겟 독자**:
- p-value를 "효과가 존재할 확률"로 잘못 이해하는 연구자
- MLE를 쓰지만 왜 MLE가 "점근적 최적"인지 Cramér-Rao로 증명 못하는 개발자
- Confidence Interval과 Credible Interval의 차이를 수학적으로 설명 못하는 사람
- 왜 표본평균의 분산이 $\sigma^2/n$이고 표본분산이 $(n-1)$로 나누는지 증명 못하는 사람
- A/B 테스트를 하면서 "통계적 검정력이 80%" 같은 표현의 실제 의미를 모르는 사람

**선행 학습**:
- **Linear Algebra Deep Dive** (다변수 정규분포, 이차형식의 분포)
- **Calculus & Optimization Deep Dive** (MLE가 로그가능도의 최대화, 헤시안)
- **Probability Theory Deep Dive** (측도, 확률변수, 수렴 이론, CLT) — **필수**

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 통계적 모형과 표본이론 (5개 문서)
- **통계 모델(Statistical Model)의 정의** — $\{\mathbb{P}_\theta : \theta \in \Theta\}$ 모수 공간 위의 분포족, 식별가능성(identifiability)
- **표본 통계량(Statistic)과 표집분포** — 통계량은 데이터의 가측함수, 표본평균·표본분산의 분포 유도
- **표본평균의 분포** — $\bar{X} \sim \mathcal{N}(\mu, \sigma^2/n)$ 유도 (MGF 또는 CLT), 유한표본 vs 점근
- **표본분산과 카이제곱 분포** — $(n-1)S^2/\sigma^2 \sim \chi^2_{n-1}$ 완전 증명 (코크란 정리)
- **t-통계량과 F-통계량의 유도** — $T = (\bar{X} - \mu)/(S/\sqrt{n})$가 왜 $t_{n-1}$인가, $F$-통계량의 비율 구조

### Chapter 2: 충분통계량과 지수족 (5개 문서)
- **충분통계량(Sufficient Statistic)** — Fisher-Neyman 인수분해 정리 완전 증명, 데이터에서 파라미터에 관한 모든 정보를 담는다는 것의 의미
- **최소충분통계량(Minimal Sufficient)** — Lehmann-Scheffé 정리, 파라미터 식별에 필요한 "최소" 통계량
- **완비성(Completeness)** — 완비·충분 통계량의 정의, UMVUE 유도에서의 역할
- **지수족(Exponential Family)의 정의와 성질** — 자연 파라미터·자연 통계량, 왜 대부분의 분포가 지수족인가
- **지수족과 MLE의 닫힌 형태** — 지수족에서 MLE가 적률 방정식의 해가 되는 이유

### Chapter 3: 점추정 (Point Estimation) (6개 문서)
- **추정량의 성질** — 불편성(unbiasedness), 일관성(consistency), 효율성(efficiency), 충분성(sufficiency)의 정의와 관계
- **Cramér-Rao 하한(CRLB)** — Fisher 정보 $I(\theta)$의 정의, $\text{Var}(\hat\theta) \geq 1/I(\theta)$ 완전 증명
- **UMVUE와 Rao-Blackwell 정리** — 충분통계량에 조건을 걸면 분산이 줄어드는 이유 증명, Lehmann-Scheffé 정리
- **MLE(Maximum Likelihood Estimator)** — 로그가능도 최대화, MLE가 적률법(Method of Moments)보다 일반적인 이유
- **MLE의 점근 정규성** — $\sqrt{n}(\hat\theta_n - \theta_0) \xrightarrow{d} \mathcal{N}(0, I(\theta_0)^{-1})$ 완전 증명, 여기서 CLT와 델타 방법이 결합
- **MAP(Maximum A Posteriori)와의 관계** — MLE = MAP with uniform prior, regularization의 베이지안 해석

### Chapter 4: 구간추정과 가설검정 (6개 문서)
- **신뢰구간의 엄밀한 정의** — "95% CI"가 무슨 의미인가, 빈도주의 해석의 정확한 서술
- **Pivotal Quantity로 CI 구성** — 추축량(pivot)의 정의, Z·t·$\chi^2$·F 분포 기반 CI 유도
- **가설검정의 프레임워크** — 귀무가설 $H_0$, 대립가설 $H_1$, 검정통계량, 기각역의 정의
- **제1종·제2종 오류와 검정력** — 유의수준 $\alpha$, 검정력 $1 - \beta$, 표본크기 계산 공식 유도
- **Neyman-Pearson 보조정리** — 최적 기각역은 가능도비, 단순 vs 단순 가설에서 UMP 검정의 존재 증명
- **UMP 검정과 단조 가능도비** — 일반 가설에서 UMP가 존재하는 조건(MLR), 왜 지수족에서 자주 성립하는가

### Chapter 5: 점근 이론 (Asymptotic Theory) (5개 문서)
- **델타 방법(Delta Method)** — $\sqrt{n}(g(\hat\theta_n) - g(\theta_0)) \xrightarrow{d} \mathcal{N}(0, g'(\theta_0)^2 / I(\theta_0))$ 증명, 변환 후 점근 분산
- **일관성의 증명 기법** — Slutsky 정리, 연속 사상 정리, 일관성 증명의 표준 절차
- **LRT(Likelihood Ratio Test)의 점근 분포** — $-2 \log \Lambda \xrightarrow{d} \chi^2_{k}$ 윌크스 정리 증명
- **Wald 검정, Score 검정, LRT의 점근 등가성** — 세 검정이 귀무 하에서 동일한 점근 분포, 실전에서 어느 것을 쓸 것인가
- **M-추정량과 일반 점근 이론** — MLE를 포함하는 일반 추정량 클래스, 점근 분산의 샌드위치 공식

### Chapter 6: 베이지안 추론 (5개 문서)
- **베이즈 정리의 연속 버전** — $p(\theta | x) = p(x|\theta) p(\theta) / p(x)$, 증거(evidence) $p(x) = \int p(x|\theta) p(\theta) d\theta$
- **공액 사전분포(Conjugate Prior)** — Beta-Binomial, Gamma-Poisson, Normal-Normal 유도, 왜 사후가 같은 분포족?
- **무정보 사전분포와 Jeffreys Prior** — Jeffreys' $\pi(\theta) \propto \sqrt{I(\theta)}$가 reparameterization 불변인 이유 증명
- **베이지안 추론의 점근 — Bernstein-von Mises 정리** — 표본크기가 크면 사후 분포가 MLE를 중심으로 한 정규분포에 수렴, 사전의 영향 소멸
- **Credible Interval vs Confidence Interval** — 두 구간의 의미가 근본적으로 다른 이유, 하지만 대부분의 경우 비슷하게 보이는 수학적 근거

### Chapter 7: AI/ML에서의 통계 이론 (5개 문서)
- **Empirical Risk Minimization과 MLE** — ERM 프레임워크, 많은 ML 손실함수가 가능도의 로그로 유도됨 (MSE ↔ Gaussian MLE, Cross-Entropy ↔ Multinomial MLE)
- **Regularization의 베이지안 해석** — L2 = Gaussian prior, L1 = Laplace prior, Dropout = 근사 베이지안
- **GLM(일반화선형모형)과 지수족** — Logistic Regression, Poisson Regression이 지수족 GLM, Link 함수의 의미
- **통계적 학습 이론의 맛보기** — VC 차원, Rademacher 복잡도로 연결되는 일반화 오차 경계 (Stat. Learning Theory 레포에서 심화)
- **확률적 딥러닝 — Bayesian Neural Network** — 가중치의 사후 분포, VI(Variational Inference)로 근사, Laplace Approximation

---

각 챕터는 **5~6개 문서**로 구성해줘. 총 **37개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 이론이 AI에서 중요한가
## 📐 수학적 선행 조건 (LA, Calc, Prob 레포 참조)
## 📖 직관적 이해
## ✏️ 엄밀한 정의
   — 추정량·분포족·가설 등 명확히
## 🔬 정리와 증명
   — Cramér-Rao, Neyman-Pearson, Wilks 등
## 💻 시뮬레이션 검증
   — MLE의 점근 분포를 몬테카를로로 확인
   — CI의 coverage rate 경험적 측정
## 🔗 AI/ML 연결
   — ERM, GLM, Bayesian NN, A/B 테스트 등
## ⚖️ 가정과 한계
   — 모델 설정(model specification)이 틀렸을 때
   — Regularity conditions 위반 시
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **추정량을 "확률변수"로 취급** — 표본 $X_1,...,X_n$이 iid일 때 $\hat\theta(X_1,...,X_n)$의 분포를 항상 의식
2. **점근 결과는 유한표본 실험과 비교** — "n → ∞"의 결과가 n=100에서도 유효한지 시뮬레이션
3. **가설검정의 철학적 엄밀성** — "p-value = 효과가 존재할 확률"은 틀린 해석, 정확한 정의 반복 강조
4. **베이즈와 빈도주의 병치** — 같은 문제를 두 관점으로 풀어 비교
5. **ML 응용은 반드시 포함** — 모든 통계 개념을 ML의 구체 예시와 연결

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0      # stats
statsmodels==0.14.0  # 회귀, 가설검정
matplotlib==3.8.0
seaborn==0.13.0
pymc==5.10.0       # 베이지안 추론
arviz==0.17.0      # 베이지안 진단
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (MLE 점근 정규성 검증)
import numpy as np
import matplotlib.pyplot as plt
from scipy import stats

# 지수분포 λ의 MLE: λ̂ = 1/X̄
# 이론적 점근 분포: √n(λ̂ - λ) →^d 𝒩(0, λ²)
# ( Fisher 정보 I(λ) = 1/λ² )

true_lambda = 2.0
n_values = [10, 30, 100, 500]
n_trials = 10000

fig, axes = plt.subplots(1, 4, figsize=(16, 3))
for i, n in enumerate(n_values):
    mle_estimates = []
    for _ in range(n_trials):
        X = np.random.exponential(1/true_lambda, size=n)
        lam_hat = 1 / X.mean()
        mle_estimates.append(lam_hat)
    
    # 정규화: √n(λ̂ - λ)
    normalized = np.sqrt(n) * (np.array(mle_estimates) - true_lambda)
    
    # 이론적 분포: 𝒩(0, λ²)
    axes[i].hist(normalized, bins=60, density=True, alpha=0.6)
    x = np.linspace(normalized.min(), normalized.max(), 200)
    axes[i].plot(x, stats.norm.pdf(x, 0, true_lambda), 'r-', lw=2)
    axes[i].set_title(f'n = {n}')
plt.tight_layout(); plt.show()
# n이 커질수록 이론 분포와 일치함을 확인
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Probability Theory 필수 선행" 명시
   - 빈도주의와 베이지안 양쪽을 다룸 강조
   - "추정량은 확률변수, 그 분포를 유도한다"가 핵심 메시지
3. **챕터별 문서 작성**: 표본이론 → 충분성 → 점추정 → 가설검정 → 점근 → 베이지안 → ML응용

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Statistical Inference** (Casella & Berger) — 수리통계학 표준
- **Theory of Point Estimation** (Lehmann & Casella) — 점추정 고전
- **Testing Statistical Hypotheses** (Lehmann & Romano) — 가설검정 고전
- **Asymptotic Statistics** (van der Vaart) — 점근 이론 표준
- **Bayesian Data Analysis** (Gelman et al.) — 베이지안 추론 표준
- **All of Statistics** (Wasserman) — ML 독자용 요약서

---

## 💡 핵심 분석 대상

```
통계 모델: {ℙ_θ : θ ∈ Θ}
  │
  ▼
데이터 X_1, ..., X_n ~ iid ℙ_θ
  │
  ▼
통계량 T(X_1,...,X_n) = 데이터의 함수 (자신도 확률변수)
  │
  ├── 표본평균 X̄_n
  ├── 표본분산 S²_n
  ├── MLE θ̂
  └── 이들의 표집분포는 무엇인가?
        │
        ▼
점추정의 성질
  ├── 불편성: 𝔼[θ̂] = θ
  ├── 일관성: θ̂_n →^p θ
  ├── 효율성: Var(θ̂) = Cramér-Rao 하한
  │     ↕ Cramér-Rao: Var(θ̂) ≥ 1/(n·I(θ))
  └── 충분성: θ̂가 충분통계량의 함수
        │
        ▼
MLE의 점근 성질 (n → ∞)
  √n(θ̂_MLE - θ₀) →^d 𝒩(0, I(θ₀)^{-1})
  
  증명 구조:
  1. 일관성 (θ̂ →^p θ₀)
  2. 스코어 함수 0 주변에서 Taylor 전개
  3. CLT로 스코어의 분포 규명
  4. Slutsky로 결합

가설검정 프레임워크
  H₀: θ ∈ Θ₀  vs  H₁: θ ∈ Θ₁
  
  검정통계량 T(X), 기각역 R
  제1종 오류: α = ℙ_θ₀(T ∈ R)  (유의수준)
  제2종 오류: β(θ) = ℙ_θ(T ∉ R) for θ ∈ Θ₁
  검정력: 1 - β(θ)
  
  Neyman-Pearson 보조정리:
    가장 강력한 검정(MP)은 가능도비 형태
    T = p(x;θ₁)/p(x;θ₀), R = {T > k}
  
  점근 검정 3종:
  ├── Wald: (θ̂ - θ₀)² · I(θ̂)  →^d χ²
  ├── Score: U(θ₀)² / I(θ₀)  →^d χ²
  └── LRT: -2 log Λ  →^d χ² (Wilks 정리)

베이지안 추론
  p(θ | x) = p(x|θ) p(θ) / p(x)
  
  공액 사전:
  Beta(α,β) × Bernoulli(n,p) → Beta(α+k, β+n-k)
  N(μ₀, σ₀²) × N(x | μ, σ²) → N(μ_post, σ_post²)
  
  Bernstein-von Mises: n → ∞이면 사후가 MLE 중심 정규분포에 수렴

ML 응용:
  ├── MSE 손실 ⇔ Gaussian MLE
  ├── Cross-entropy ⇔ Multinomial MLE
  ├── L2 reg ⇔ Gaussian prior (MAP)
  ├── L1 reg ⇔ Laplace prior
  ├── Dropout ⇔ 근사 베이지안 (Gal 2016)
  ├── BNN: 가중치의 사후 분포
  └── A/B 테스트: 가설검정 + 표본크기 계산
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~6개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (37개 목표)
- Python + SciPy + statsmodels + PyMC 실험 환경
- Probability Theory 레포의 어떤 정리를 전제로 사용하는지 명시
- 후속 레포(Convex Optimization, Information Theory, Information Geometry, Bayesian ML)와의 연결점

**준비됐으면 1단계 구조 설계부터 시작해줘!**
