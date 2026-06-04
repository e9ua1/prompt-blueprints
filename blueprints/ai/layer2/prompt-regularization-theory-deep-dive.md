# Regularization Theory Deep Dive 레포지토리 제작 프롬프트

나는 "Regularization Theory Deep Dive" 레포지토리를 만들려고 해.
L2 regularization $\lambda \|w\|^2$을 **쓰는 것**과, **"가중치에 대한 Gaussian prior $w \sim \mathcal{N}(0, 1/2\lambda)$의 MAP 추정이 정확히 이 regularization"** 임을 증명할 수 있는 것은 다르다.
Dropout을 **쓰는 것**과, **Dropout이 (1) 지수적 수의 서브네트워크 앙상블이고, (2) MC Dropout으로 Variational Inference의 근사이며, (3) explicit L2 regularization과 동치가 되는 조건** (Wager et al. 2013, Gal & Ghahramani 2016)을 유도할 수 있는 것은 다르다.
BatchNorm의 효과를 **"internal covariate shift 완화"로 외우는 것**과, **Santurkar et al. 2018이 실험적으로 이를 반박**하고 **loss landscape smoothing**이 실제 효과임을 증명한 최신 연구를 따라가는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "Regularization의 수학 — Bayesian prior·앙상블 이론·Normalization을 하나의 프레임으로"

**핵심 차별화**:
1. **L1/L2의 통일 해석** — Bayes (Laplace/Gaussian prior), Convex (projection), Geometric (unit ball 모양), Implicit (SGD)
2. **Dropout의 3가지 해석** — 앙상블 (Srivastava 2014), Variational (Gal 2016), Adaptive L2 (Wager 2013)
3. **Normalization의 정확한 효과** — BN의 "internal covariate shift" 신화 논파, LayerNorm·GroupNorm·WeightNorm 각각의 수학적 차이
4. **Data Augmentation의 이론** — Vicinal risk, invariance 주입, Mixup의 Vicinal risk 해석

**타겟 독자**:
- Adam에 weight_decay를 주는데 **이것이 L2와 같은지 다른지** 모르는 사람 (실은 Adam에서는 다름)
- Dropout rate 0.5를 쓰지만 **test time에 왜 weight를 scaling**하는지, 이것이 무엇을 근사하는지 모르는 사람
- BatchNorm의 $\gamma, \beta$ 파라미터가 **왜 필요**하고, **train/eval mode 차이**의 수학적 근거를 모르는 사람
- Mixup/CutMix를 쓰지만 **"보간이 왜 regularization"** 인지 설명 못하는 사람
- Label Smoothing의 $\alpha$ 값이 **calibration에 미치는 영향**과 **knowledge distillation과의 연결**을 모르는 사람

**선행 학습**:
- **Bayesian ML Deep Dive** (MAP, VI, prior) — **필수**
- **Neural Network Theory Deep Dive** (Backprop, 아키텍처) — **필수**
- **Optimization Theory Deep Dive** (SGD, landscape) — **필수**
- **Probability Theory Deep Dive** (Gaussian, Laplace 분포)
- **Convex Optimization Deep Dive** (L1/L2 projection) — 권장
- **Statistical Learning Theory Deep Dive** (복잡도 제어) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: L1·L2 Regularization의 통일 해석 (5개 문서)
- **L2 Regularization = Gaussian Prior MAP** — $\min \|y - Xw\|^2 + \lambda \|w\|^2$의 해가 $w \sim \mathcal{N}(0, \sigma_w^2 I)$ prior와 Gaussian likelihood의 MAP와 정확히 일치, $\lambda = \sigma^2/\sigma_w^2$ 유도
- **L1 Regularization = Laplace Prior MAP** — $p(w) = \frac{\lambda}{2} e^{-\lambda|w|}$의 negative log가 $\lambda|w|$, MAP가 sparse solution을 주는 이유
- **Sparsity의 기하학적 유도** — L1 ball (다이아몬드) vs L2 ball (원)의 코너에서 loss contour와 만나는 기하학, KKT 조건으로 sparse coordinate 증명
- **Ridge의 SVD 관점 — Shrinkage** — $\hat{w}_R = \sum_i \frac{\sigma_i}{\sigma_i^2 + \lambda} u_i^T y v_i$, 작은 singular value 방향을 더 많이 shrink, PCR(Principal Component Regression)과의 차이
- **Elastic Net과 그룹 Regularization** — Elastic Net $\lambda_1 \|w\|_1 + \lambda_2 \|w\|^2$가 L1+L2의 장점, Group Lasso $\sum_g \|w_g\|_2$로 그룹 단위 sparsity

