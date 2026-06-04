# Generalization Theory Deep Dive 레포지토리 제작 프롬프트

나는 "Generalization Theory Deep Dive" 레포지토리를 만들려고 해.
"큰 모델이 왜 잘 일반화하는가"를 **막연히 말하는 것**과, **VC 차원이 $O(W \log W)$인 ResNet50이 ImageNet에서 VC bound로 $\text{gap} \leq 10^{12}$ 을 주는 vacuous한 이유**와, **현대 이론이 어떻게 이를 극복**하는지 설명할 수 있는 것은 다르다.
Double Descent를 **현상으로 아는 것**과, **Belkin et al. (2019)의 실험을 Random Fourier Features로 재현**하고, **interpolation threshold에서 test error가 발산하는 이유**를 $U$-shape vs modern의 수학적 유도로 이해하는 것은 다르다.
Neural Tangent Kernel을 **이름으로 아는 것**과, **$\Theta(x, y) = \lim_{w \to \infty} \langle \nabla_\theta f(x; \theta), \nabla_\theta f(y; \theta) \rangle$이 초기화에서 일정 kernel로 수렴**하고, **훈련 역학이 kernel regression으로 환원**되는 증명(Jacot et al. 2018)을 따라갈 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "왜 딥러닝이 일반화하는가 — 고전이론의 실패와 현대적 설명"

**핵심 차별화**:
1. **고전 이론의 실패를 정량화** — Zhang et al. 2017 "Rethinking Generalization" 실험을 재현, random label에도 fit되는데 일반화하는 모순
2. **Double Descent의 수학적 유도** — Belkin et al., Nakkiran et al., Hastie et al. 2019의 random features 계산, test error가 $\lambda \to 0, p \to n$에서 발산
3. **Neural Tangent Kernel 완전 유도** — Jacot et al. 2018의 무한폭 극한, NTK의 kernel regression 등가성, "lazy" vs "feature learning" regime
4. **현대 현상들의 이해** — Grokking (Power et al. 2022), Lottery Ticket (Frankle & Carbin 2019), Scaling Laws (Kaplan 2020; Hoffmann 2022)

**타겟 독자**:
- 딥러닝을 쓰는데 "왜 이게 되는가"가 학문적으로 궁금한 연구자
- Uniform convergence가 딥러닝에서 vacuous한데 **대안이 무엇인지** 궁금한 사람
- Random Fourier Features로 Double Descent를 **직접 재현**하고 싶은 사람
- Grokking 현상에서 "일반화가 갑자기 일어나는" 메커니즘이 궁금한 사람
- Lottery Ticket을 들어봤는데 **원 논문의 증거**와 **후속 논쟁**(Liu et al. 2019, Frankle 2020)을 이해하고 싶은 사람

**선행 학습**:
- **Statistical Learning Theory Deep Dive** (VC, Rademacher, PAC) — **필수**
- **Neural Network Theory Deep Dive** (Backprop, 아키텍처) — **필수**
- **Functional Analysis Deep Dive** (RKHS, kernel) — **NTK에 필수**
- **Kernel Methods Deep Dive** (kernel regression) — **NTK에 필수**
- **Optimization Theory Deep Dive** (SGD 수렴, landscape) — 권장
- **Probability Theory Deep Dive** (random matrix 기초) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 고전 이론과 딥러닝의 불일치 (5개 문서)
- **고전 Bound의 Vacuous 문제** — ResNet50의 VC 차원 추정, ImageNet에서 고전 bound $\gg 1$, "의미없음(vacuous)"의 정량화, Zhang et al. 2017 재현
- **Random Label Experiment (Zhang et al. 2017)** — 무작위 라벨도 완전히 fit, 훈련 오차 0이지만 테스트 오차 chance, uniform convergence의 함의 ("학습 가능" 정의 재검토)
- **Rademacher Complexity도 Vacuous한 이유** — Bartlett 1998/2002 norm-based bound를 실제 훈련된 NN에 적용, 여전히 vacuous, Nagarajan & Kolter 2019의 uniform convergence 한계
- **Implicit Regularization의 증거** — 과매개변화된 NN이 interpolation 했음에도 일반화, SGD 자체가 regularization 역할, Soudry et al. 2018 gradient descent의 max-margin 수렴
- **일반화 퍼즐의 4가지 현상** — (1) over-parameterization에도 일반화 (2) Double Descent (3) Grokking (지연 일반화) (4) Scaling Laws — 고전이론이 설명 못함

