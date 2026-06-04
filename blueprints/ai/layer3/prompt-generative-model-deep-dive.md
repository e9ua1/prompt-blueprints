# Generative Model Deep Dive 레포지토리 제작 프롬프트

나는 "Generative Model Deep Dive" 레포지토리를 만들려고 해.
GAN의 minimax 목표 $\min_G \max_D V(D, G) = \mathbb{E}_{x \sim p_{\text{data}}}[\log D(x)] + \mathbb{E}_{z \sim p_z}[\log(1 - D(G(z)))]$를 **아는 것**과, **최적 discriminator $D^*(x) = p_{\text{data}}(x)/(p_{\text{data}}(x) + p_g(x))$를 대입하면 $V$가 Jensen-Shannon divergence $2 \cdot JSD(p_{\text{data}} \| p_g) - \log 4$로 환원**되는 Goodfellow 2014 증명을 유도할 수 있는 것은 다르다.
VAE의 ELBO를 **쓰는 것**과, **$\text{ELBO} = \mathbb{E}_q[\log p(x|z)] - \text{KL}(q(z|x) \| p(z))$가 정확히 reconstruction loss + regularization으로 분해**되고, **reparameterization trick이 Monte Carlo 기댓값의 low-variance gradient**를 어떻게 가능케 하는지 증명할 수 있는 것은 다르다.
Normalizing Flow를 **듣는 것**과, **change of variables $\log p(x) = \log p(z) - \sum_l \log |\det J_{f_l}|$이 invertible transformation을 통해 정확한 likelihood를 주는 수학**과, **RealNVP의 coupling layer가 삼각 Jacobian으로 $\det$을 $O(n)$에 계산 가능케 하는 설계**를 유도할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "생성 모델의 수학 — 암묵적(GAN) vs 명시적(VAE, Flow, Diffusion) likelihood의 통합 비교"

**핵심 차별화**:
1. **GAN의 수렴 이론 완전 분석** — Goodfellow 2014의 JSD 유도, Nash equilibrium, mode collapse의 수학, Wasserstein GAN의 1-Lipschitz 제약과 증명
2. **VAE의 정보이론적 해석** — ELBO의 rate-distortion, β-VAE의 정보 bottleneck, posterior collapse의 원인과 해결
3. **Normalizing Flow의 전체 계보** — RealNVP → Glow → Neural ODE → Continuous Normalizing Flow, change of variables의 엄밀 유도
4. **Diffusion Model과 기존 모델과의 통합** — DDPM의 ELBO = weighted denoising score matching, Score-SDE로 모든 것을 통합 (SDE 레포 교차)

**타겟 독자**:
- GAN을 훈련하는데 mode collapse 발생 시 **어떤 수학적 원인이 있는지**, WGAN/Spectral Norm 같은 해결책의 **이론적 근거**를 모르는 사람
- VAE의 posterior collapse 현상을 **KL 항의 수학적 분석**으로 설명 못하는 사람
- Normalizing Flow가 왜 GAN보다 느리고 VAE보다 memory 집약적인지, **Jacobian $\det$ 계산의 복잡도**에서 오는 trade-off를 모르는 사람
- Stable Diffusion을 사용하는데 **classifier-free guidance, DDIM의 수학적 trick**을 이해 못하는 사람
- Autoregressive (PixelCNN, GPT)와 위 4가지의 관계, **모든 generative model의 우열**을 비교 못하는 사람

