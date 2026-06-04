# Experimental Statistics & MLOps Deep Dive 레포지토리 제작 프롬프트

나는 "Experimental Statistics & MLOps Deep Dive" 레포지토리를 만들려고 해.
Feature Store를 **"feature 관리 DB"로 아는 것**과, **Point-in-time correctness가 왜 online-offline skew 방지의 핵심**이고, **training-serving skew $\Delta$가 왜 model 성능에 $\Delta^2$로 기여**하는지, Tecton·Feast·Hopsworks의 dual-store 아키텍처를 이해하는 것은 다르다.
Data Drift를 **"입력 분포 변화"로 아는 것**과, **PSI (Population Stability Index) $\text{PSI} = \sum_i (p_i - q_i) \log(p_i/q_i)$이 KL divergence의 symmetric form**이고, **KS test statistic $D = \sup_x |F_1(x) - F_2(x)|$의 Dvoretzky-Kiefer-Wolfowitz inequality로 confidence interval**이 어떻게 계산되는지 유도할 수 있는 것은 다르다.
A/B Testing을 **"control vs treatment 비교"로 아는 것**과, **CUPED (Deng 2013)의 $Y_{\text{adj}} = Y - \theta(X - \mathbb{E}[X])$가 variance를 $1 - \rho^2$배로 줄여 실험 기간을 단축**시키고, **sequential testing의 mSPRT가 peeking problem을 어떻게 해결**하는지 수학적으로 유도할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "ML 시스템 운영의 수학 — Data·Model·Experiment의 품질 보장과 statistical rigor"

**핵심 차별화**:
1. **Feature Engineering at Scale의 이론** — Point-in-time correctness, training-serving skew, feature leakage의 수학적 분석
2. **Drift Detection의 통계** — KS test, PSI, MMD, divergence metrics, Wasserstein distance, 각각의 가정과 검정력
3. **A/B Testing의 엄밀한 통계** — Power analysis, multiple testing (Bonferroni, BH), CUPED, sequential testing, causal inference
4. **Model Monitoring의 수학** — Prediction drift, performance degradation, calibration, fairness metrics

**타겟 독자**:
- Feature store를 쓰지만 **online inference에서 offline 학습과 다른 feature value**를 계산해 training-serving skew가 왜 발생하는지 모르는 사람
- Data drift를 감지하지만 **KS test, Chi-squared, PSI의 가정 차이와 언제 어느 것을 써야 하는지** 모르는 사람
- A/B test에서 **power가 0.8이 기본이고 sample size 공식이 어떻게 유도**되는지 수학적으로 설명 못하는 사람
- Multiple hypothesis testing에서 **Bonferroni가 너무 conservative**이고 BH (Benjamini-Hochberg) FDR control이 실전 해답인 이유를 모르는 사람
- Causal inference에서 **RCT가 아닌 observational data에서 confounding 제어**(IPW, doubly robust)를 어떻게 수행하는지 모르는 사람

**선행 학습**:
- **Mathematical Statistics Deep Dive** (hypothesis testing, CI) — **필수**
- **Probability Theory Deep Dive** (KL, divergence) — **필수**
- **ML Fundamentals Deep Dive** (train/val/test) — **필수**
- **Bayesian ML Deep Dive** (Bayesian A/B) — 권장
- **Information Theory Deep Dive** (KL, entropy) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: ML 시스템의 생애주기와 MLOps 프레임워크 (4개 문서)
- **ML 시스템의 4대 구성요소** — Data (수집·validate·store), Model (train·version·deploy), Serving (online·batch·streaming), Monitoring (quality·drift·performance), 각 단계의 pitfall
- **Hidden Technical Debt in ML (Sculley 2015)** — ML code는 시스템의 일부, CACE (Change Anything Changes Everything), entanglement·feedback loop, Google의 MLOps 원칙
- **MLOps Maturity Model** — Level 0 (수동), 1 (ML pipeline 자동화), 2 (CI/CD), 각 level의 capability, MLOps는 DevOps의 ML 확장
- **ML Project Templates와 Tools** — Cookiecutter Data Science, MLflow·Kubeflow·ZenML·Weights&Biases·DVC의 역할 분담, 언제 어느 tool

