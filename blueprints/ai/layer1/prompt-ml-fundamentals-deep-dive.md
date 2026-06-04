# ML Fundamentals Deep Dive 레포지토리 제작 프롬프트

나는 "ML Fundamentals Deep Dive" 레포지토리를 만들려고 해.
`sklearn.linear_model.LinearRegression().fit(X, y)`를 **호출하는 것**과, **Normal Equation $\hat{\beta} = (X^T X)^{-1} X^T y$를 MLE·기하학적 투영·Moore-Penrose pseudoinverse 세 관점에서 유도**할 수 있는 것은 다르다.
`RandomForestClassifier`를 **쓰는 것**과, **Random Forest가 tree 개수 $B \to \infty$에서 수렴하는 것이 "동일분포 상관된 확률변수의 평균"의 수렴 공식**에서 나오고, **bias-variance decomposition으로 분산이 $\rho \sigma^2 + \frac{1-\rho}{B} \sigma^2$로 감소**함을 증명할 수 있는 것은 다르다.
`XGBoost`를 **쓰는 것**과, **Gradient Boosting이 함수공간에서의 경사하강법**이고, **AdaBoost의 지수손실 최소화가 Gradient Boosting의 특수 경우**임을 유도할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "고전 ML을 수학적으로 재구성 — 선형 회귀·트리·앙상블이 왜 동작하는지 증명 가능하게"

**핵심 차별화**:
1. **선형 회귀를 3가지 관점에서 완전 유도** — MLE, 기하학적 수직투영(Functional Analysis의 Hilbert 투영), Moore-Penrose pseudoinverse
2. **Ridge/Lasso의 베이지안·기하학·최적화 삼위일체** — MAP 추정(Gaussian/Laplace prior), 볼록 제약 투영, 쌍대성
3. **결정트리 분할 기준을 정보이론으로** — 정보이득은 상호정보량, Gini는 분산 감소의 이산 버전
4. **앙상블 3대 원리 완전 증명** — Bagging(분산 감소), Boosting(함수공간 gradient descent), Random Forest(상관관계 감소)

**타겟 독자**:
- Linear Regression을 쓰지만 $(X^TX)^{-1}$이 왜 특이일 때 pseudoinverse로 대체되는지 모르는 사람
- Logistic Regression의 sigmoid가 **왜 지수분포족의 canonical link function**인지 모르는 사람
- Random Forest의 성능이 tree 개수에 따라 **왜 수렴하고 언제 포화**하는지 증명 못하는 사람
- AdaBoost를 쓰지만 **가중치 업데이트 공식 $w_i \leftarrow w_i \exp(\alpha \mathbf{1}[y_i \neq h(x_i)])$이 왜 지수손실의 coordinate descent**인지 모르는 사람
- XGBoost를 쓰지만 **2차 Taylor 근사가 왜 정확히 Newton-Raphson의 tree 버전**인지 설명 못하는 사람