### Chapter 2: Norm-based Generalization Bounds (5개 문서)
- **Margin Theory for Deep Networks (Bartlett et al. 2017)** — $O(\prod_l \|W_l\|_\sigma \cdot \sqrt{\text{something}} / \sqrt{n})$, spectral norm 기반 경계, distance from initialization 포함
- **PAC-Bayes for Neural Networks** — Prior와 posterior에 대한 KL로 경계, Dziugaite & Roy 2017의 non-vacuous bound (처음으로!), flat minima와 PAC-Bayes의 연결
- **Neyshabur의 Path-Norm** — Path-norm $\sum_{\text{paths}} \prod |w_e|$가 scale-invariant, capacity control로 사용, 경로 기반 측도의 이점
- **Compression-based Bounds (Arora et al. 2018)** — 훈련된 NN을 압축 가능하면 작은 effective complexity, pruning·quantization으로 증명, Lottery Ticket과의 연결
- **왜 Norm-based도 완전하지 않은가** — $\prod \|W\|$가 훈련 동안 증가, 이미 vacuous한 경우 많음, 설명력의 한계

### Chapter 3: Neural Tangent Kernel (6개 문서)
- **NTK의 정의와 유도** — $\Theta(x, y) = \langle \nabla_\theta f(x; \theta), \nabla_\theta f(y; \theta) \rangle$, 무한폭 극한에서 상수 kernel로 수렴 증명 (Jacot, Gabriel, Hongler 2018)
- **NTK Regime의 훈련 동역학** — $\theta_t$에서 $f(x; \theta_t)$가 초기값에 대해 선형, $f(x; \theta_t) - f(x; \theta_0) \approx \Theta(x, \cdot)(K + \lambda I)^{-1} y$, kernel regression과 동치
- **Neural Network Gaussian Process (NNGP)** — 무한폭 NN의 초기화에서 출력이 GP로 수렴 (Lee et al. 2018, Matthews et al. 2018), NTK와 NNGP의 관계
- **NTK의 재생커널 속성** — NTK가 PD → RKHS $\mathcal{H}_\Theta$ 존재, 무한폭 NN 훈련 = $\mathcal{H}_\Theta$에서의 kernel ridge regression, Functional Analysis 레포와의 교차
- **NTK의 한계 — Lazy vs Feature Learning** — NTK regime에서 feature가 변하지 않음 ("lazy"), Chizat et al. 2019, 실전 NN은 feature learning이 핵심, Mean-field regime으로의 전이
- **NTK 계산과 실증** — Random initialization에서 empirical NTK 계산, 유한 폭에서 수렴 속도, Neural Tangents 라이브러리 활용

### Chapter 4: Double Descent (5개 문서)
- **Classic U-shape vs Double Descent** — 고전: bias-variance trade-off로 U-shape, 현대: interpolation threshold 넘으면 다시 test error 감소, Belkin et al. 2019 "Reconciling Modern Machine Learning Practice"
- **Random Fourier Features로 Double Descent 재현** — Mei & Montanari 2019의 정확한 asymptotic, test error가 $p/n$의 함수로 분석, Random matrix 이론 (Marchenko-Pastur)
- **Neural Network에서의 Double Descent** — Nakkiran et al. 2019 "Deep Double Descent", epoch-wise, model-wise, sample-wise double descent 각각
- **Bias-Variance에서의 재해석** — Interpolation regime에서 variance가 감소하는 이유, effective degrees of freedom의 재정의, Hastie et al. 2019 ridgeless regression 분석
- **Regularization과 Double Descent** — 적절한 $\lambda$로 peak 제거, Implicit regularization (SGD, dropout)의 효과, 실전 deep learning에서 double descent를 보기 어려운 이유

### Chapter 5: Grokking과 Implicit Bias (4개 문서)
- **Grokking 현상 (Power et al. 2022)** — Modular arithmetic에서 훈련 오차 0 이후 오랜 plateau → 갑작스러운 test accuracy 상승, "지연 일반화" 재현
- **Grokking의 해석들** — Weight norm dynamics (Liu et al. 2022), "slingshot effect", representation learning의 전이, progress measure
- **Implicit Bias of SGD** — Soudry et al. 2018: logistic loss에서 GD가 max-margin solution으로 수렴 (separable case), 이것이 왜 일반화를 설명하나
- **Shah et al. 2020 "Pitfalls of Simplicity Bias"** — SGD의 simplicity bias가 항상 좋은 것은 아님, 쉬운 feature만 배우고 robust feature 무시, shortcut learning