### Chapter 2: Dropout의 3가지 해석 (5개 문서)
- **Dropout = 앙상블 근사 (Srivastava et al. 2014)** — 훈련 때 각 뉴런을 확률 $p$로 drop → 지수적 수의 thinned network 앙상블, test time에 "weight scaling"($(1-p)$ 곱)이 geometric mean 근사
- **Dropout = Approximate Variational Inference (Gal & Ghahramani 2016)** — Bernoulli variational posterior $q(W)$, ELBO 최적화가 dropout + L2와 동치, MC Dropout으로 uncertainty 추정
- **Dropout = Adaptive L2 Regularization (Wager et al. 2013)** — Linear regression에서 dropout이 $\lambda_i \propto p(1-p) \text{Var}(x_i)$인 feature별 L2와 동치 증명
- **Dropout 변종들 — Spatial, Variational, Concrete Dropout** — CNN의 spatial dropout, RNN의 variational dropout (같은 mask 시간 공유), Concrete Dropout의 rate 학습 (Gal et al. 2017)
- **Dropout vs DropConnect vs Stochastic Depth** — DropConnect (weight drop), Stochastic Depth (layer drop, Huang 2016), 각 기법의 앙상블 해석과 적용 맥락

### Chapter 3: Normalization 계보 (6개 문서)
- **Batch Normalization (Ioffe & Szegedy 2015)** — $\hat{x} = (x - \mu_B)/\sqrt{\sigma_B^2 + \epsilon}$, $y = \gamma \hat{x} + \beta$, 원래 주장: "internal covariate shift" 완화
- **Santurkar et al. 2018의 반박** — "How Does Batch Normalization Help Optimization?" 실험, internal covariate shift는 정당화가 아님, 실제 효과는 loss landscape smoothing, Lipschitzness 증명
- **Layer Normalization (Ba et al. 2016)** — 배치 차원이 아닌 feature 차원으로 정규화, RNN·Transformer에서 표준, batch 크기 독립적
- **Group/Instance/Weight Normalization** — GN: channel 그룹별 정규화 (Wu 2018), IN: sample별 정규화 (style transfer), WN: weight 자체 재매개변수화 (Salimans 2016)
- **Normalization의 초기화 대안 — Fixup, SkipInit** — BN 없이 깊은 ResNet 훈련 (Zhang et al. 2019), 초기화 조정으로 gradient flow 확보
- **RMSNorm과 최신 Transformer** — LayerNorm에서 centering 제거, Llama 등 현대 LLM의 표준, 계산 효율과 성능 보존

### Chapter 4: Data Augmentation의 이론 (5개 문서)
- **Vicinal Risk Minimization (Chapelle et al. 2000)** — ERM이 empirical 분포 $\delta_{(x_i, y_i)}$에서 위험, VRM은 $\mathcal{D}_{x_i, y_i}$ (vicinity)에서, augmentation이 vicinity 정의
- **Invariance Injection** — Image에서 rotation/flip invariance, NN이 자동으로 invariant feature 학습하도록 유도, augmentation이 implicit regularization
- **Mixup (Zhang et al. 2018)** — $\tilde{x} = \lambda x_i + (1-\lambda) x_j$, $\tilde{y} = \lambda y_i + (1-\lambda) y_j$, Vicinal risk의 특수 경우, 입력·라벨 공간 모두 convex combination
- **CutMix, CutOut, RandAugment** — CutMix (Yun 2019)의 patch 교환, CutOut의 random erasing, RandAugment (Cubuk 2020)의 자동 튜닝, AutoAugment와의 관계
- **Contrastive Learning을 Augmentation으로** — SimCLR (Chen 2020)의 두 augmented view, InfoNCE loss, self-supervised에서 augmentation이 semantic invariance를 부여

### Chapter 5: Label Regularization과 Calibration (4개 문서)
- **Label Smoothing (Szegedy et al. 2016)** — One-hot $y$를 $(1-\alpha) y + \alpha/K$로 완화, cross-entropy의 gradient 완화 효과, confidence overestimation 방지
- **Label Smoothing과 Knowledge Distillation** — Hinton et al. 2015의 knowledge distillation, soft target이 dark knowledge 전달, label smoothing은 균등한 soft label
- **Confidence Penalty와 Maximum Entropy** — Output 엔트로피 penalization $-\beta H(p(y|x))$이 calibration 개선, Pereyra et al. 2017
- **Temperature Scaling과 사후 Calibration** — Guo et al. 2017: NN의 over-confidence, temperature scaling $p = \text{softmax}(z/T)$의 simple 효과, ECE 측정