**선행 학습**:
- **Probability Theory Deep Dive** (분포, 기댓값, KL) — **필수**
- **Information Theory Deep Dive** (Entropy, KL, JSD, MI) — **필수**
- **Bayesian ML Deep Dive** (VAE, ELBO, Variational Inference) — **필수**
- **Stochastic Differential Equations Deep Dive** (Diffusion, Score matching) — **Diffusion에 필수**
- **Neural Network Theory Deep Dive** (Backprop, architecture) — **필수**
- **Optimization Theory Deep Dive** (Minimax, saddle point) — **GAN에 필수**
- **Convex Optimization Deep Dive** (Dual, Lagrangian) — WGAN에 유용

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Generative Modeling의 수학적 분류 (4개 문서)
- **Generative vs Discriminative 모델** — $p(x)$ (density estimation) vs $p(y|x)$ (classification), 생성 모델의 4가지 역할 (샘플링, likelihood, representation, anomaly detection), Bayes 정리로 연결
- **Explicit vs Implicit Likelihood 모델** — Autoregressive·Flow·VAE·Diffusion = explicit (tractable likelihood), GAN·EBM = implicit (sampling만 가능), 각 접근의 장단점
- **모든 생성 모델의 통합 목표 — $\text{KL}(p_{\text{data}} \| p_\theta)$ 최소화** — MLE가 KL 최소화와 동치, 각 모델이 이 목표를 어떻게 근사 (직접 / ELBO / minimax / score matching)
- **Evaluation Metrics — IS, FID, Precision/Recall, NLL** — Inception Score, Fréchet Inception Distance의 수학, generation quality 측정, likelihood-based vs perception-based

### Chapter 2: Autoregressive Model (4개 문서)
- **Autoregressive 분해** — $p(x) = \prod_i p(x_i | x_{<i})$, Chain rule, 순서 선택이 modeling에 미치는 영향
- **PixelRNN, PixelCNN (van den Oord 2016)** — 이미지를 pixel sequence로, masked convolution으로 future 참조 방지, row·column LSTM, conditional generation
- **WaveNet (van den Oord 2016)** — Audio의 autoregressive 생성, dilated causal convolution으로 long-range dependency, generation 속도 문제
- **Autoregressive Transformer — GPT as Generative Model** — 자연스럽게 autoregressive, text·image (ImageGPT, Parti)·audio·video 생성, scale로 품질 향상, Transformer 레포와 연결

### Chapter 3: Variational Autoencoder (VAE) (5개 문서)
- **VAE의 유도 (Kingma & Welling 2013)** — Latent model $p(x, z) = p(x|z) p(z)$, $\log p(x) = \mathcal{L}(\theta, \phi; x) + \text{KL}(q_\phi(z|x) \| p(z|x))$로 ELBO 유도, amortized inference
- **Reparameterization Trick 재검증** — $z \sim q_\phi(z|x)$를 $z = \mu + \sigma \odot \epsilon, \epsilon \sim \mathcal{N}(0, I)$로, gradient 통과의 수학적 정당성 (Bayesian ML 레포와 교차)
- **β-VAE와 Information Bottleneck (Higgins 2017)** — $\mathcal{L} = \mathbb{E}[\log p(x|z)] - \beta \text{KL}(q(z|x) \| p(z))$, $\beta$의 역할: disentangled representation, rate-distortion trade-off
- **Posterior Collapse의 원인과 해결** — Decoder가 강력할 때 $q(z|x) \approx p(z)$로 붕괴 → latent 사용 안함, 원인(KL 항이 지배, 강한 decoder), 해결책(KL annealing, Free Bits, δ-VAE)
- **VQ-VAE와 Discrete Latent (van den Oord 2017)** — Continuous latent 대신 codebook $\{e_1, ..., e_K\}$, straight-through estimator, DALL-E·Jukebox의 기반

### Chapter 4: Normalizing Flow (5개 문서)
- **Change of Variables와 Flow의 정의** — $x = f_\theta(z)$, $\log p(x) = \log p(z) - \log |\det J_f(z)|$, invertible transform의 체인 $\prod |\det J|$, exact likelihood 제공
- **RealNVP — Coupling Layer (Dinh 2017)** — 입력을 두 부분 $x_{1:d}, x_{d+1:D}$로 분리, $y_{1:d} = x_{1:d}$, $y_{d+1:D} = x_{d+1:D} \odot \exp(s(x_{1:d})) + t(x_{1:d})$, 삼각 Jacobian → $|\det| = \exp(\sum s)$
- **Glow (Kingma & Dhariwal 2018)** — $1 \times 1$ invertible convolution 도입, coupling + activation normalization + $1 \times 1$ conv, 고해상도 이미지 생성
- **Autoregressive Flow — MAF, IAF** — Masked Autoregressive Flow (Papamakarios 2017): $x_i = f(z_i, x_{<i})$, Inverse AF (Kingma 2016): 반대 방향, 각각 likelihood·sampling 속도 trade-off
- **Continuous Normalizing Flow와 Neural ODE** — $\frac{dz}{dt} = f_\theta(z(t), t)$, $\log p(z(T)) - \log p(z(0)) = -\int_0^T \text{tr}(\partial f / \partial z) dt$, FFJORD의 Hutchinson trace estimator