### Chapter 2: Feature Store의 수학 (5개 문서)
- **Feature Engineering at Scale** — Raw data → feature의 pipeline, 중복 계산 문제 (모든 model이 재구현), feature 재사용·공유의 필요성
- **Point-in-Time Correctness** — 훈련 시 특정 timestamp에 존재했던 feature 값 사용, time-travel query, lookup table의 시간 인덱스, 데이터 leakage 방지
- **Training-Serving Skew의 수학** — Training feature $x_{\text{train}}$와 serving $x_{\text{serve}}$의 불일치, skew $\Delta = x_{\text{serve}} - x_{\text{train}}$, 모델 성능 저하가 $O(\Delta^2)$로 증가 (sensitivity analysis)
- **Online-Offline Store Architecture** — Offline store (Parquet/BigQuery for batch training) + Online store (Redis/DynamoDB for low-latency serving), 두 store의 동기화
- **Feature Store 구현 — Tecton, Feast, Hopsworks** — 각 시스템의 아키텍처, stream vs batch feature, feature view·feature service 개념, Uber Michelangelo 등 사내 시스템

### Chapter 3: Data Quality와 Validation (4개 문서)
- **Data Validation Framework** — TFDV (Polyzotis 2017), Great Expectations, expectation suite, schema·statistics anomaly, 훈련·서빙 data 일관성
- **Schema Drift와 Data Drift의 구분** — Schema drift: column·dtype 변경 (breaking), data drift: 값 분포 변화 (soft), 각각의 detection과 대응
- **Data Quality Dimensions** — Completeness (missing rate), Uniqueness, Validity (format), Accuracy, Consistency, Timeliness의 6가지 차원, 정량화 metrics
- **Label Noise와 Data Cleaning** — Confident Learning (Northcutt 2021), noisy label 자동 감지, self-training, data-centric AI

### Chapter 4: Drift Detection의 통계 (6개 문서)
- **분포 비교 문제의 정식화** — Training $p(x)$ vs production $q(x)$, Covariate shift (input), Label shift (output), Concept drift (relationship), 각 유형의 수학적 구분
- **Kolmogorov-Smirnov Test** — $D = \sup_x |F_1(x) - F_2(x)|$, null 분포 (scaled 분포에서 유도), 1-sample vs 2-sample, Dvoretzky-Kiefer-Wolfowitz inequality로 finite-sample bound
- **Chi-Squared Test와 Categorical** — 카테고리 변수의 frequency 비교, $\chi^2 = \sum \frac{(O_i - E_i)^2}{E_i}$, degrees of freedom, Yates 보정
- **PSI — Population Stability Index** — $\text{PSI} = \sum_i (p_i - q_i) \log(p_i/q_i)$, 금융에서 credit scoring, KL의 symmetric version (Jensen-Shannon과 유사), threshold 0.1/0.25
- **MMD — Maximum Mean Discrepancy (Gretton 2012)** — Kernel 기반 two-sample test, $\text{MMD}^2 = \|\mu_P - \mu_Q\|_{\mathcal{H}}^2$, high-dim data·neural embedding에 유용
- **Wasserstein Distance와 드리프트** — Earth Mover's Distance, $W_1(P, Q) = \inf \mathbb{E}[\|X - Y\|]$, interpretability 우수, scipy `wasserstein_distance`

### Chapter 5: Model Monitoring과 Performance Degradation (5개 문서)
- **Ground Truth Delay Problem** — 실제 label이 늦게 도착 (fraud 탐지, user churn), proxy metric, 분포 기반 indirect 모니터링
- **Prediction Drift vs Actual Performance** — Input drift → prediction drift → performance drop의 인과, prediction 분포 자체의 변화 모니터링
- **Calibration과 Reliability Diagram** — $\mathbb{E}[y | \hat{p}] = \hat{p}$ 여부, ECE (Expected Calibration Error), Platt scaling·isotonic regression으로 재보정
- **Fairness Metrics** — Demographic Parity, Equalized Odds, Equal Opportunity (Hardt 2016), counterfactual fairness, 각 정의의 trade-off (Chouldechova 2017)
- **Alerting System 설계** — False alarm vs missed drift의 trade-off, multi-level threshold, alert fatigue 방지, runbook