### Chapter 6: Lottery Ticket Hypothesis와 Pruning Theory (4개 문서)
- **Lottery Ticket Hypothesis (Frankle & Carbin 2019)** — 큰 NN 안에 작은 "winning ticket" 서브네트워크 존재, 혼자 훈련해도 원 성능 달성, magnitude pruning + rewinding
- **Stable Lottery Tickets (Frankle et al. 2020)** — 초기화 정확한 값이 아닌 early training point로 rewinding, linear mode connectivity, 큰 NN에서의 확장성
- **비평과 반론 (Liu et al. 2019)** — "Rethinking the Value of Network Pruning", 무작위 초기화 + scratch retrain도 비슷한 성능, LTH의 조건부 타당성
- **Pruning Theory — Strong LTH** — Ramanujan et al. 2020: pruning만으로 훈련 없이 좋은 서브네트워크 찾기, "hidden in the initialization", over-parameterization의 의미

### Chapter 7: Neural Scaling Laws와 현대 현상 (4개 문서)
- **Chinchilla Scaling Laws (Hoffmann et al. 2022)** — Loss = $A/N^\alpha + B/D^\beta + E$, compute-optimal 훈련, Kaplan et al. 2020과의 차이 (데이터 크기 중요성)
- **Broken Neural Scaling Laws (Caballero 2022)** — 단일 power law 너머의 현상, emergent abilities, scale별 상이한 behavior
- **Emergent Abilities (Wei et al. 2022)** — 특정 능력이 scale에서 갑자기 나타남, chain-of-thought, in-context learning, 측정 방식의 artifact라는 반론 (Schaeffer et al. 2023)
- **In-Context Learning의 이론** — Prompt에서 예제로 학습하는 현상, gradient descent와의 등가성 주장 (Akyürek et al. 2023, von Oswald et al. 2023), attention = 학습된 optimizer

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **33개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 현상이 딥러닝 이해에 중요한가
## 📐 수학적 선행 조건 (SLT, NN Theory, Kernel, FA 레포 참조)
## 📖 직관적 이해
   — 현상의 physical/geometric 직관
## ✏️ 엄밀한 정의·정리
## 🔬 증명 또는 수학적 유도
   — NTK 수렴, Double Descent asymptotic, PAC-Bayes
## 💻 실험 재현
   — Zhang et al., Belkin et al., Frankle et al. 원 논문 실험 재현
   — Grokking, Double Descent toy 예제
## 🔗 이론과 실전의 간극
   — 이 이론이 실전 딥러닝을 얼마나 설명하는가
## ⚖️ 가정과 한계
   — 무한폭? Lazy regime? Toy problem?
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **모든 주요 논문 실험 재현** — Zhang 2017, Belkin 2019, Frankle 2019, Power 2022 모두 NumPy/PyTorch로
2. **Classic vs Modern 대비 시각화** — U-shape vs Double Descent, uniform bound vs NTK
3. **NTK 시각화** — 훈련 중 empirical NTK 변화 추적, width 증가에 따른 수렴 확인
4. **Scaling law plot** — Loss vs compute/data/params log-log plot, Chinchilla 재현
5. **Grokking 재현** — Modular arithmetic toy에서 train/test accuracy 분리 관찰
6. **역사적 맥락** — 각 발견 년도와 이후 반론·확장 타임라인 필수

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
matplotlib==3.8.0
torch==2.1.0
neural-tangents==0.6.0   # NTK 라이브러리
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Random Fourier Features로 Double Descent 재현)
import numpy as np
import matplotlib.pyplot as plt

np.random.seed(42)

# True function + noise
n = 100
X_train = np.random.uniform(-1, 1, (n, 1))
y_train = np.sin(np.pi * X_train).flatten() + 0.3 * np.random.randn(n)
X_test = np.linspace(-1, 1, 500).reshape(-1, 1)
y_test = np.sin(np.pi * X_test).flatten()

