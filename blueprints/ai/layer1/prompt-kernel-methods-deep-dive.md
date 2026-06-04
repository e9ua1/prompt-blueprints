# Kernel Methods Deep Dive 레포지토리 제작 프롬프트

나는 "Kernel Methods Deep Dive" 레포지토리를 만들려고 해.
`sklearn.svm.SVC(kernel='rbf')`를 **쓰는 것**과, **kernel trick의 정당성이 Mercer 정리에서 나오고, $k(x, y) = \sum \lambda_n \phi_n(x) \phi_n(y)$의 implicit feature map이 무한차원**임을 증명할 수 있는 것은 다르다.
Gaussian Process를 **사용하는 것**과, **GP의 posterior mean이 kernel ridge regression과 완전히 동치**이고, **GP regression의 예측 분산이 covariance function의 nonlinear update $k_*^T (K + \sigma^2 I)^{-1} k_*$로 shrink**하는 메커니즘을 유도할 수 있는 것은 다르다.
MMD를 **정의하는 것**과, **MMD(p, q) = 0 ⟺ p = q (characteristic kernel)**이며, **GAN 대안 MMD-GAN의 이론적 근거**임을 증명할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "Kernel을 ML의 엔진으로 — SVM·GP·MMD·Kernel PCA를 하나의 RKHS 프레임워크에서"

**핵심 차별화**:
1. **Kernel Method의 통일 관점** — Representer 정리로 SVM, KRR, KPCA, GP가 **모두** $\sum \alpha_i k(\cdot, x_i)$ 형태임을 보이기
2. **SVM의 primal/dual 완전 유도** — 라그랑주 쌍대에서 kernel 등장, KKT 조건으로 support vector 해석
3. **Gaussian Process를 함수 prior로** — GP regression/classification을 Bayesian ML 관점으로, posterior의 mean과 covariance 완전 유도
4. **MMD와 현대 응용** — Two-sample test, MMD-GAN, distribution regression 모두 하나의 RKHS 내적으로

**타겟 독자**:
- SVM을 쓰지만 **쌍대 문제가 왜 $\max \sum \alpha_i - \frac{1}{2}\sum \alpha_i \alpha_j y_i y_j k(x_i, x_j)$ 형태**인지 유도 못하는 사람
- Gaussian Process를 쓰지만 **공분산 함수 선택이 왜 prior 선택**인지, 왜 GP가 "함수에 대한 정규분포"인지 모르는 사람
- Kernel Ridge Regression과 Gaussian Process의 **관계**(정확히 같은 예측)를 모르는 사람
- RBF kernel의 $\sigma$ 파라미터가 **왜 복잡도 정규화와 연결**되는지 설명 못하는 사람
- GAN 훈련이 어려워 MMD-based 생성모델에 관심 있지만 **MMD의 이론적 성질**을 모르는 사람

**선행 학습**:
- **Functional Analysis Deep Dive** (RKHS, Mercer 정리, Moore-Aronszajn) — **필수**
- **Linear Algebra Deep Dive** (스펙트럴 분해, 양정치 행렬) — **필수**
- **Probability Theory Deep Dive** (다변수정규분포) — **필수**
- **Convex Optimization Deep Dive** (Lagrangian 쌍대, KKT) — **필수** (SVM 유도)
- **Mathematical Statistics Deep Dive** (Bayesian 추론) — GP에 필수

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Kernel의 기초와 Mercer 정리 (5개 문서)
- **Positive Definite Kernel의 정의** — $k: \mathcal{X} \times \mathcal{X} \to \mathbb{R}$ 대칭, $\sum \alpha_i \alpha_j k(x_i, x_j) \geq 0$ 모든 $\{x_i, \alpha_i\}$에 대해, 그람 행렬 $K$의 양정치성
- **대표 커널의 목록과 성질** — Linear $x^Ty$, Polynomial $(x^Ty + c)^d$, Gaussian/RBF $\exp(-\|x-y\|^2/2\sigma^2)$, Laplace $\exp(-\|x-y\|/\sigma)$, Sigmoid tanh (PD 아님!), 각각의 PD성 증명
- **Kernel의 연산자 — Sum, Product, Composition** — $k_1 + k_2$, $k_1 \cdot k_2$, $f(x) k(x, y) f(y)$ 모두 PD, 새 커널을 기존 커널로 만들기
- **Mercer 정리의 서술과 해석** — 컴팩트 집합 위의 연속 PD 커널은 $k(x, y) = \sum_n \lambda_n \phi_n(x) \phi_n(y)$ (고유함수 전개), feature map $\phi(x) = (\sqrt{\lambda_n} \phi_n(x))_n$이 $\ell^2$로
- **Characteristic Kernel과 Universal Kernel** — Characteristic: 평균 embedding $\mu_p = \mathbb{E}_p[k(\cdot, X)]$가 $p \mapsto \mu_p$로 단사 → MMD의 근거, Universal: $C(\mathcal{X})$에서 dense(Stone-Weierstrass)