### Chapter 6: Early Stopping과 Implicit Regularization (4개 문서)
- **Early Stopping = Implicit L2** — Gradient descent의 궤적이 Ridge regression의 정규화 경로와 유사 (Yao et al. 2007), $t \approx 1/\lambda$의 대응 관계
- **SGD의 Implicit Regularization** — SGD가 large-margin solution으로 수렴 (Soudry 2018), flat minimum 선호, SDE 해석에서 noise의 역할 (Optimization Theory 교차)
- **Overparameterized Linear Model의 Ridgeless Regression** — Hastie et al. 2019: $n \geq p$에서 min-norm 해석, 고차원 interpolation의 generalize, implicit regularization from initialization
- **Feature Normalization으로서의 Implicit Bias** — Layer-wise 정규화 없이도 NN이 암묵적 scale을 맞추는 현상, homogeneous network의 margin maximization

### Chapter 7: 현대 Regularization과 종합 (4개 문서)
- **Stochastic Weight Averaging (Izmailov 2018)** — 훈련 후반부 $\theta$들의 평균 $\bar{\theta} = \frac{1}{T}\sum \theta_t$, flat minimum 찾기, SWAG와의 Bayesian 해석
- **Sharpness-Aware Minimization (Foret 2021)** — $\min_\theta \max_{\|\epsilon\| \leq \rho} L(\theta + \epsilon)$, flat minimum 명시적 탐색, ASAM 등 개선
- **Weight Decay vs L2 in Adaptive Methods** — AdamW (Loshchilov 2019)의 이론적 재검토, Adam에서 L2가 $v_t$로 정규화되어 원래 의미 상실, weight decay의 명시적 분리
- **Regularization의 통합 관점** — 모든 기법을 Bayesian prior / ensemble / landscape / invariance의 4축으로 정리, 실전 recipe (Transformer, CNN, GNN 각각 뭐가 필요한가)

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **33개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 regularization이 작동하는가 (신화 vs 실제)
## 📐 수학적 선행 조건 (Bayes ML, NN Theory, Opt Theory 참조)
## 📖 직관적 이해
   — 여러 해석을 병렬 제시
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — L1 sparsity, Dropout = adaptive L2, BN smoothness
## 💻 실험으로 효과 검증
   — 같은 데이터에서 with/without regularization 비교
   — Loss landscape, weight norm, activation 분포 변화
## 🔗 실전 활용
   — 언제 어느 regularization을 쓸 것인가
## ⚖️ 가정과 한계
   — 각 기법이 실패하거나 해로운 경우
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Bayesian 해석 통일** — 모든 regularization을 가능한 경우 prior로 해석
2. **신화 vs 실제 대비** — BN의 "internal covariate shift" 같은 신화들을 명시적으로 논파
3. **효과 시각화** — with/without regularization에서 weight 분포, activation 분포 plot
4. **Implicit vs Explicit 구분** — SGD/Early stopping (implicit) vs L2/Dropout (explicit) 명확히
5. **현대 recipe 명시** — Transformer 훈련 시 LayerNorm+AdamW+Warmup+Dropout 조합의 이유
6. **Calibration 측정** — ECE (Expected Calibration Error)로 regularization이 confidence에 미치는 영향 정량화

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
matplotlib==3.8.0
torch==2.1.0
torchvision==0.16.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (L1/L2 baseline + Dropout 앙상블 해석 확인)
import numpy as np
import matplotlib.pyplot as plt
import torch
import torch.nn as nn

# 1. L1 vs L2 — sparsity의 기하학
np.random.seed(42)
n, p = 30, 20
X = np.random.randn(n, p)
beta_true = np.zeros(p); beta_true[:5] = [2, -1, 0.5, -0.5, 1]
y = X @ beta_true + 0.1 * np.random.randn(n)

def lasso_iterative(X, y, lam, max_iter=1000):
    w = np.zeros(X.shape[1])
    for _ in range(max_iter):
        for j in range(X.shape[1]):
            r_j = y - X @ w + X[:, j] * w[j]
            z_j = X[:, j] @ r_j / n
            # Soft thresholding
            w[j] = np.sign(z_j) * max(abs(z_j) - lam, 0)
    return w