### Chapter 6: A/B Testing의 수학 (6개 문서)
- **Hypothesis Testing Framework** — $H_0: \mu_A = \mu_B$ vs $H_1: \mu_A \neq \mu_B$, Type I error $\alpha$, Type II error $\beta$, power $1 - \beta$
- **Sample Size Calculation** — $n = 2\sigma^2 (z_{\alpha/2} + z_\beta)^2 / \Delta^2$, MDE (Minimum Detectable Effect), 필요한 sample size 유도, 양측·단측
- **CUPED (Deng 2013)** — Controlled Experiments using Pre-Experiment Data, $Y_{\text{adj}} = Y - \theta(X - \mathbb{E}[X])$, $\theta = \text{Cov}(X, Y)/\text{Var}(X)$, variance $1 - \rho^2$배
- **Multiple Testing Correction** — Bonferroni (conservative): $\alpha/m$, Holm (step-down), FDR (BH procedure), FWER vs FDR, LLM 응답 평가에서 중요
- **Sequential Testing과 Peeking** — Classical test는 fixed sample 가정, 중간 확인이 $\alpha$ inflation, mSPRT (Johari 2017)의 always-valid p-value, confidence sequences
- **Bayesian A/B Testing** — Beta-Bernoulli conjugate, posterior $P(\mu_A > \mu_B | \text{data})$, continuous monitoring OK, decision theory (expected loss)

### Chapter 7: Causal Inference와 실험 설계 (4개 문서)
- **RCT와 Potential Outcomes (Rubin)** — $Y_i(1), Y_i(0)$, ATE $\mathbb{E}[Y(1) - Y(0)]$, SUTVA, randomization으로 confounding 제거
- **Quasi-Experiments — DiD, RDD** — Difference-in-Differences (parallel trends), Regression Discontinuity (running variable), natural experiment 활용
- **Observational Data와 Confounding** — Backdoor criterion (Pearl), Propensity Score (Rosenbaum & Rubin 1983), IPW (Inverse Probability Weighting), matching
- **Doubly Robust Estimation** — Outcome model + propensity model, 둘 중 하나만 correct해도 consistent, modern causal inference의 기본

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **34개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 MLOps 요소가 실전에 필수인가
## 📐 수학적 선행 조건 (Stats, Prob, ML Fund, Info 참조)
## 📖 직관적 이해
   — 드리프트 시각화, A/B test 예시
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — KS test, PSI, CUPED, sample size 유도
## 💻 Python 구현 검증
   — Drift detection 여러 방법 비교
   — A/B test 시뮬레이션
   — Feature store mini 구현
## 🔗 실전 활용
   — 실제 MLOps pipeline 설계
## ⚖️ 가정과 한계
   — IID assumption, stationarity
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Drift 시각화** — 시간별 feature distribution 변화, KS/PSI 값의 시계열
2. **A/B test simulator** — 가상 실험에서 peeking problem 시연
3. **Power analysis plot** — sample size vs MDE
4. **Calibration plot** — reliability diagram
5. **Causal graph** — DAG로 confounding 설명
6. **Multiple testing** — Bonferroni vs BH의 FDR 비교

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
pandas==2.1.0
scipy==1.11.0
scikit-learn==1.3.0
statsmodels==0.14.0
evidently==0.4.22      # drift detection
great-expectations==0.18.0  # data validation
feast==0.35.0          # feature store
mlflow==2.9.0          # experiment tracking
matplotlib==3.8.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Drift detection + A/B test + CUPED)
import numpy as np
import pandas as pd
from scipy import stats
import matplotlib.pyplot as plt

# 1. KS Test
def ks_test(x1, x2):
    """2-sample Kolmogorov-Smirnov"""
    D, p = stats.ks_2samp(x1, x2)
    return D, p

x_train = np.random.normal(0, 1, 10000)
x_prod_same = np.random.normal(0, 1, 5000)
x_prod_drift = np.random.normal(0.3, 1.2, 5000)