### Chapter 2: RKHS와 Representer 정리 (5개 문서)
- **RKHS 구성 (Moore-Aronszajn)** — PD kernel $k$ 주어지면 유일한 RKHS $\mathcal{H}_k$ 존재, 구성: $\text{span}\{k_x\}$의 내적 $\langle k_x, k_y \rangle = k(x, y)$ 정의, 완비화
- **재생성질과 평가범함수** — $f(x) = \langle f, k_x \rangle_{\mathcal{H}_k}$, 평가 $\delta_x : f \mapsto f(x)$가 유계 선형범함수(Riesz로 유도), $\mathcal{H}_k$ 노름의 의미
- **Representer 정리 완전 증명** — $\min_{f \in \mathcal{H}} \sum L(y_i, f(x_i)) + \Omega(\|f\|)$의 최적해는 $f^* = \sum_{i=1}^n \alpha_i k(\cdot, x_i)$, 증명: $f = f_\parallel + f_\perp$ 분해에서 $f_\perp$가 손실에 영향 안 주고 노름만 증가
- **Representer 정리의 계산적 의미** — 무한차원 $\mathcal{H}_k$의 최적화가 유한차원 $\alpha \in \mathbb{R}^n$ 최적화로 환원 → kernel trick의 수학적 정당성
- **$\mathcal{H}_k$의 함수 공간적 성질** — Gaussian RBF의 $\mathcal{H}_k$는 Sobolev 공간과 관련, 차원별 근사 오차 — 어떤 함수가 RKHS에 속하고 어떤 것이 안 속하는가

### Chapter 3: Support Vector Machine (SVM) (6개 문서)
- **Margin 최대화와 Hard-margin SVM** — 선형 분리 가능 데이터에서 margin $= 2/\|w\|$ 최대화 ⟺ $\min \frac{1}{2}\|w\|^2$ s.t. $y_i(w^Tx_i + b) \geq 1$, 기하학적 유도
- **라그랑주 쌍대와 Dual Form** — Lagrangian $L = \frac{1}{2}\|w\|^2 - \sum \alpha_i(y_i(w^Tx_i + b) - 1)$, KKT로 $w = \sum \alpha_i y_i x_i$, dual $\max \sum \alpha_i - \frac{1}{2}\sum \alpha_i \alpha_j y_i y_j x_i^T x_j$ 유도
- **Kernel SVM** — Dual에서 $x^T y$를 $k(x, y)$로 치환하는 kernel trick, Representer 정리와 일치, 예측 $\hat{y}(x) = \text{sign}(\sum \alpha_i y_i k(x_i, x) + b)$
- **Soft-margin SVM과 Hinge Loss** — Slack $\xi_i \geq 0$로 분리 불가능 데이터 처리, $\min \frac{1}{2}\|w\|^2 + C \sum \xi_i$, Hinge loss $\max(0, 1 - y_i f(x_i))$로 재작성
- **SMO (Sequential Minimal Optimization)** — 한 번에 2개 $\alpha_i, \alpha_j$만 업데이트, 해석적 업데이트 공식 유도, KKT violation 기반 working set 선택
- **SVM Regression (SVR)** — $\epsilon$-insensitive loss $\max(0, |y - f(x)| - \epsilon)$, primal/dual 유도, $\epsilon$의 기하학적 해석