### Chapter 5: Generative Adversarial Network (GAN) (6개 문서)
- **GAN의 수학적 정식화 (Goodfellow 2014)** — $\min_G \max_D V(D, G) = \mathbb{E}_{x \sim p_d}[\log D(x)] + \mathbb{E}_{z \sim p_z}[\log(1 - D(G(z)))]$, two-player zero-sum game
- **최적 D에서 V의 JSD 환원 증명** — $D^*(x) = p_d(x) / (p_d(x) + p_g(x))$를 대입하면 $V(D^*, G) = 2 \cdot JSD(p_d \| p_g) - \log 4$, minimax 해가 $p_g = p_d$인 이유
- **GAN 훈련의 불안정성과 Mode Collapse** — JSD의 gradient가 non-overlapping support에서 정보 없음, vanishing gradient 문제, generator가 일부 mode만 학습하는 현상
- **Wasserstein GAN (Arjovsky 2017)** — Wasserstein-1 distance $W(p_d, p_g) = \inf \mathbb{E}[\|X - Y\|]$, Kantorovich-Rubinstein 쌍대로 $\sup_{\|f\|_L \leq 1} \mathbb{E}_{p_d}[f] - \mathbb{E}_{p_g}[f]$, 1-Lipschitz 제약을 weight clipping 또는 gradient penalty (WGAN-GP, Gulrajani 2017)
- **Spectral Normalization (Miyato 2018)** — 각 weight matrix의 spectral norm을 1로, Lipschitz 보장, WGAN보다 구현 간단, 이론적 정당성
- **Progressive GAN, StyleGAN, StyleGAN2** — Progressive growing, style injection via AdaIN, noise injection, 최고 품질 얼굴 생성, mode coverage 개선

### Chapter 6: Diffusion Model (5개 문서)
- **Denoising Diffusion Probabilistic Models (Ho 2020)** — Forward: $q(x_t | x_{t-1}) = \mathcal{N}(\sqrt{1-\beta_t} x_{t-1}, \beta_t I)$, reverse: $p_\theta(x_{t-1} | x_t) = \mathcal{N}(\mu_\theta, \Sigma_\theta)$, ELBO 유도
- **DDPM Loss의 단순화** — ELBO를 parametrize하면 $L_{\text{simple}} = \mathbb{E}_{t, x_0, \epsilon}[\|\epsilon - \epsilon_\theta(x_t, t)\|^2]$로 환원, noise prediction이 reverse process 학습과 동치
- **Score-Based Model (Song & Ermon 2019)** — Score $\nabla_x \log p(x)$ 학습, Denoising Score Matching으로 손실 유도, DDPM과의 등가성 (weighted score matching)
- **Score-SDE (Song 2021)** — Forward/Reverse SDE 프레임워크, $dx = -f(x, t) dt + g(t) dW$ (forward), reverse-time SDE로 generation, SDE 레포와의 연결
- **Classifier-Free Guidance와 현대 응용** — Ho & Salimans 2022: $\tilde{\epsilon} = (1 + w) \epsilon_\theta(x_t, y) - w \epsilon_\theta(x_t, \emptyset)$, Stable Diffusion·DALL-E 3·Imagen의 핵심