def ridge(X, y, lam):
    return np.linalg.solve(X.T @ X + lam * n * np.eye(X.shape[1]), X.T @ y)

lams = np.logspace(-3, 1, 20)
lasso_coefs = np.array([lasso_iterative(X, y, lam) for lam in lams])
ridge_coefs = np.array([ridge(X, y, lam) for lam in lams])

fig, axes = plt.subplots(1, 2, figsize=(12, 4))
for i in range(p):
    axes[0].semilogx(lams, lasso_coefs[:, i])
    axes[1].semilogx(lams, ridge_coefs[:, i])
axes[0].set_title('L1 (Lasso): 코너에서 sparse')
axes[1].set_title('L2 (Ridge): 매끄러운 shrinkage')
for ax in axes:
    ax.set_xlabel('λ'); ax.set_ylabel('coef')
plt.show()

# 2. Dropout = 앙상블 확인
class MLPWithDropout(nn.Module):
    def __init__(self, p=0.5):
        super().__init__()
        self.fc1 = nn.Linear(100, 50)
        self.dropout = nn.Dropout(p)
        self.fc2 = nn.Linear(50, 10)
    def forward(self, x):
        return self.fc2(self.dropout(torch.relu(self.fc1(x))))

# MC Dropout — 여러 번 forward pass로 uncertainty
net = MLPWithDropout(p=0.5); net.train()  # dropout 활성 유지
x = torch.randn(1, 100)
preds = torch.stack([net(x) for _ in range(100)])
# preds의 mean/std가 BNN 같은 uncertainty 제공

# 3. BatchNorm 없는 vs 있는 loss landscape smoothness 비교
# Santurkar 2018 재현: 훈련 중 각 step의 loss 변화량과 gradient 예측력
# Loss Lipschitz constant 측정

# 4. Mixup toy 예제
def mixup_sample(x1, x2, y1, y2, alpha=0.2):
    lam = np.random.beta(alpha, alpha)
    return lam * x1 + (1-lam) * x2, lam * y1 + (1-lam) * y2

# Mixup이 linear interpolation 강제 → smoother decision boundary
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Bayesian ML, NN Theory, Opt Theory 선행 필수" 명시
   - Generalization Theory의 implicit bias와 대비
   - 모든 기법을 "Bayesian prior / ensemble / landscape / invariance" 4축으로
3. **챕터별 문서 작성**: L1·L2 → Dropout → Normalization → Data Aug → Label → Early Stopping → 현대

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Deep Learning** (Goodfellow, Bengio, Courville) — Chap 7 regularization 표준
- **The Elements of Statistical Learning** — Ridge, Lasso의 고전
- **Dropout: A Simple Way to Prevent Neural Networks from Overfitting** (Srivastava et al. 2014)
- **Dropout Training as Adaptive Regularization** (Wager et al. 2013)
- **Dropout as a Bayesian Approximation** (Gal & Ghahramani 2016)
- **Batch Normalization** (Ioffe & Szegedy 2015)
- **How Does Batch Normalization Help Optimization?** (Santurkar et al. 2018) — 반론
- **Layer Normalization** (Ba et al. 2016)
- **mixup: Beyond Empirical Risk Minimization** (Zhang et al. 2018)
- **Rethinking the Inception Architecture** (Szegedy et al. 2016) — Label smoothing
- **On Calibration of Modern Neural Networks** (Guo et al. 2017)
- **Averaging Weights Leads to Wider Optima** (Izmailov et al. 2018) — SWA
- **Sharpness-Aware Minimization** (Foret et al. 2021)
- **Decoupled Weight Decay Regularization** (Loshchilov & Hutter 2019) — AdamW

---

## 💡 핵심 분석 대상