### Chapter 4: Gaussian Process (6개 문서)
- **GP의 정의와 공분산 함수** — GP = "임의 유한집합 $\{x_1, ..., x_n\}$에 대해 $(f(x_1), ..., f(x_n))$이 다변수정규분포", mean function $m(x)$, covariance function $k(x, y)$로 완전 특징화
- **GP Regression — Posterior 유도** — Prior $f \sim GP(0, k)$, 관측 $y = f(x) + \epsilon, \epsilon \sim \mathcal{N}(0, \sigma^2)$, joint Gaussian 조건부로 posterior $f(x_*) | y \sim \mathcal{N}(m_*, \sigma_*^2)$ 공식 유도
- **GP Regression과 Kernel Ridge Regression의 동치** — GP posterior mean $m_* = k_*^T(K + \sigma^2 I)^{-1} y$가 KRR 해와 정확히 일치, 차이: GP는 분산도 제공 (uncertainty)
- **GP Classification과 Laplace Approximation** — Bernoulli likelihood $p(y|f) = \sigma(yf)$, non-Gaussian posterior, Laplace approximation으로 Gaussian 근사, Expectation Propagation 대안
- **Hyperparameter Learning — Marginal Likelihood** — $\log p(y | \theta) = -\frac{1}{2}y^T K_\theta^{-1} y - \frac{1}{2}\log|K_\theta| - \frac{n}{2}\log 2\pi$ 최대화, 자동 Occam's razor(복잡도 페널티 $-\frac{1}{2}\log|K|$)
- **Sparse GP와 Inducing Points** — 풀 GP의 $O(n^3)$ 비용, Inducing points $Z \subset \mathcal{X}$로 FITC/VFE 근사, $O(nm^2)$ 계산, Titsias의 변분 근사

### Chapter 5: Kernel Ridge Regression과 Kernel PCA (4개 문서)
- **Kernel Ridge Regression 완전 유도** — Ridge의 dual 형태, Representer 정리 적용, closed-form $\alpha = (K + \lambda I)^{-1} y$, 예측 $f(x) = k(x)^T (K + \lambda I)^{-1} y$
- **Kernel PCA의 수학** — 특성공간에서 PCA, centered Gram 행렬 $\tilde{K} = K - \mathbf{1}K/n - K\mathbf{1}/n + \mathbf{1}K\mathbf{1}/n^2$의 고유값 분해, projection $\phi(x) \cdot v_k = \sum \alpha_k^i k(x_i, x)$
- **Spectral Clustering — Graph Laplacian의 관점** — Similarity matrix로 graph 구성, $L = D - W$의 고유벡터로 clustering, Kernel PCA의 특수한 경우, Normalized cut과의 관계
- **Kernel k-Means** — Feature space에서 k-means, 명시적 $\phi$ 없이 Gram 행렬로 계산, 임의 모양 클러스터 탐지

### Chapter 6: Maximum Mean Discrepancy (MMD)와 Two-Sample Test (5개 문서)
- **MMD의 정의와 RKHS 해석** — $MMD(p, q; \mathcal{H}) = \|\mu_p - \mu_q\|_\mathcal{H}$, 평균 embedding $\mu_p = \mathbb{E}_{X \sim p}[k(\cdot, X)]$, characteristic kernel일 때 MMD = 0 ⟺ p = q
- **MMD의 샘플 추정량** — $\widehat{MMD}^2 = \frac{1}{n^2}\sum k(x_i, x_j) - \frac{2}{nm}\sum k(x_i, y_j) + \frac{1}{m^2}\sum k(y_i, y_j)$, 불편추정량의 유도
- **Two-Sample Test (Gretton et al. 2012)** — Null $H_0: p = q$, MMD² 분포를 permutation이나 점근분포로 critical value 설정, 분포 비교에 왜 강력한가
- **MMD-GAN과 생성모델** — Generator의 출력 분포와 데이터 분포 간 MMD 최소화, GAN의 adversarial 훈련보다 안정적, kernel 선택의 중요성
- **Kernel Embedding of Distributions 일반화** — Distribution regression ($x \mapsto p_x$의 regression), Hilbert-Schmidt Independence Criterion (HSIC)로 독립성 검정