### Chapter 7: 통합과 최신 동향 (4개 문서)
- **Autoregressive vs VAE vs Flow vs GAN vs Diffusion — 통합 비교** — 각 모델의 likelihood tractability, sampling 속도, sample quality, 수식·메커니즘 비교 표, trade-off 정리
- **Consistency Model과 Rectified Flow** — Diffusion의 sampling 속도 문제 해결, Consistency Model (Song 2023)의 one-step generation, Rectified Flow (Liu 2022)의 straight trajectory
- **Generative Model과 Energy-Based Model** — $p(x) = e^{-E(x)}/Z$, EBM의 MCMC 훈련 어려움, 최근 부흥 (JEM, Yang 2022), Diffusion과의 관계
- **현대 생성모델의 Frontier** — Multimodal (DALL-E, Parti, GPT-4V), Video (Sora, Runway), 3D (NeRF, DreamFusion), 과학적 응용 (AlphaFold 3, material discovery)

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **33개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 generative model이 필요한가
## 📐 수학적 선행 조건 (Prob, Info, Bayes, SDE, Opt 레포 참조)
## 📖 직관적 이해
   — "생성"을 샘플링 과정의 시각화로
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — GAN JSD 환원, VAE ELBO, Flow change of variables, DDPM loss
## 💻 PyTorch 구현 검증
   — MNIST/CIFAR에서 각 모델 훈련
   — 샘플 생성 비교, FID/IS 측정
## 🔗 실전 활용
   — 언제 어느 모델 선택, hybrid 접근
## ⚖️ 가정과 한계
   — 각 모델의 실패 모드
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Sample 품질 시각화** — MNIST/CIFAR에서 각 모델의 샘플을 grid로
2. **Training dynamics plot** — Loss curve, FID 변화, mode coverage 시각화
3. **Latent space 탐색** — VAE, GAN의 latent interpolation, attribute arithmetic
4. **Diffusion process 시각화** — forward noise 주입 → reverse denoising 과정
5. **모든 모델 bench table** — Params, Train time, Sample quality, NLL, Sample speed
6. **실패 사례 재현** — GAN mode collapse, VAE blur, posterior collapse

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
torchvision==0.16.0
matplotlib==3.8.0
diffusers==0.25.0     # Diffusion 참조
normflows==1.7        # Flow 참조
scipy==1.11.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (GAN JSD 환원 확인 + DDPM 1D)
import torch
import torch.nn as nn
import numpy as np
import matplotlib.pyplot as plt

# 1. GAN JSD 환원 수치 확인
# 최적 D에서 V = 2 JSD(p_d || p_g) - log 4
def compute_jsd(p, q):
    m = 0.5 * (p + q)
    kl_pm = (p * np.log((p + 1e-10) / (m + 1e-10))).sum()
    kl_qm = (q * np.log((q + 1e-10) / (m + 1e-10))).sum()
    return 0.5 * (kl_pm + kl_qm)

x_grid = np.linspace(-5, 5, 200)
p_data = np.exp(-0.5 * (x_grid - 1)**2) / np.sqrt(2*np.pi)
p_gen = np.exp(-0.5 * (x_grid + 0.5)**2) / np.sqrt(2*np.pi)
p_data /= p_data.sum(); p_gen /= p_gen.sum()

# Optimal D*(x) = p_d / (p_d + p_g)
D_star = p_data / (p_data + p_gen + 1e-10)
V_optimal = (p_data * np.log(D_star + 1e-10)).sum() + \
            (p_gen * np.log(1 - D_star + 1e-10)).sum()

JSD = compute_jsd(p_data, p_gen)
V_from_JSD = 2 * JSD - np.log(4)
print(f'V(D*, G) = {V_optimal:.4f}')
print(f'2·JSD - log 4 = {V_from_JSD:.4f}')
# 두 값이 일치 (정리 확인)

# 2. 1D DDPM
class DDPM1D(nn.Module):
    def __init__(self, hidden=128):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(2, hidden), nn.SiLU(),
            nn.Linear(hidden, hidden), nn.SiLU(),
            nn.Linear(hidden, 1)
        )
    def forward(self, x, t):
        return self.net(torch.cat([x, t.float().unsqueeze(-1)], -1))

T = 100
betas = torch.linspace(1e-4, 0.02, T)
alphas = 1 - betas
alphas_cumprod = torch.cumprod(alphas, 0)

def forward_diffusion(x0, t, eps):
    a = alphas_cumprod[t].unsqueeze(-1)
    return torch.sqrt(a) * x0 + torch.sqrt(1 - a) * eps

# Training: noise prediction
model = DDPM1D()
opt = torch.optim.Adam(model.parameters(), lr=1e-3)