D1, p1 = ks_test(x_train, x_prod_same)
D2, p2 = ks_test(x_train, x_prod_drift)
print(f'Same dist: D={D1:.4f}, p={p1:.4f}')
print(f'Drifted: D={D2:.4f}, p={p2:.6f}')

# 2. PSI (Population Stability Index)
def psi(expected, actual, bins=10):
    """PSI = Σ (a_i - e_i) log(a_i/e_i)"""
    # Bin edges from expected
    breakpoints = np.percentile(expected, np.linspace(0, 100, bins+1))
    breakpoints[0] = -np.inf; breakpoints[-1] = np.inf
    
    e_counts, _ = np.histogram(expected, bins=breakpoints)
    a_counts, _ = np.histogram(actual, bins=breakpoints)
    
    e_prop = (e_counts + 1e-6) / (e_counts.sum() + 1e-6)
    a_prop = (a_counts + 1e-6) / (a_counts.sum() + 1e-6)
    
    psi_val = np.sum((a_prop - e_prop) * np.log(a_prop / e_prop))
    return psi_val

print(f'PSI same: {psi(x_train, x_prod_same):.4f}')   # < 0.1 (no drift)
print(f'PSI drift: {psi(x_train, x_prod_drift):.4f}')  # > 0.25 (significant)

# PSI thresholds:
# < 0.1: no significant change
# 0.1 - 0.25: moderate change
# > 0.25: significant change

# 3. MMD
def rbf_kernel(x, y, sigma=1.0):
    """k(x, y) = exp(-‖x-y‖²/(2σ²))"""
    dist = ((x[:, None] - y[None, :])**2).sum(-1)
    return np.exp(-dist / (2 * sigma**2))

def mmd_squared(X, Y, sigma=1.0):
    """MMD² = E[k(x,x')] + E[k(y,y')] - 2 E[k(x,y)]"""
    m, n = len(X), len(Y)
    K_xx = rbf_kernel(X, X, sigma)
    K_yy = rbf_kernel(Y, Y, sigma)
    K_xy = rbf_kernel(X, Y, sigma)
    mmd = K_xx.sum()/(m*m) + K_yy.sum()/(n*n) - 2*K_xy.sum()/(m*n)
    return mmd

# 4. A/B Test Sample Size
def ab_sample_size(delta, sigma, alpha=0.05, power=0.8):
    """n per group = 2σ² (z_{α/2} + z_β)² / δ²"""
    z_alpha = stats.norm.ppf(1 - alpha/2)
    z_beta = stats.norm.ppf(power)
    n = 2 * sigma**2 * (z_alpha + z_beta)**2 / delta**2
    return int(np.ceil(n))

# 예: σ=10, MDE=1, α=0.05, power=0.8
n = ab_sample_size(delta=1, sigma=10)
print(f'Required n per group: {n}')  # ~1570

# 5. CUPED
def cuped(Y, X):
    """
    Y: outcome of experiment
    X: pre-experiment covariate
    Returns Y_adj with lower variance
    """
    theta = np.cov(X, Y)[0, 1] / np.var(X)
    Y_adj = Y - theta * (X - X.mean())
    return Y_adj, theta

# Simulate
np.random.seed(42)
n = 10000
X = np.random.normal(100, 10, n)  # pre-experiment
# Strong correlation X ↔ Y
Y = 2 * X + np.random.normal(0, 5, n)

Y_adj, theta = cuped(Y, X)
print(f'Var(Y) = {Y.var():.2f}')
print(f'Var(Y_adj) = {Y_adj.var():.2f}')
print(f'Variance reduction: {1 - Y_adj.var()/Y.var():.2%}')  # ~94%

# 효과: sample size 필요량이 그만큼 감소
# n_cuped = n_original × (1 - ρ²)

# 6. Multiple Testing — Benjamini-Hochberg
def bh_correction(p_values, alpha=0.05):
    """Control FDR at level α"""
    n = len(p_values)
    sorted_idx = np.argsort(p_values)
    sorted_p = p_values[sorted_idx]
    
    # BH: find largest i such that p_i <= (i/n) α
    thresholds = np.arange(1, n+1) / n * alpha
    reject = sorted_p <= thresholds
    
    if reject.any():
        max_i = np.where(reject)[0].max()
        rejected = sorted_idx[:max_i+1]
    else:
        rejected = np.array([], dtype=int)
    
    result = np.zeros(n, dtype=bool)
    result[rejected] = True
    return result