### Chapter 7: Kernel Method 심화 주제 (4개 문서)
- **Kernel Learning — Multiple Kernel Learning** — 여러 kernel의 볼록 결합 $k = \sum \beta_i k_i$, $\beta$ 학습을 SDP 또는 gradient로, automatic kernel selection
- **Random Features (Rahimi & Recht 2007)** — Bochner 정리로 shift-invariant kernel의 Fourier 표현, 유한차원 random feature $\phi(x) = \sqrt{2/D}(\cos(\omega_i^T x + b_i))_i$로 근사, 빅데이터 대응
- **Deep Kernel Learning** — Neural Network feature $\phi_\theta(x)$ 위에 kernel $k(\phi_\theta(x), \phi_\theta(y))$, GP와 NN의 하이브리드, Wilson et al. 2016
- **Neural Tangent Kernel (NTK) 연결** — 무한폭 NN의 NTK가 특정 RKHS의 재생핵, NN 훈련이 NTK-RKHS의 kernel regression과 동치 (Jacot et al. 2018), Layer 2 Generalization Theory로 이어짐

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **35개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 방법이 ML에서 중요한가
## 📐 수학적 선행 조건 (Functional Analysis 레포 핵심 참조)
## 📖 직관적 이해
   — "kernel이 비선형성을 선형 방법에 주입" 중심 비유
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Mercer, Representer, GP posterior, MMD=0 ⟺ p=q
## 💻 NumPy 구현 검증
   — SVM dual 직접 구현, GP 바닥부터 구현
   — Random Features로 빅데이터 kernel 근사
## 🔗 실전 활용
   — 언제 kernel이 유리한가, 언제 딥러닝이 이기는가
## ⚖️ 가정과 한계
   — $O(n^2)$ 메모리, $O(n^3)$ 계산, kernel 선택 감도
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **FA 레포의 Chapter 5(RKHS)와 상보성** — FA는 이론적 정초, 이 레포는 ML 알고리즘 중심
2. **"하나의 Representer 정리로 모든 kernel method 통일"** — SVM, KRR, KPCA, GP가 같은 형태임을 반복 강조
3. **GP 분산 시각화 필수** — 1D regression에서 예측 + 신뢰구간 plot, uncertainty가 데이터에서 멀어질 때 증가하는 것 보여주기
4. **Kernel 비교 시각화** — 같은 데이터에 Linear, RBF, Polynomial 적용해 decision boundary 차이
5. **sklearn 결과와 바닥 구현 비교** — SVM, KRR, GP 모두 바닥부터 + sklearn 결과 일치 확인
6. **Random Features로 scaling** — 모든 주요 kernel method에 Random Features 구현 포함

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
matplotlib==3.8.0
cvxpy==1.4.0           # SVM dual 최적화
scikit-learn==1.3.0    # 검증용
GPy==1.13.0            # GP 비교용
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Gaussian Process regression 바닥부터 + MMD 계산)
import numpy as np
import matplotlib.pyplot as plt

def rbf_kernel(X, Y, sigma=1.0):
    d = np.sum((X[:, None] - Y[None, :]) ** 2, axis=-1)
    return np.exp(-d / (2 * sigma**2))

# 훈련 데이터
np.random.seed(42)
X_train = np.random.uniform(-3, 3, 20).reshape(-1, 1)
y_train = np.sin(X_train).flatten() + 0.1 * np.random.randn(20)

# GP prior: mean 0, RBF kernel
sigma_k, sigma_n = 1.0, 0.1
X_test = np.linspace(-5, 5, 200).reshape(-1, 1)

# Posterior (Rasmussen & Williams 2.2)
K = rbf_kernel(X_train, X_train, sigma_k)
K_s = rbf_kernel(X_train, X_test, sigma_k)
K_ss = rbf_kernel(X_test, X_test, sigma_k)