def rff_regression(X_train, y_train, X_test, p, sigma=0.5, lam=1e-6):
    """Random Fourier Features ridge regression"""
    d = X_train.shape[1]
    W = np.random.randn(d, p) / sigma
    b = np.random.uniform(0, 2*np.pi, p)
    Phi_train = np.cos(X_train @ W + b)
    Phi_test = np.cos(X_test @ W + b)
    beta = np.linalg.solve(Phi_train.T @ Phi_train + lam * np.eye(p),
                           Phi_train.T @ y_train)
    y_pred_train = Phi_train @ beta
    y_pred_test = Phi_test @ beta
    return y_pred_train, y_pred_test

# p를 변화시키며 train/test error 측정
p_list = [5, 10, 20, 50, 80, 95, 99, 100, 101, 105, 150, 300, 1000]
train_errs, test_errs = [], []
for p in p_list:
    errs_tr, errs_te = [], []
    for _ in range(20):  # 여러 random features 평균
        y_pr, y_pe = rff_regression(X_train, y_train, X_test, p, lam=1e-8)
        errs_tr.append(np.mean((y_pr - y_train) ** 2))
        errs_te.append(np.mean((y_pe - y_test) ** 2))
    train_errs.append(np.mean(errs_tr))
    test_errs.append(np.mean(errs_te))

plt.figure(figsize=(10, 5))
plt.loglog(p_list, train_errs, 'o-', label='Train MSE')
plt.loglog(p_list, test_errs, 's-', label='Test MSE')
plt.axvline(n, linestyle='--', color='r', label=f'interpolation threshold p=n={n}')
plt.xlabel('p (number of features)'); plt.ylabel('MSE (log)')
plt.title('Double Descent: p=n 부근에서 test error 발산, 초과 후 다시 감소')
plt.legend(); plt.show()

# Grokking 재현 — modular addition
# Task: a + b mod P where P=97
# Expected: train quickly fits, test takes much longer
import torch
import torch.nn as nn

# 간략한 재현 - 실제 실험은 더 오래 걸림
class GrokNet(nn.Module):
    def __init__(self, P=97, d=128):
        super().__init__()
        self.emb = nn.Embedding(P, d)
        self.mlp = nn.Sequential(nn.Linear(2*d, d), nn.ReLU(), nn.Linear(d, P))
    def forward(self, a, b):
        return self.mlp(torch.cat([self.emb(a), self.emb(b)], dim=-1))

# 훈련 loop, train_acc와 test_acc 분리 추적
# train_acc가 먼저 100%, test_acc는 plateau 후 갑자기 상승 → Grokking
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "SLT, NN Theory, Kernel, FA 선행 필수" 명시
   - "SLT의 한계 → 이 레포의 필요성" bridge
   - 현재 진행 중인 연구 영역으로 표시 (정답이 없는 열린 문제들)
3. **챕터별 문서 작성**: 고전의 실패 → Norm-based → NTK → Double Descent → Grokking → LTH → Scaling

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Understanding Deep Learning Requires Rethinking Generalization** (Zhang et al. 2017) — 가장 유명한 퍼즐 제기
- **Reconciling Modern Machine Learning Practice and the Classical Bias-Variance Tradeoff** (Belkin et al. 2019) — Double Descent
- **Deep Double Descent** (Nakkiran et al. 2019)
- **Neural Tangent Kernel: Convergence and Generalization** (Jacot, Gabriel, Hongler 2018) — NTK 원전
- **Wide Neural Networks of Any Depth Evolve as Linear Models** (Lee et al. 2019)
- **The Lottery Ticket Hypothesis** (Frankle & Carbin 2019)
- **Grokking: Generalization Beyond Overfitting** (Power et al. 2022)
- **Scaling Laws for Neural Language Models** (Kaplan et al. 2020)
- **Training Compute-Optimal Large Language Models** (Hoffmann et al. 2022) — Chinchilla
- **Uniform Convergence May Be Unable to Explain Generalization** (Nagarajan & Kolter 2019)
- **Spectrally-Normalized Margin Bounds** (Bartlett, Foster, Telgarsky 2017)
- **Computing Nonvacuous Generalization Bounds** (Dziugaite & Roy 2017)

---

## 💡 핵심 분석 대상