**선행 학습**:
- **Linear Algebra Deep Dive** (Pseudoinverse, 수직투영, SVD) — **필수**
- **Probability Theory Deep Dive** (결합분포, 조건부 기댓값) — **필수**
- **Mathematical Statistics Deep Dive** (MLE, Fisher 정보, Exponential Family) — **필수**
- **Calculus & Optimization Deep Dive** (Gradient Descent, Newton's method)
- **Convex Optimization Deep Dive** (Lagrangian, KKT) — Ridge/Lasso에 유용
- **Information Theory Deep Dive** (상호정보량, 엔트로피) — 결정트리에 유용

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 선형 회귀 — 3가지 관점의 유도 (6개 문서)
- **MLE 관점에서의 선형 회귀** — 가정 $y = X\beta + \epsilon, \epsilon \sim \mathcal{N}(0, \sigma^2 I)$에서 log-likelihood 최대화가 $\min \|y - X\beta\|^2$로 귀결, Normal Equation $\hat{\beta} = (X^TX)^{-1}X^Ty$ 유도
- **기하학적 관점 — 수직투영** — $\hat{y} = X\hat{\beta}$는 $y$의 $\text{col}(X)$로의 수직투영, $\hat{y} - y \perp \text{col}(X)$에서 정규방정식 기하학적 유도, Hilbert 공간의 수직분해와 연결
- **Moore-Penrose Pseudoinverse와 Rank-deficient 경우** — $X^TX$가 특이할 때 $X^+ = \lim_{\lambda \to 0^+} (X^TX + \lambda I)^{-1} X^T$, SVD를 이용한 $X^+$ 계산, 최소노름 해의 유일성
- **Ridge Regression의 3가지 해석** — (1) $\min \|y - X\beta\|^2 + \lambda \|\beta\|^2$의 해 $\hat{\beta} = (X^TX + \lambda I)^{-1} X^T y$, (2) $\beta \sim \mathcal{N}(0, \tau^2 I)$ prior의 MAP, (3) 제약 최적화 $\|\beta\| \leq c$의 쌍대
- **Lasso와 Sparsity** — L1 제약의 기하학(다이아몬드 모양), KKT 조건으로 sparsity 유도, Gaussian noise + Laplace prior의 MAP, Coordinate Descent 알고리즘
- **Bias-Variance Decomposition** — $\mathbb{E}[(y - \hat{f}(x))^2] = \text{Bias}^2 + \text{Var} + \sigma^2$ 증명, Ridge가 variance 감소를 위해 bias 증가를 허용, 정규화 경로

### Chapter 2: Logistic Regression과 일반화선형모형(GLM) (5개 문서)
- **Logistic Regression의 MLE** — Bernoulli likelihood $\prod p_i^{y_i}(1-p_i)^{1-y_i}$, log-odds $\log \frac{p}{1-p} = \beta^T x$, log-likelihood의 볼록성(concavity) 증명
- **IRLS (Iteratively Reweighted Least Squares)** — Newton-Raphson 업데이트가 가중 최소제곱 문제 해결과 동치 증명, Hessian $X^T W X$ 유도, 수치적 구현
- **Exponential Family와 Canonical Link** — GLM의 세 구성요소(분포, 선형예측자, link function), Canonical link이 왜 MLE를 단순화하는지(Fisher scoring = IRLS), Logit/Probit/Log-log 비교
- **Multinomial/Softmax Regression** — One-vs-Rest vs 직접 multinomial, softmax의 identifiability 문제와 reference class 고정, Cross-entropy loss가 categorical MLE
- **분리 문제(Separation Problem)와 Firth Correction** — 완전분리 시 MLE 발산, Firth의 penalized likelihood로 유한해 보장, Bayesian 관점

### Chapter 3: 결정트리 (Decision Tree) (5개 문서)
- **결정트리의 분할 기준 — 정보이득** — 엔트로피 감소 $IG(S, A) = H(S) - \sum_v \frac{|S_v|}{|S|} H(S_v)$가 상호정보량 $I(Y; A)$와 동치, ID3 알고리즘 유도
- **Gini Impurity vs Entropy** — Gini $1 - \sum p_k^2$가 분산의 이산 버전, CART에서 Gini 선호 이유, 두 기준의 Taylor 전개로 근접성 증명
- **회귀 트리와 MSE 분할** — 각 leaf의 예측값이 leaf 내 평균일 때 MSE 최소화, CART의 탐욕 분할 알고리즘
- **가지치기(Pruning)와 Cost-Complexity** — 복잡도 매개변수 $\alpha$로 $R_\alpha(T) = R(T) + \alpha|T|$ 최소화, Weakest link pruning, Cross-validation으로 $\alpha$ 선택
- **결정트리의 한계 — 불안정성과 축정렬 편향** — 훈련 샘플 작은 변화가 완전히 다른 트리 생성, 축-정렬 분할(oblique tree로 해결), high-variance estimator(앙상블의 동기)

### Chapter 4: Bagging과 Random Forest (5개 문서)
- **Bootstrap과 OOB Error** — 부트스트랩 샘플링이 $1 - (1-1/n)^n \to 1 - 1/e \approx 63.2\%$의 데이터를 포함, Out-of-bag 샘플로 validation 대체
- **Bagging의 분산 감소 메커니즘** — $B$개 모델의 평균은 $\text{Var} = \frac{1}{B}\sigma^2$로 감소 (독립 가정), 상관된 경우 $\rho \sigma^2 + \frac{1-\rho}{B} \sigma^2$ 유도
- **Random Forest의 추가 무작위성** — 각 split에서 feature 부분집합 무작위 선택이 tree 간 상관관계 $\rho$ 감소 → 분산 추가 감소
- **Random Forest의 수렴성** — $B \to \infty$에서 RF predictor가 무한 앙상블로 수렴 증명 (Breiman 2001), generalization error가 tree 수에 단조감소
- **Feature Importance — Permutation과 MDI** — Mean Decrease in Impurity vs Permutation importance, 각각의 편향(MDI의 high-cardinality 편향), SHAP로 개선

### Chapter 5: Boosting (6개 문서)
- **AdaBoost의 수학적 유도** — 지수손실 $L(y, f) = e^{-yf}$ 가정, Forward Stagewise Additive Modeling으로 AdaBoost 업데이트 규칙 유도 (Friedman, Hastie, Tibshirani 2000)
- **AdaBoost의 이론적 성질** — 훈련 오차의 지수적 감소 $\prod \sqrt{4\epsilon_t(1-\epsilon_t)}$, margin distribution 관점, VC 경계와 일반화 오차
- **Gradient Boosting — 함수공간의 경사하강법** — 손실 $L(y, f)$의 음의 gradient $-\partial L / \partial f$에 tree를 fit, 함수공간 $\mathcal{F}$에서의 steepest descent, 학습률 $\eta$의 역할
- **XGBoost — 2차 Taylor 근사** — Loss $L(y, f_t) \approx L(y, f_{t-1}) + g_t \Delta f + \frac{1}{2} h_t \Delta f^2$, leaf value 최적해 $w^* = -G/(H + \lambda)$, Newton-Raphson의 tree 버전
- **LightGBM과 Histogram-based Splitting** — Gradient-based One-Side Sampling (GOSS), Exclusive Feature Bundling (EFB), leaf-wise vs level-wise tree 성장
- **Boosting의 과적합 저항성** — AdaBoost가 훈련오차 0 이후에도 테스트 오차 감소하는 현상, margin theory(Schapire et al. 1998)로 설명

### Chapter 6: 나이브 베이즈와 판별분석 (4개 문서)
- **Naive Bayes의 조건부 독립 가정** — $p(x | y) = \prod p(x_j | y)$, 이 "나이브한" 가정이 실전에서 왜 잘 동작하는지 — 분류 경계만 필요하므로 확률 추정이 부정확해도 됨
- **Gaussian Naive Bayes vs LDA vs QDA** — GNB는 대각 공분산 가정, LDA는 클래스 간 공분산 공유, QDA는 클래스별 공분산, 세 모델의 결정경계(선형/이차) 유도
- **LDA의 Fisher Discriminant 해석** — between-class / within-class variance 비 최대화, 일반화 고유값 문제 $S_B w = \lambda S_W w$ 해결, PCA와의 차이
- **Generative vs Discriminative Model** — Naive Bayes vs Logistic Regression의 asymptotic 비교 (Ng & Jordan 2001), 데이터 크기별 우열 교차

### Chapter 7: KNN·클러스터링·차원축소 (5개 문서)
- **K-Nearest Neighbors의 점근적 성질** — Cover-Hart 정리: 1-NN의 점근 오차 ≤ 2 × Bayes error, 차원의 저주 — 거리 집중(concentration of distances), curse of dimensionality
- **K-Means와 EM의 관계** — K-Means를 GMM의 hard-assignment 버전으로 유도, EM의 일반적 정식화, K-Means++ 초기화와 경쟁비(competitive ratio)
- **Hierarchical Clustering** — Agglomerative(bottom-up)와 Divisive(top-down), linkage 기준 (single, complete, average, Ward), ultrametric과 덴드로그램
- **DBSCAN과 밀도 기반 클러스터링** — 핵심점·경계점·노이즈 정의, 임의 모양 클러스터 탐지, $\epsilon$과 MinPts 매개변수의 해석
- **PCA·t-SNE·UMAP 비교** — PCA(선형, 분산 최대), t-SNE(KL(p||q) 최소화, local 구조), UMAP(topological, manifold 가정), 각 방법의 수학적 기초

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **36개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 방법이 ML에서 중요한가
## 📐 수학적 선행 조건 (LA, Stats, Calc 레포 참조)
## 📖 직관적 이해
   — 기하학·확률·알고리즘 세 가지 비유
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Normal Equation 3 방식, AdaBoost 유도, Cover-Hart 등
## 💻 NumPy 구현 검증
   — sklearn 없이 바닥부터 구현, sklearn과 결과 일치 확인
## 🔗 실전 연결
   — 언제 이 방법을 선택하는가, hyperparameter 해석
## ⚖️ 가정과 한계
   — 선형·IID·Gaussian noise·조건부 독립이 깨지면?
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **모든 알고리즘을 "바닥부터" NumPy로 구현** — sklearn은 검증용으로만
2. **3가지 관점 동시 제시** — 확률(MLE/MAP), 기하(투영/경계), 최적화(목적함수)
3. **Bias-Variance plot 필수** — 모든 장에서 데이터 크기·모델 복잡도별 학습곡선 시각화
4. **수학적 유도를 "한 줄씩"** — $\partial L / \partial \beta = 0 \Rightarrow \ldots$ 단계를 빼먹지 않기
5. **sklearn 내부 코드 교차 참조** — 중요 알고리즘은 sklearn 소스 링크와 비교
6. **실전 데이터셋** — Boston Housing (regression), Iris (classification), California Housing 등 표준

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
matplotlib==3.8.0
scikit-learn==1.3.0     # 검증용
xgboost==2.0.0          # Boosting 비교
lightgbm==4.1.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Normal Equation 3 방식 구현 + 비교)
import numpy as np
from sklearn.linear_model import LinearRegression