L = np.linalg.cholesky(K + sigma_n**2 * np.eye(len(X_train)))
alpha = np.linalg.solve(L.T, np.linalg.solve(L, y_train))

# Posterior mean
mu = K_s.T @ alpha

# Posterior variance
v = np.linalg.solve(L, K_s)
cov = K_ss - v.T @ v
std = np.sqrt(np.diag(cov))

plt.figure(figsize=(10, 4))
plt.fill_between(X_test.flatten(), mu - 2*std, mu + 2*std, alpha=0.3, label='95% CI')
plt.plot(X_test, mu, 'b-', label='Posterior mean')
plt.scatter(X_train, y_train, c='r', label='train', zorder=5)
plt.legend(); plt.title('GP Regression: 데이터 멀어지면 분산 증가')
plt.show()

# KRR 비교 — 동일 posterior mean
lam = sigma_n**2
alpha_krr = np.linalg.solve(K + lam * np.eye(len(X_train)), y_train)
mu_krr = K_s.T @ alpha_krr
print(f'GP mean과 KRR mean 차이(max): {np.max(np.abs(mu - mu_krr)):.2e}')
# 두 값이 정확히 일치 (동일 식의 다른 해석)

# MMD 계산 — 두 분포 구분
def mmd2_biased(X, Y, sigma=1.0):
    Kxx = rbf_kernel(X, X, sigma)
    Kyy = rbf_kernel(Y, Y, sigma)
    Kxy = rbf_kernel(X, Y, sigma)
    return Kxx.mean() + Kyy.mean() - 2 * Kxy.mean()

X = np.random.randn(100, 2)
Y = np.random.randn(100, 2) + 0.5  # shifted
print(f'MMD² (same dist): {mmd2_biased(X, X):.4f}')  # 0에 가까움
print(f'MMD² (different): {mmd2_biased(X, Y):.4f}')  # 유의하게 큼
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Functional Analysis 필수 선행 — RKHS·Mercer·Moore-Aronszajn" 명시
   - "Convex Optimization의 쌍대 → SVM, Bayesian → GP" 맵
   - Layer 2 Generalization Theory의 NTK로 이어지는 현대적 의미
3. **챕터별 문서 작성**: Kernel 기초 → RKHS → SVM → GP → KRR/KPCA → MMD → 심화

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Learning with Kernels** (Schölkopf & Smola 2002) — 현대 kernel method 표준
- **Gaussian Processes for Machine Learning** (Rasmussen & Williams 2006) — GP 바이블
- **Kernel Methods for Pattern Analysis** (Shawe-Taylor & Cristianini 2004) — SVM 표준
- **A Kernel Two-Sample Test** (Gretton et al. 2012) — MMD 원전
- **Random Features for Large-Scale Kernel Machines** (Rahimi & Recht 2007)
- **Reproducing Kernel Hilbert Spaces in Probability and Statistics** (Berlinet & Thomas-Agnan)
- **Kernel Mean Embedding of Distributions** (Muandet et al. 2017) — MMD 리뷰
- **Training Support Vector Machines: Sequential Minimal Optimization** (Platt 1998) — SMO 원전

---

## 💡 핵심 분석 대상