# Target: mixture of 2 gaussians
for step in range(3000):
    x0 = torch.randn(256, 1)
    x0 = torch.where(torch.rand(256, 1) > 0.5, x0 * 0.5 + 2, x0 * 0.5 - 2)
    t = torch.randint(0, T, (256,))
    eps = torch.randn_like(x0)
    xt = forward_diffusion(x0, t, eps)
    eps_pred = model(xt, t)
    loss = ((eps - eps_pred)**2).mean()
    opt.zero_grad(); loss.backward(); opt.step()

# Sampling
@torch.no_grad()
def sample(n=1000):
    x = torch.randn(n, 1)
    for t in reversed(range(T)):
        z = torch.randn_like(x) if t > 0 else 0
        a, ac, b = alphas[t], alphas_cumprod[t], betas[t]
        eps = model(x, torch.full((n,), t))
        x = (1/torch.sqrt(a)) * (x - (b/torch.sqrt(1-ac)) * eps) + torch.sqrt(b) * z
    return x

samples = sample(2000).numpy().flatten()
plt.hist(samples, bins=50, density=True, alpha=0.5, label='DDPM samples')
# plt.plot(x_grid, target_density, label='target')
plt.title('1D DDPM: 2-modal Gaussian mixture 학습'); plt.legend(); plt.show()
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Prob, Info, Bayes, SDE, NN Theory, Opt 선행 필수" 명시
   - "5가지 모델 통합 비교"를 프론트 페이지에
   - Stable Diffusion, GPT image generation 등 현대 실전 응용
3. **챕터별 문서 작성**: 분류 → Autoregressive → VAE → Flow → GAN → Diffusion → 통합

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Deep Learning** (Goodfellow) — Chap 20 생성모델
- **Generative Adversarial Nets** (Goodfellow et al. 2014) — GAN 원전
- **Auto-Encoding Variational Bayes** (Kingma & Welling 2013) — VAE
- **Pixel Recurrent Neural Networks** (van den Oord et al. 2016) — PixelRNN
- **Density Estimation Using Real NVP** (Dinh et al. 2017) — RealNVP
- **Glow** (Kingma & Dhariwal 2018)
- **FFJORD: Free-Form Continuous Dynamics** (Grathwohl et al. 2019)
- **Wasserstein GAN** (Arjovsky et al. 2017)
- **Improved Training of Wasserstein GANs** (Gulrajani et al. 2017) — WGAN-GP
- **Spectral Normalization for GANs** (Miyato et al. 2018)
- **Denoising Diffusion Probabilistic Models** (Ho et al. 2020) — DDPM
- **Score-Based Generative Modeling through SDEs** (Song et al. 2021)
- **Classifier-Free Diffusion Guidance** (Ho & Salimans 2022)
- **A Style-Based Generator Architecture for GANs** (Karras et al. 2019) — StyleGAN

---

## 💡 핵심 분석 대상