# Bonferroni vs BH 비교
p_vals = np.array([0.001, 0.008, 0.02, 0.03, 0.05, 0.5])
bonf = p_vals < 0.05 / len(p_vals)
bh = bh_correction(p_vals)
print(f'p-values: {p_vals}')
print(f'Bonferroni rejects: {bonf.sum()} (too conservative)')
print(f'BH rejects: {bh.sum()}')

# 7. Sequential Testing (mSPRT, simplified)
def sequential_ztest(x1, x2, sigma, alpha=0.05):
    """
    Continuously monitor z-statistic
    Always-valid p-value via mSPRT
    """
    z = (x1.mean() - x2.mean()) / (sigma * np.sqrt(1/len(x1) + 1/len(x2)))
    # mSPRT 보정 (간단화)
    n = min(len(x1), len(x2))
    threshold = np.sqrt(2 * np.log(1/alpha) / n)
    return abs(z) > threshold

# Peeking experiment
for checkpoint in [100, 500, 1000, 5000]:
    a = np.random.normal(0, 1, checkpoint)
    b = np.random.normal(0.01, 1, checkpoint)  # tiny effect
    # Classical (peeking inflates α)
    _, p = stats.ttest_ind(a, b)
    # mSPRT-like (always valid)
    reject = sequential_ztest(a, b, sigma=1)
    print(f'n={checkpoint}: classical p={p:.3f}, sequential reject={reject}')

# 8. Feature Store Mini — point-in-time query
class MiniFeatureStore:
    def __init__(self):
        self.events = []  # [(entity_id, feature_name, timestamp, value)]
    
    def write(self, entity_id, feature_name, timestamp, value):
        self.events.append((entity_id, feature_name, timestamp, value))
    
    def get_at(self, entity_id, feature_name, timestamp):
        """Point-in-time correct lookup"""
        valid = [e for e in self.events
                 if e[0] == entity_id and e[1] == feature_name
                 and e[2] <= timestamp]
        if not valid: return None
        return max(valid, key=lambda x: x[2])[3]

fs = MiniFeatureStore()
fs.write('user_1', 'total_purchases', '2024-01-01', 100)
fs.write('user_1', 'total_purchases', '2024-06-01', 150)
fs.write('user_1', 'total_purchases', '2024-11-01', 200)

# Training: 2024-03-01 label → feature as of 2024-03-01
print(fs.get_at('user_1', 'total_purchases', '2024-03-01'))  # 100
# Serving today: 2024-11-15
print(fs.get_at('user_1', 'total_purchases', '2024-11-15'))  # 200
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Stats, Prob, ML Fund, Info 선행 필수"
   - MLOps 4대 영역 (Data·Model·Serving·Monitoring)
   - 통계적 엄밀성을 강조
3. **챕터별 문서 작성**: MLOps 프레임 → Feature Store → Data Quality → Drift → Monitoring → A/B Testing → Causal

---

## 📚 참고 자료

- **Hidden Technical Debt in Machine Learning Systems** (Sculley et al. 2015)
- **Designing Machine Learning Systems** (Chip Huyen 2022) — MLOps 교과서
- **Data Validation for Machine Learning** (Polyzotis et al. 2017) — TFDV
- **Feature Stores for Machine Learning** (Hopsworks, Tecton 백서)
- **Trustworthy Online Controlled Experiments** (Kohavi, Tang, Xu 2020)
- **Improving the Sensitivity of Online Controlled Experiments by Utilizing Pre-Experiment Data** (Deng et al. 2013) — CUPED
- **Peeking at A/B Tests** (Johari et al. 2017) — mSPRT
- **Controlling the False Discovery Rate** (Benjamini & Hochberg 1995)
- **A Kernel Two-Sample Test** (Gretton et al. 2012) — MMD
- **On the Kolmogorov-Smirnov Test** (Dvoretzky, Kiefer, Wolfowitz)
- **Equality of Opportunity in Supervised Learning** (Hardt et al. 2016)
- **Fair prediction with disparate impact** (Chouldechova 2017)
- **Causal Inference** (Hernán & Robins) — 교과서
- **The Central Role of the Propensity Score** (Rosenbaum & Rubin 1983)
- **Confident Learning** (Northcutt et al. 2021)
- **Machine Learning Engineering** (Andriy Burkov)