np.random.seed(42)
X = np.random.randn(100, 5)
beta_true = np.array([1, -2, 0.5, 0, 3])
y = X @ beta_true + 0.1 * np.random.randn(100)

# 방식 1: Normal Equation (직접)
beta_ne = np.linalg.solve(X.T @ X, X.T @ y)

# 방식 2: QR Decomposition (수치적 안정)
Q, R = np.linalg.qr(X)
beta_qr = np.linalg.solve(R, Q.T @ y)

# 방식 3: SVD + Pseudoinverse
U, s, Vt = np.linalg.svd(X, full_matrices=False)
beta_svd = Vt.T @ (np.diag(1/s) @ (U.T @ y))

# sklearn과 비교
lr = LinearRegression(fit_intercept=False).fit(X, y)

print(f'Normal Equation: {beta_ne}')
print(f'QR:              {beta_qr}')
print(f'SVD:             {beta_svd}')
print(f'sklearn:         {lr.coef_}')
print(f'True:            {beta_true}')

# 수직투영 검증: (y - Xβ) ⊥ col(X)
residual = y - X @ beta_ne
print(f'X^T @ residual (should be ~0): {X.T @ residual}')
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Layer 0 전반(LA, Prob, Stats, Calc) 선행" 명시
   - "NN Theory(Layer 2)의 비교 기준" 강조 — 선형 회귀가 가장 간단한 신경망
   - sklearn 구현과 수학을 연결하는 실천적 접근