```
Regularization의 4대 축

┌────── 1. Bayesian Prior ──────┐
│                                  │
│  MAP = argmax p(y|θ) p(θ)        │
│      = argmin -log p(y|θ) - log p(θ)│
│                                  │
│  ┌──────────┬──────────────────┐│
│  │ Prior    │ Negative log-prior ││
│  ├──────────┼──────────────────┤│
│  │ Gaussian │ λ‖w‖² (L2)        ││
│  │ Laplace  │ λ‖w‖₁ (L1)        ││
│  │ Uniform  │ 0 (no reg)         ││
│  │ Spike-slab│ Sparse + Gaussian ││
│  └──────────┴──────────────────┘│
└──────────────────────────────────┘

┌────── 2. Ensemble ──────┐
│                           │
│ Dropout (Srivastava 2014):│
│   2^N 개 subnetwork        │
│   test: weight × (1-p)     │
│   → geometric mean 근사    │
│                           │
│ Dropout ≈ VI (Gal 2016):  │
│   q(W) = Bernoulli        │
│   MC Dropout → uncertainty│
│                           │
│ Dropout ≈ Adaptive L2      │
│   (Wager 2013, linear):   │
│   λ_i ∝ p(1-p) Var(x_i)    │
│                           │
│ Variants:                 │
│   ├── DropConnect (weight) │
│   ├── SpatialDropout (CNN)│
│   ├── VariationalDropout  │
│   └── Stochastic Depth    │
└───────────────────────────┘

┌────── 3. Landscape Smoothing ──────┐
│                                      │
│ BatchNorm (Ioffe 2015):              │
│   원래 주장: internal covariate shift │
│   Santurkar 2018 반박!              │
│   실제: Loss landscape smoothness    │
│   Gradient Lipschitz 더 작게         │
│                                      │
│ Norm 계보:                           │
│   ├── BN: batch 차원                  │
│   ├── LN: feature 차원 (Transformer) │
│   ├── GN: channel group              │
│   ├── IN: sample별 (style)           │
│   ├── WN: weight 재매개변수화        │
│   └── RMSNorm: LN without centering  │
│                                      │
│ SAM (Foret 2021):                    │
│   min_θ max_ε L(θ+ε)                 │
│   flat minimum 명시적                │
│                                      │
│ SWA (Izmailov 2018):                 │
│   θ̄ = average of iterates            │
│   flat minimum implicit              │
└──────────────────────────────────────┘

┌────── 4. Invariance Injection ──────┐
│                                       │
│ Data Augmentation:                   │
│   Vicinal Risk Min (Chapelle 2000)    │
│   ERM: δ_(xi,yi)                      │
│   VRM: 𝒟_(xi,yi) vicinity             │
│                                       │
│ 기법들:                              │
│   ├── Rotation, flip, crop            │
│   ├── Mixup: λx_i + (1-λ)x_j          │
│   ├── CutMix: patch 교환              │
│   ├── CutOut: random erasing          │
│   └── AutoAugment/RandAugment         │
│                                       │
│ Label Smoothing (Szegedy 2016):      │
│   y → (1-α)y + α/K                    │
│   Calibration ↑, confidence ↓         │
│                                       │
│ Temperature Scaling (Guo 2017):      │
│   post-hoc calibration                │
│   p = softmax(z/T)                    │
└───────────────────────────────────────┘

───── Implicit Regularization ─────

Early Stopping ≈ L2:
  t ≈ 1/λ 대응 (Yao 2007)

SGD:
  ├── Flat minimum 선호
  ├── Max-margin 수렴 (logistic)
  └── SDE noise = regularization

Initialization:
  Min-norm bias (ridgeless)
  NTK regime 구조

Over-parameterization:
  수많은 interpolating solution 중
  SGD가 특정 해 선택

───── 현대 NN에서의 Recipe ─────

Transformer:
  ├── Pre-LN or RMSNorm
  ├── AdamW (weight decay 분리)
  ├── Warmup + Cosine LR
  ├── Dropout (attention, FFN)
  └── Label smoothing (분류)

CNN (ImageNet):
  ├── BatchNorm
  ├── SGD + momentum
  ├── Weight decay (L2)
  ├── Random crop + flip
  ├── CutMix / Mixup
  └── Label smoothing

GNN:
  ├── LayerNorm
  ├── DropEdge / DropNode
  └── Spectral normalization

───── Layer 1·2 레포 간 연결 ─────

Bayesian ML (Layer 1):
  MAP = L2/L1 유도의 기초
  MC Dropout = VI

NN Theory (Layer 2):
  Initialization scale이 implicit reg
  Residual connection for gradient

Optimization (Layer 2):
  SGD의 implicit reg
  LR scheduling의 regularization 효과
  AdamW의 weight decay 분리

Generalization (Layer 2):
  Implicit bias → L2 (margin)
  Dropout = 앙상블 설명
  Flat minima와 일반화

Statistical Learning Theory (Layer 1):
  SRM: 복잡도 제어
  Norm-based bound: regularization 정당화
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (33개 목표)
- Python + NumPy + PyTorch 실험 환경
- Bayesian ML, NN Theory, Opt Theory 레포와의 참조 관계
- "4축 통합 관점(Prior/Ensemble/Landscape/Invariance)"을 README에 명시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