```
Generative Model의 5대 계보

모든 모델의 목표: p_θ(x) ≈ p_data(x)
  MLE ≡ min KL(p_data || p_θ)

───── Autoregressive ─────

p(x) = ∏_i p(x_i | x_{<i})
  ├── tractable likelihood
  ├── sequential sampling (느림)
  └── PixelRNN, WaveNet, GPT

───── VAE ─────

p(x) = ∫ p(x|z) p(z) dz  (intractable)

ELBO:
  log p(x) ≥ L = 𝔼_q[log p(x|z)] - KL(q(z|x) || p(z))
               └── reconstruction ─┘ └── regularizer ─┘

Reparameterization:
  z = μ + σ ⊙ ε, ε ~ N(0, I)
  → gradient 통과 가능

Issues:
  Blurry samples (Gaussian decoder)
  Posterior collapse (decoder 강할 때)

β-VAE:
  β > 1 → disentangled
  Information Bottleneck 관점

VQ-VAE: discrete codebook

───── Normalizing Flow ─────

x = f_θ(z), invertible
log p(x) = log p(z) - log|det J_f|

장점:
  ├── Exact likelihood
  └── Exact inference

도전:
  Invertibility + tractable det
  
Coupling layer (RealNVP):
  y_{1:d} = x_{1:d}
  y_{d+1:D} = x_{d+1:D} ⊙ exp(s) + t
  → 삼각 Jacobian, det = exp(Σ s)

Glow: 1×1 invertible conv

CNF (Neural ODE):
  dz/dt = f_θ(z, t)
  log p(z_T) = log p(z_0) - ∫ tr(∂f/∂z) dt
  FFJORD: Hutchinson trace est.

───── GAN ─────

min_G max_D V(D, G):
  V = 𝔼_pd[log D(x)] + 𝔼_pz[log(1 - D(G(z)))]

Optimal D:
  D*(x) = p_d(x) / (p_d(x) + p_g(x))

V(D*, G) = 2·JSD(p_d || p_g) - log 4

Issues:
  ├── JSD gradient 소실
  ├── Mode collapse
  ├── Training 불안정
  └── No likelihood

Wasserstein GAN:
  W(p_d, p_g) = inf 𝔼‖X-Y‖
  Kantorovich-Rubinstein:
    = sup_{‖f‖_L ≤ 1} 𝔼_pd[f] - 𝔼_pg[f]
  → discriminator는 1-Lipschitz

  Enforcement:
    WGAN: weight clipping
    WGAN-GP: gradient penalty
    Spectral Norm: ‖W‖_σ = 1

Modern: StyleGAN, BigGAN, ProgGAN

───── Diffusion ─────

Forward (fixed):
  q(x_t | x_{t-1}) = N(√{1-β_t} x_{t-1}, β_t I)
  
  x_t = √{ᾱ_t} x_0 + √{1-ᾱ_t} ε

Reverse (learned):
  p_θ(x_{t-1} | x_t) = N(μ_θ, Σ_θ)

ELBO → simple loss:
  L = 𝔼[‖ε - ε_θ(x_t, t)‖²]

Score-based view:
  Score: s(x) = ∇_x log p(x)
  Denoising Score Matching (Vincent 2011)
  DDPM ≡ weighted DSM

Score-SDE (Song 2021):
  Forward: dx = f(x,t)dt + g(t)dW
  Reverse: dx = [f - g²∇log p_t]dt + g dW̄
  → 통합 framework

Classifier-Free Guidance:
  ε̃ = (1+w) ε_θ(x, y) - w ε_θ(x, ∅)

현대 응용:
  Stable Diffusion, DALL-E 3, Imagen
  Sora (video), DreamFusion (3D)

───── 5대 계보 비교 ─────

┌──────────────┬───────┬─────────┬─────────┬──────────┐
│ Model        │ Like  │ Sample  │ Quality │ Training │
├──────────────┼───────┼─────────┼─────────┼──────────┤
│ Autoregress. │ Exact │ Slow    │ Good    │ Stable   │
│ VAE          │ Lower │ Fast    │ Blurry  │ Stable   │
│ Flow         │ Exact │ Medium  │ Medium  │ Stable   │
│ GAN          │ None  │ Fast    │ Sharp   │ Unstable │
│ Diffusion    │ Lower │ Slow    │ SOTA    │ Stable   │
└──────────────┴───────┴─────────┴─────────┴──────────┘

───── 레포 간 연결 ─────

Probability (Layer 0):
  기본 분포, KL, JSD

Info Theory (Layer 0):
  KL, JSD, Wasserstein
  ELBO가 rate-distortion

Bayesian ML (Layer 1):
  VAE의 VI 기반
  ELBO 분해

SDE (Layer 0):
  Diffusion의 연속시간 framework
  Score-SDE의 forward/reverse

Convex Opt (Layer 0):
  WGAN의 Kantorovich-Rubinstein
  Lagrangian duality

NN Theory (Layer 2):
  모든 모델의 NN parameterization

Optimization (Layer 2):
  GAN의 minimax 훈련
  Saddle point, Nash equilibrium

Transformer (Layer 3):
  Autoregressive (GPT)
  Diffusion Transformer (DiT)
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (33개 목표)
- Python + PyTorch + diffusers 실험 환경
- Prob, Info, Bayes, SDE, NN Theory, Opt, Convex, Transformer 레포의 참조 관계
- 현대 multimodal 생성으로 이어지는 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