3. **챕터별 문서 작성**: 선형회귀 → Logistic/GLM → 결정트리 → Bagging/RF → Boosting → 생성모델 → KNN/클러스터링

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **The Elements of Statistical Learning** (Hastie, Tibshirani, Friedman) — ESL, 표준 바이블
- **Pattern Recognition and Machine Learning** (Bishop) — 베이지안 관점 표준
- **Machine Learning: A Probabilistic Perspective** (Murphy) — 확률적 관점 현대 표준
- **An Introduction to Statistical Learning** (James, Witten, Hastie, Tibshirani) — 쉬운 입문
- **The Nature of Statistical Learning Theory** (Vapnik) — SLT의 창시자
- **Random Forests** (Breiman 2001) — RF 원전 논문
- **A Short Introduction to Boosting** (Freund & Schapire 1999) — AdaBoost 원전
- **XGBoost: A Scalable Tree Boosting System** (Chen & Guestrin 2016)
- **On Discriminative vs Generative Classifiers** (Ng & Jordan 2001)

---

## 💡 핵심 분석 대상

```
ML의 4대 Paradigm (이 레포의 지도)
┌──────────────────────────────────────────────────┐
│  Parametric      │  Non-parametric              │
│ ┌──────────────┐ │ ┌──────────────┐             │
│ │Linear Reg    │ │ │KNN           │             │
│ │Logistic Reg  │ │ │Decision Tree │ Supervised  │
│ │Naive Bayes   │ │ │Random Forest │             │
│ │LDA/QDA       │ │ │Boosting      │             │
│ └──────────────┘ │ └──────────────┘             │
│ ┌──────────────┐ │ ┌──────────────┐             │
│ │GMM (EM)      │ │ │DBSCAN        │ Unsupervised│
│ │PCA           │ │ │t-SNE/UMAP    │             │
│ │              │ │ │Hierarchical  │             │
│ └──────────────┘ │ └──────────────┘             │
└──────────────────────────────────────────────────┘

===== 선형 회귀: 3가지 관점 =====

Normal Equation: β̂ = (XᵀX)⁻¹ Xᵀy
  ├── MLE 관점: y = Xβ + ε, ε ~ N(0, σ²I)
  │             log L 최대화 ⇒ ‖y - Xβ‖² 최소화
  │
  ├── 기하 관점: ŷ = Xβ̂는 y의 col(X)로의 수직투영
  │             Hᵀ = X(XᵀX)⁻¹Xᵀ는 idempotent projection
  │             ŷ = Hy, residual r = (I-H)y ⊥ col(X)
  │
  └── Pseudoinverse: β̂ = X⁺ y
                    X⁺ = V Σ⁺ Uᵀ (SVD)
                    min-norm least-squares 해

Ridge: β̂_R = (XᵀX + λI)⁻¹ Xᵀy
  ├── MAP 관점: β ~ N(0, τ²I) prior
  ├── 제약 관점: min ‖y-Xβ‖² s.t. ‖β‖ ≤ c
  └── SVD 관점: β̂_R = Σ (σᵢ / (σᵢ² + λ)) uᵢᵀy vᵢ
               작은 σᵢ 방향을 shrinkage

Lasso: sparsity ← L1 geometry (diamond)
  KKT ⇒ subdifferential, coordinate descent 자연스러움

===== Decision Tree의 분할 기준 =====

Entropy-based (ID3, C4.5):
  IG(S, A) = H(S) - Σ_v (|S_v|/|S|) H(S_v) = I(Y; A)
           = 상호정보량 (MI!)
  ↑ Information Theory 레포 참조

Gini (CART):
  G(S) = Σ p_k(1 - p_k) = 1 - Σ p_k²
  ↑ 분산의 이산화 — Entropy Taylor 전개 1차항

MSE (회귀 트리):
  L = Σ_v Σ_i∈S_v (y_i - ȳ_v)²
  leaf의 예측값 ȳ_v = mean(y in leaf)

===== Ensemble 3대 원리 =====

1. Bagging (Breiman 1996):
   B개 모델 f̂_b의 평균 f̂ = (1/B) Σ f̂_b
   
   Var(f̂) = ρσ² + (1-ρ)/B · σ²
              ↑            ↑
       상관관계가 하한   B↑로 감소
   
   ⇒ 상관관계 낮추면 분산 더 감소
   ⇒ Random Forest: 각 split에서 feature 부분집합 랜덤
       → ρ 감소 → Var 추가 감소

2. Boosting (함수공간 gradient descent):
   F_t(x) = F_{t-1}(x) + η · h_t(x)
   h_t = -∇_f L(y, F_{t-1})에 fit
   
   ├── AdaBoost: exponential loss L = e^{-yf}
   │             closed-form weight update
   ├── GBM: 임의 손실 + line search
   ├── XGBoost: 2차 Taylor
   │     w* = -G / (H + λ)  (Newton step per leaf)
   └── LightGBM: histogram + GOSS + EFB

3. Stacking:
   Meta-learner가 base learner의 예측을 input으로
   Cross-validation으로 meta training set 구성

===== GLM 체계도 =====

y | x ~ Exp Family (canonical form)
g(𝔼[y|x]) = βᵀx    (link function)

┌──────────┬──────────────┬────────────────┐
│ Distribution │ Canonical link │ Model       │
├──────────┼──────────────┼────────────────┤
│ Normal   │ identity      │ Linear Reg     │
│ Bernoulli│ logit         │ Logistic Reg   │
│ Binomial │ logit         │ Multinomial LR │
│ Poisson  │ log           │ Poisson Reg    │
│ Gamma    │ inverse       │ Gamma Reg      │
└──────────┴──────────────┴────────────────┘

MLE는 Fisher scoring = IRLS
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (36개 목표)
- Python + NumPy + sklearn(검증) 실험 환경
- Layer 0 레포(LA, Prob, Stats, Calc, Convex)의 어떤 정리를 전제로 사용하는지 명시
- Layer 2(NN Theory)로 어떻게 자연스럽게 이어지는지

**준비됐으면 1단계 구조 설계부터 시작해줘!**