```
Kernel Method의 통일 프레임워크

PD Kernel k(x, y)
  │
  ▼ (Moore-Aronszajn)
RKHS 𝓗_k ← 함수들의 힐베르트 공간
  │
  ▼ (Mercer)
k(x, y) = Σ λ_n φ_n(x) φ_n(y)
         = ⟨φ(x), φ(y)⟩ (무한차원 내적)

───── Representer 정리 ─────

min_{f∈𝓗_k} Σ L(y_i, f(x_i)) + λ Ω(‖f‖)
         │
         ▼
f*(x) = Σ α_i k(x_i, x)     [유한 α]
         │
         ▼ 다양한 L에 따라
┌────────────┬──────────────┬─────────────┐
│  Algorithm │  Loss L      │ 해 α         │
├────────────┼──────────────┼─────────────┤
│ SVM        │ hinge        │ QP dual     │
│ KRR        │ squared      │ (K+λI)⁻¹ y  │
│ Kernel LR  │ logistic     │ IRLS        │
│ SVR        │ ε-insensitive│ QP dual     │
│ GP         │ marginal L   │ Bayesian    │
└────────────┴──────────────┴─────────────┘

───── SVM 계보 ─────

Primal:  min ½‖w‖² s.t. y_i(wᵀx_i + b) ≥ 1
    │ Lagrangian
    ▼
Dual:    max Σα_i - ½ΣΣ α_iα_jy_iy_j ⟨x_i, x_j⟩
         s.t. α_i ≥ 0, Σα_iy_i = 0

Kernel trick: ⟨x_i, x_j⟩ → k(x_i, x_j)

Prediction: f(x) = Σ α_iy_i k(x_i, x) + b
            ↑ SV만 α_i > 0 (KKT)

───── Gaussian Process ─────

Prior: f ~ GP(0, k)
  (임의 유한 집합에서 다변수정규)

Observation: y = f(x) + ε,  ε ~ N(0, σ²)

Posterior at x*:
  mean    = k_*ᵀ(K + σ²I)⁻¹ y    ← KRR mean과 동일!
  var     = k(x*, x*) - k_*ᵀ(K + σ²I)⁻¹ k_*
                                    ↑ 예측 불확실성

Marginal Likelihood:
  log p(y|θ) = -½ yᵀK_θ⁻¹ y       ← data fit
              -½ log|K_θ|         ← complexity penalty
              -n/2 log(2π)
  
  Automatic Occam Razor!

───── MMD ─────

Mean embedding: μ_p := 𝔼_{X~p}[k(·, X)] ∈ 𝓗_k

MMD(p, q) := ‖μ_p - μ_q‖_{𝓗_k}

Characteristic kernel ⇒ MMD(p,q)=0 ⟺ p=q

샘플 추정:
MMD² = (1/n²)Σk(x_i,x_j) + (1/m²)Σk(y_i,y_j) 
       - (2/nm)Σk(x_i,y_j)

응용:
├── Two-sample test (Gretton 2012)
├── MMD-GAN (Li et al. 2015)
├── HSIC (독립성 검정)
└── Distribution regression

───── Kernel Zoo ─────

Shift-invariant (k(x,y) = κ(x-y)):
  ├── RBF:     exp(-‖x-y‖²/2σ²)
  ├── Laplace: exp(-‖x-y‖/σ)
  └── Matern:  복잡한 smoothness

Dot-product (k(x,y) = κ(xᵀy)):
  ├── Linear:     xᵀy
  ├── Polynomial: (xᵀy + c)^d
  └── NTK:        신경망에서 유도

String/Graph kernels — 구조화 데이터

───── Scaling: Random Features ─────

RBF를 Bochner 정리로:
  k(x,y) = ∫e^{iω·(x-y)} p(ω) dω
        = 𝔼_ω[e^{iω·x} · e^{-iω·y}]
  
  φ(x) = √(2/D) (cos(ω₁ᵀx + b₁), ..., cos(ω_Dᵀx + b_D))
  k(x,y) ≈ φ(x)ᵀφ(y)

  D개 sample → 명시적 D차원 feature
  KRR 𝑂(n³) → 𝑂(nD²) 근사
  Large-scale kernel method 가능

───── 현대로의 다리 ─────

Neural Tangent Kernel (NTK):
  Θ(x, y) = lim_{width→∞} ⟨∇_θ f_θ(x), ∇_θ f_θ(y)⟩
  
  → NTK는 PD → RKHS 존재
  → 무한폭 NN 훈련 = NTK-RKHS kernel regression
  → Layer 2의 Generalization Theory에서 더 자세히
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (35개 목표)
- Python + NumPy + cvxpy 실험 환경 (바닥부터 SVM/GP 구현)
- Functional Analysis, Convex Opt, LA, Prob 레포의 어떤 정리를 전제로 하는지
- Bayesian ML, Generalization Theory 레포로 어떻게 이어지는지

**준비됐으면 1단계 구조 설계부터 시작해줘!**