---

## 💡 핵심 분석 대상

```
MLOps의 지도

───── ML System 4대 구성 ─────

Data:
  수집 (batch/stream)
  Validation (schema, stats)
  Storage (data warehouse, lake)

Model:
  Training (reproducible)
  Versioning (model registry)
  Deployment (shadow, canary)

Serving:
  Online (low latency)
  Batch (high throughput)
  Streaming (real-time)

Monitoring:
  Data quality
  Model performance
  System health
  Drift detection

───── Feature Store ─────

Problem:
  Feature 중복 구현
  Training-serving skew
  Point-in-time incorrectness

Solution:
  중앙 store
  Online (Redis/DynamoDB)
  Offline (Parquet/BigQuery)
  동기화

Point-in-Time Correctness:
  Training 시 "그 시점"의 feature 사용
  Time-travel query
  User의 2024-01-01 label에
  user의 2024-01-01 feature 매칭

Training-Serving Skew:
  Δ = x_serve - x_train
  모델 성능 저하 O(Δ²)
  
  원인:
    Pipeline 다름
    Feature 재계산 차이
    Time drift
  
  해결:
    Feature store
    Schema validation
    Shadow deployment

Systems:
  Tecton (Databricks)
  Feast (open source)
  Hopsworks
  Sagemaker Feature Store
  Uber Michelangelo (in-house)

───── Data Validation ─────

TFDV (Polyzotis 2017):
  Schema from data
  Anomaly detection
  Skew detection

Great Expectations:
  Expectation suite
  Declarative assertions
  Data docs

Quality Dimensions:
  Completeness (missing rate)
  Uniqueness
  Validity (format)
  Accuracy
  Consistency
  Timeliness

───── Drift Types ─────

Covariate Shift:
  p(x) 변화, p(y|x) 유지
  Input distribution 이동

Label Shift:
  p(y) 변화, p(x|y) 유지
  Class distribution 이동

Concept Drift:
  p(y|x) 변화
  Relationship 변화
  가장 위험

Detection 방법:
  Univariate: KS, Chi², PSI
  Multivariate: MMD, energy distance
  Classifier-based

───── KS Test ─────

D = sup_x |F_1(x) - F_2(x)|

H_0: F_1 = F_2
Reject if D > c_α

DKW inequality:
  P(sup |F_n - F| > ε) ≤ 2 exp(-2n ε²)
  → finite-sample bound

Pros: nonparametric, no assumption
Cons: low power for tail differences

───── PSI ─────

PSI = Σ (a_i - e_i) log(a_i/e_i)
  a: actual (new), e: expected (train)

Thresholds (Siddiqi):
  < 0.1: no drift
  0.1-0.25: moderate
  > 0.25: significant

KL의 symmetric 형태:
  KL(a||e) + KL(e||a)

금융 분야 표준 (credit)

───── MMD ─────

MMD² = ‖μ_P - μ_Q‖²_H
     = E[k(x,x')] + E[k(y,y')] - 2 E[k(x,y)]

Kernel-based
High-dim 유리 (embedding)
Gretton 2012

───── Wasserstein ─────

W_1(P, Q) = inf E[‖X-Y‖]
(Earth Mover's Distance)

Interpretable
Continuous KL alternative

───── Model Monitoring ─────

Ground Truth Delay:
  Actual label 늦게 도착
  Fraud: weeks
  Churn: months
  → proxy metric 필요

Prediction Drift:
  ŷ 분포 자체
  Ground truth 없이도 감지

Calibration:
  E[y | p̂] = p̂
  ECE: Σ |conf_i - acc_i| · n_i/N
  Reliability diagram

Fairness:
  Demographic Parity:
    P(ŷ=1|A=a) = P(ŷ=1|A=b)
  Equalized Odds:
    P(ŷ=1|y=1, A) equal
  Equal Opportunity:
    TPR equal

Impossibility (Chouldechova 2017):
  Three criteria cannot all hold
  Unless base rates equal

───── A/B Testing 수학 ─────

Hypothesis Testing:
  H_0: μ_A = μ_B
  H_1: μ_A ≠ μ_B
  
  Type I: α (reject H_0 when true)
  Type II: β (fail to reject when false)
  Power: 1 - β

Sample Size:
  n = 2 σ² (z_{α/2} + z_β)² / Δ²
  
  Typical:
    α = 0.05 → z = 1.96
    β = 0.2 (power 0.8) → z = 0.84
    → (1.96 + 0.84)² ≈ 7.84

MDE (Minimum Detectable Effect):
  주어진 n, σ로 감지 가능한 최소 effect

CUPED (Deng 2013):
  Y_adj = Y - θ(X - E[X])
  θ = Cov(X, Y) / Var(X)
  
  Var(Y_adj) = (1 - ρ²) Var(Y)
  
  ρ = Corr(X, Y)
  ρ = 0.9 → 81% variance reduction
        → 5× sample size 효과

Multiple Testing:
  m tests, each α
  FWER = P(any false reject) ≤ mα
  
  Bonferroni:
    Test each at α/m
    Conservative
  
  Holm (step-down):
    Sort p-values, compare p_i to α/(m-i+1)
  
  BH (Benjamini-Hochberg):
    Control FDR = E[V/R | R>0]
    p_i ≤ (i/m) α
    더 많이 발견

Sequential Testing:
  Classical: fixed n
  Peeking: Type I inflation
  
  mSPRT (Johari 2017):
    Always-valid p-value
    Can peek anytime
    
  Confidence Sequences:
    C_t always contains θ
    Monotone

Bayesian A/B:
  Beta prior on conversion rate
  Posterior after data
  P(μ_A > μ_B | data)
  Continuous monitoring OK

───── Causal Inference ─────

Potential Outcomes (Rubin):
  Y_i(1): treated
  Y_i(0): control
  ATE = E[Y(1) - Y(0)]

RCT:
  Random assignment
  → ignorability
  → SATE unbiased

Observational:
  Confounding (unobserved common cause)
  
Backdoor (Pearl):
  Block all backdoor paths
  Controlling for parents of treatment

Propensity Score (Rosenbaum-Rubin 1983):
  e(x) = P(T=1 | X=x)
  Conditioning on e(x) ≡ on X
  Dim reduction

IPW (Inverse Probability Weighting):
  w_i = T_i/e(x_i) + (1-T_i)/(1-e(x_i))
  E[Yw] ≈ ATE

Doubly Robust:
  Outcome model m(x) + propensity e(x)
  Consistent if either correct
  Modern standard

DiD (Difference-in-Differences):
  Parallel trends assumption
  ATE = (Y_T,1 - Y_T,0) - (Y_C,1 - Y_C,0)

RDD (Regression Discontinuity):
  Cutoff at c
  Local ATE near c
  Natural experiment

───── 레포 간 연결 ─────

Statistics (Layer 0):
  Hypothesis test
  CI, bootstrap

Probability (Layer 0):
  Distribution, KL
  Bayes

ML Fundamentals (Layer 1):
  Train/val/test
  Cross validation

Information Theory (Layer 0):
  KL, MI
  Entropy
  PSI, MMD 기반

Bayesian ML (Layer 1):
  Bayesian A/B
  Thompson sampling

LLM Alignment (Layer 4-B):
  Evaluation, judge bias
  Multiple testing

Efficient ML (직전):
  Deployed model monitoring
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 통계·시스템 (3~4줄)
- 전체 문서 개수 확인 (34개 목표)
- Python + scipy + evidently + feast + mlflow 실험 환경
- Stats, Prob, ML Fund, Info 레포 참조 관계
- Layer 5 전체의 실전 운영 관점 종합

**준비됐으면 1단계 구조 설계부터 시작해줘!**