```
딥러닝 일반화 이론의 지도

┌─── 고전 이론의 실패 ───┐
│                          │
│ VC bound:               │
│   NN의 VC ~ O(W log W)  │
│   ImageNet: gap ≤ 10¹²  │
│   → vacuous (의미없음)  │
│                          │
│ Zhang 2017 실험:        │
│   random label에도 fit  │
│   → uniform conv의 한계 │
│                          │
│ Rademacher bound:       │
│   norm-based도 크면 vacuous│
│                          │
│ 결론: 새 이론 필요       │
└──────────────────────────┘
            │
            ▼
┌─── 현대 이론의 4 방향 ───┐

1. Norm-based (refinement)
   ├── Margin theory (Bartlett 2017)
   │   gap ≤ O(prod‖W_l‖_σ / √n)
   ├── PAC-Bayes (Dziugaite 2017)
   │   첫 non-vacuous bound
   ├── Path-norm (Neyshabur)
   │   scale-invariant
   └── Compression (Arora 2018)

2. NTK / Lazy regime
   ├── NTK = lim_{w→∞} <∇f, ∇f>
   │   → 상수 kernel (Jacot 2018)
   ├── Training = kernel regression
   │   f(x; θ_t) - f(x; θ_0) ≈ NTK 해
   ├── NNGP = 초기화의 GP limit
   │   (Lee 2018, Matthews 2018)
   ├── Feature learning과의 차이
   │   Lazy ≠ Rich
   │   Chizat 2019
   └── 실전 NN 일부 설명

3. Double Descent
   ├── p < n: U-shape (classic)
   ├── p = n: test error ↑ (peak)
   ├── p > n: 다시 ↓ (modern)
   │
   ├── Random Features 정확한 분석
   │   Mei & Montanari 2019
   │   Hastie 2019 (ridgeless)
   ├── Model-wise, sample-wise,
   │   epoch-wise 세 가지 형태
   │   (Nakkiran 2019)
   └── 실전 regularization으로 peak 제거

4. Implicit Bias
   ├── GD of logistic → max margin
   │   (Soudry 2018)
   ├── Simplicity bias
   │   shortcut learning 위험
   │   (Shah 2020)
   └── SGD의 flat minima 선호

───── 현대 현상들 ─────

Grokking (Power 2022):
  Train acc → 100% 먼저
  Test acc → plateau → 갑자기 100%
  Modular arithmetic 실험
  해석: representation phase transition

Lottery Ticket (Frankle 2019):
  Large NN 내부에 작은 winning ticket
  Magnitude pruning + rewinding
  Initialization의 운명?
  Liu 2019 반론: random init도 OK
  Strong LTH (Ramanujan 2020):
    Pruning만으로 훈련 없이 좋은 NN

Scaling Laws:
  Kaplan 2020:
    Loss ∝ N^(-α) · D^(-β) · C^(-γ)
  Chinchilla (Hoffmann 2022):
    Data 중요성 재평가
    N_opt ∝ C^0.5, D_opt ∝ C^0.5
  
Emergent Abilities (Wei 2022):
  작은 모델 chance → 큰 모델 갑자기 가능
  Chain-of-thought, in-context learning
  반론: measurement artifact (Schaeffer 2023)

In-Context Learning:
  Few-shot prompt로 학습
  Attention = 학습된 optimizer?
  (von Oswald 2023, Akyürek 2023)

───── 레포 간 연결 ─────

SLT 레포 (Layer 1):
  고전 bound (VC, Rademacher)
  → 이 레포의 "왜 고전이 실패하는가"

Kernel Methods (Layer 1):
  RKHS, kernel regression
  → NTK의 수학적 기초

Functional Analysis (Layer 0):
  RKHS의 엄밀한 기초
  Mercer 정리, Moore-Aronszajn
  → NTK가 재생커널

NN Theory (Layer 2):
  UAT, 초기화, 아키텍처
  → 이 레포가 "왜 훈련이 일반화하는가"

Optimization Theory (Layer 2):
  SGD, implicit bias
  → Grokking, max-margin 수렴

Regularization Theory (Layer 2, 다음):
  명시적 regularization
  → implicit regularization과 대비
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·현상·논문 (3~4줄)
- 전체 문서 개수 확인 (33개 목표)
- Python + NumPy + PyTorch + neural-tangents 실험 환경
- SLT, NN Theory, Kernel, FA 레포와의 참조 관계
- 이 레포가 "열린 문제들"을 다루는 특성 명시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
