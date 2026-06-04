# Diffusion Model Deep Dive 레포지토리 제작 프롬프트

나는 "Diffusion Model Deep Dive" 레포지토리를 만들려고 해.
DDPM을 **사용하는 것**과, **Ho et al. (2020)의 forward process $q(x_t | x_{t-1}) = \mathcal{N}(\sqrt{1-\beta_t} x_{t-1}, \beta_t I)$에서 closed-form $q(x_t | x_0) = \mathcal{N}(\sqrt{\bar{\alpha}_t} x_0, (1-\bar{\alpha}_t) I)$을 유도**하고, **Reverse ELBO를 unrolled하면 $L_{\text{vlb}} = L_T + \sum_{t=2}^T L_{t-1} + L_0$, 그리고 noise prediction parametrization으로 $L_{\text{simple}} = \mathbb{E}_{t, x_0, \epsilon}[\|\epsilon - \epsilon_\theta(x_t, t)\|^2]$로 간소화**되는 전 과정을 증명할 수 있는 것은 다르다.
Score Matching을 **이름으로 아는 것**과, **Vincent (2011)의 Denoising Score Matching $\mathbb{E}_{q(x,\tilde{x})}[\|s_\theta(\tilde{x}) - \nabla_{\tilde{x}} \log q(\tilde{x}|x)\|^2]$이 DDPM의 noise prediction과 수학적으로 동등**하고, **Song & Ermon (2021)의 Score-SDE가 forward SDE $dx = f(x, t) dt + g(t) dW$와 reverse SDE $dx = [f(x, t) - g(t)^2 \nabla_x \log p_t(x)] dt + g(t) d\bar{W}$로 모든 diffusion을 통합**함을 유도할 수 있는 것은 다르다.
DDIM을 **"빠른 샘플링"으로 아는 것**과, **Song et al. (2021)의 non-Markovian forward process $q_\sigma(x_{t-1} | x_t, x_0)$가 어떻게 $\sigma \to 0$에서 deterministic ODE가 되고**, **이것이 sampling step을 50으로 줄여도 품질이 유지되는 이유**를 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "Diffusion Model의 수학 — DDPM·Score-SDE·DDIM·Flow Matching을 통합된 stochastic process 관점에서"

**핵심 차별화**:
1. **DDPM ELBO의 완전 유도** — Forward closed-form, Reverse parameterization, KL 분해, noise prediction의 수학적 필연성
2. **Score Matching과 DDPM의 등가성** — Vincent 2011 DSM → Song 2019 NCSN → DDPM의 weighted score matching 해석
3. **Score-SDE의 통합 관점** — Forward/Reverse SDE 쌍, VP-SDE (Variance-Preserving) vs VE-SDE (Variance-Exploding), probability flow ODE
4. **DDIM의 Non-Markovian 구조와 ODE 극한** — $\sigma_t = 0$ 극한이 deterministic sampling, 같은 $\epsilon_\theta$로 step 10-50으로 축소

**타겟 독자**:
- Stable Diffusion을 쓰지만 **latent diffusion의 VAE + UNet 분리 구조**(Rombach 2022)가 왜 픽셀 diffusion보다 효율적인지 수학적으로 설명 못하는 사람
- Classifier-free guidance $\tilde{\epsilon} = (1+w) \epsilon_\theta(x, y) - w \epsilon_\theta(x, \emptyset)$를 쓰는데 **$w$가 왜 sharpness-diversity trade-off**인지 모르는 사람
- DDIM과 DDPM의 차이를 **"deterministic vs stochastic"**로 아는데 **수학적으로 어떻게 non-Markovian을 설계**했는지 유도 못하는 사람
- DiT (Peebles 2023)가 UNet을 Transformer로 대체했을 때 **scaling law가 더 잘 적용되는 이유**를 이해 못하는 사람
- Consistency Model (Song 2023), Rectified Flow (Liu 2022)가 **diffusion의 느린 샘플링 문제**를 각각 어떻게 해결하는지 차이를 모르는 사람

**선행 학습**:
- **Generative Model Deep Dive** (DDPM 기초, 다른 모델과 비교) — **필수**
- **Stochastic Differential Equations Deep Dive** (SDE, Fokker-Planck) — **필수** (Score-SDE)
- **Probability Theory Deep Dive** (KL, 조건부 기댓값) — **필수**
- **Information Theory Deep Dive** (ELBO, KL) — **필수**
- **Vision Transformer Deep Dive** (DiT, UNet 대안) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Diffusion Model의 수학적 기초 (5개 문서)
- **Diffusion의 물리적 기원** — Brownian motion, Fokker-Planck 방정식, thermodynamic nonequilibrium (Sohl-Dickstein 2015), DDPM의 직관적 재구성
- **Forward Process — Noise 주입의 Markov Chain** — $q(x_t | x_{t-1}) = \mathcal{N}(\sqrt{1-\beta_t} x_{t-1}, \beta_t I)$, $\beta_t$ schedule (linear, cosine), 스케일 정규화의 이유
- **Forward Closed-Form의 증명** — $\alpha_t = 1 - \beta_t$, $\bar{\alpha}_t = \prod_{s=1}^t \alpha_s$, $q(x_t | x_0) = \mathcal{N}(\sqrt{\bar{\alpha}_t} x_0, (1-\bar{\alpha}_t) I)$, Gaussian의 재파라미터화 + 정규화 기호
- **Reverse Process의 정의** — $p_\theta(x_{t-1} | x_t) = \mathcal{N}(\mu_\theta(x_t, t), \Sigma_\theta(x_t, t))$, forward가 작은 $\beta_t$면 reverse도 Gaussian 근사 가능
- **$q(x_{t-1} | x_t, x_0)$ Posterior 유도** — Bayes' rule, $q(x_{t-1} | x_t, x_0) = \mathcal{N}(\tilde{\mu}_t, \tilde{\beta}_t I)$, $\tilde{\mu}_t = \frac{\sqrt{\bar{\alpha}_{t-1}} \beta_t}{1 - \bar{\alpha}_t} x_0 + \frac{\sqrt{\alpha_t}(1-\bar{\alpha}_{t-1})}{1-\bar{\alpha}_t} x_t$

### Chapter 2: ELBO 유도와 Loss Simplification (5개 문서)
- **VLB (Variational Lower Bound) 분해** — $-\log p_\theta(x_0) \leq \mathbb{E}_q[-\log \frac{p_\theta(x_{0:T})}{q(x_{1:T}|x_0)}] = L_{\text{vlb}}$, Jensen's inequality의 수학
- **ELBO의 3개 항 분해** — $L_{\text{vlb}} = \underbrace{\text{KL}(q(x_T|x_0) \| p(x_T))}_{L_T} + \sum_{t=2}^T \underbrace{\text{KL}(q(x_{t-1}|x_t, x_0) \| p_\theta(x_{t-1}|x_t))}_{L_{t-1}} + \underbrace{-\log p_\theta(x_0|x_1)}_{L_0}$, 각 항의 의미
- **Noise Prediction Parameterization** — $\mu_\theta(x_t, t) = \frac{1}{\sqrt{\alpha_t}}(x_t - \frac{\beta_t}{\sqrt{1-\bar{\alpha}_t}} \epsilon_\theta(x_t, t))$, noise $\epsilon$ 직접 예측이 $\mu$ 예측보다 안정적
- **$L_{\text{simple}}$ 유도** — Two Gaussian KL로 $L_{t-1}$ 표현 → weighted MSE on noise → Ho et al.의 simple loss $L_{\text{simple}} = \mathbb{E}_{t, x_0, \epsilon}[\|\epsilon - \epsilon_\theta(\sqrt{\bar{\alpha}_t} x_0 + \sqrt{1-\bar{\alpha}_t} \epsilon, t)\|^2]$, weight 제거의 justification
- **Improved DDPM (Nichol & Dhariwal 2021)** — Learned variance $\Sigma_\theta$, cosine noise schedule, hybrid loss $L_{\text{simple}} + \lambda L_{\text{vlb}}$, log-likelihood 개선

### Chapter 3: Score-Based Model과 SDE (5개 문서)
- **Score Function과 Langevin Dynamics** — $s(x) = \nabla_x \log p(x)$, Langevin MCMC $x_{t+1} = x_t + \eta s(x_t) + \sqrt{2\eta} \epsilon$, 수렴하면 $p$ 분포 샘플
- **Denoising Score Matching (Vincent 2011)** — Perturbed $\tilde{x} = x + \sigma \epsilon$, $s_\theta(\tilde{x}) \approx \nabla_{\tilde{x}} \log q(\tilde{x}|x) = -(\tilde{x} - x)/\sigma^2$, 이것이 simple noise prediction과 등가
- **NCSN — Noise Conditional Score Network (Song & Ermon 2019)** — Multi-scale noise $\sigma_1 > \sigma_2 > \ldots$, $s_\theta(x, \sigma)$ 조건부 score, annealed Langevin sampling
- **Score-SDE 통합 (Song 2021)** — Forward SDE $dx = f(x, t) dt + g(t) dW$, reverse-time SDE $dx = [f(x, t) - g(t)^2 s_\theta(x, t)] dt + g(t) d\bar{W}$, Anderson 1982의 reverse-time SDE 정리
- **VP-SDE vs VE-SDE** — Variance-Preserving (DDPM): $dx = -\frac{1}{2}\beta(t) x \, dt + \sqrt{\beta(t)} dW$, Variance-Exploding (NCSN): $dx = \sqrt{\frac{d[\sigma^2(t)]}{dt}} dW$, sub-VP-SDE 등 variants

### Chapter 4: DDIM과 ODE Sampling (4개 문서)
- **DDIM 동기 — DDPM의 느린 샘플링** — 1000 step reverse가 필요, 각 step마다 UNet forward, 생성에 수 분~수십 분, 가속 필요
- **Non-Markovian Forward Process (Song 2021)** — $q_\sigma(x_{1:T}|x_0) = q_\sigma(x_T|x_0) \prod_{t=2}^T q_\sigma(x_{t-1}|x_t, x_0)$, DDPM과 같은 marginal $q(x_t | x_0)$ 유지, 다른 joint
- **DDIM Sampling Equation** — $x_{t-1} = \sqrt{\bar{\alpha}_{t-1}} \cdot \hat{x}_0 + \sqrt{1 - \bar{\alpha}_{t-1} - \sigma_t^2} \cdot \epsilon_\theta(x_t, t) + \sigma_t \cdot \epsilon_t$, $\sigma_t$ controls stochasticity
- **$\sigma_t = 0$ 극한: Probability Flow ODE** — Deterministic mapping, same $\epsilon_\theta$로 50 step (또는 10 step) 샘플링 가능, higher-order ODE solver (DPM-Solver, DPM++) 가속

### Chapter 5: Guidance와 Conditioning (5개 문서)
- **Classifier Guidance (Dhariwal & Nichol 2021)** — Pretrained classifier $p_\phi(y|x_t)$의 gradient 사용, $\tilde{s} = s_\theta(x, t) + s \cdot \nabla_x \log p_\phi(y | x_t)$, $s$가 guidance scale
- **Classifier-Free Guidance (Ho & Salimans 2022)** — Conditional과 unconditional을 하나의 network로, training 시 $\emptyset$로 conditioning drop, inference $\tilde{\epsilon} = (1 + w) \epsilon_\theta(x, y) - w \epsilon_\theta(x, \emptyset)$
- **CFG의 Trade-off 분석** — $w$ 증가 → sample이 $p(x|y)$에 더 집중, diversity ↓, quality ↑, typical $w \in [5, 15]$, dynamic thresholding (Imagen)
- **Cross-Attention으로 Text Conditioning** — Stable Diffusion의 UNet에 text embedding cross-attention, CLIP text encoder, T5의 영향 (Imagen)
- **Negative Prompt와 Compositional Generation** — $\epsilon_\theta(x, y_{\text{neg}})$를 penalty로, Universal Guidance (Bansal 2023), compositional diffusion

### Chapter 6: Latent Diffusion과 현대 아키텍처 (5개 문서)
- **Latent Diffusion — Stable Diffusion (Rombach 2022)** — Pretrained VAE로 $x \to z \in \mathbb{R}^{h \times w \times c}$ (compression), latent space에서 diffusion, 계산 비용 4-8× 감소, high-resolution 가능
- **UNet 아키텍처 (Ronneberger 2015)의 Diffusion 적용** — Encoder-decoder with skip, ResBlocks + Attention, timestep embedding injection, conditioning injection
- **DiT — Diffusion Transformer (Peebles 2023)** — UNet 대신 ViT-style Transformer, patch embedding + Transformer blocks + AdaLN-Zero, scaling law 잘 적용, SD3·Sora의 기반
- **MM-DiT (Stable Diffusion 3, Esser 2024)** — Image + text를 동일 stream으로, joint attention, modality 대칭 처리
- **Cascaded Diffusion (Imagen)** — Low-res generation → super-resolution diffusion cascade, text-to-image에서 high-res 효율

### Chapter 7: 가속과 최신 방법 (4개 문서)
- **Consistency Model (Song 2023)** — $f_\theta(x_t, t) \approx x_0$ consistently across time, one-step or few-step generation, distillation from pretrained diffusion
- **Rectified Flow (Liu 2022)** — $x_t = (1-t) x_0 + t z$, $z \sim \mathcal{N}(0, I)$, straight trajectory, $v_\theta(x_t, t) \approx z - x_0$, reflow로 점점 직선화
- **Flow Matching (Lipman 2023)** — Conditional flow matching, continuous normalizing flow의 실전적 훈련, Rectified Flow과 밀접, SD3의 기반
- **Distillation과 Sampling Speed** — Progressive distillation (Salimans 2022), SnapFusion의 압축, SDXL-Turbo·SDXL-Lightning의 real-time generation

---

각 챕터는 **4~5개 문서**로 구성해줘. 총 **33개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 방법이 diffusion의 핵심인가
## 📐 수학적 선행 조건 (Generative, SDE, Prob, Info 참조)
## 📖 직관적 이해
   — Forward/Reverse 시각화, latent space 이동
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Forward closed-form, ELBO 분해, DDIM ODE, CFG 유도
## 💻 PyTorch 구현 검증
   — 1D/2D toy에서 DDPM 바닥부터
   — DDIM 가속 샘플링 비교
   — Stable Diffusion inference 재현
## 🔗 실전 활용
   — Text-to-image, image editing, 3D, video
## ⚖️ 가정과 한계
   — Sampling cost, training cost, mode coverage
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Forward/Reverse 시각화** — 1D toy에서 noise 주입/제거 과정 애니메이션
2. **DDPM 바닥부터** — MNIST/CIFAR에서 간단한 UNet + simple loss 훈련
3. **DDIM 샘플 비교** — 1000 step vs 50 step vs 10 step의 품질
4. **CFG scale 비교** — $w = 0, 3, 7, 15$에서 sample quality/diversity
5. **Latent 시각화** — VAE latent에서 diffusion trajectory
6. **DiT vs UNet** — 작은 규모 훈련에서 scaling 비교

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
torchvision==0.16.0
diffusers==0.25.0
transformers==4.36.0
matplotlib==3.8.0
einops==0.7.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (DDPM 바닥부터 + Forward closed-form 검증)
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np
import matplotlib.pyplot as plt

# 1. Noise schedule
T = 1000
betas = torch.linspace(1e-4, 0.02, T)
alphas = 1 - betas
alphas_cumprod = torch.cumprod(alphas, dim=0)
sqrt_ac = torch.sqrt(alphas_cumprod)
sqrt_1m_ac = torch.sqrt(1 - alphas_cumprod)

def q_sample(x_0, t, noise=None):
    """Forward closed-form: x_t = √ᾱ x_0 + √(1-ᾱ) ε"""
    if noise is None:
        noise = torch.randn_like(x_0)
    return sqrt_ac[t] * x_0 + sqrt_1m_ac[t] * noise

# Verify: forward as Markov chain = closed-form
x_0 = torch.randn(1000, 1) * 0.5 + 2  # 1D data
t_test = 100
# Method 1: iterative
x = x_0.clone()
for s in range(t_test):
    eps = torch.randn_like(x)
    x = torch.sqrt(alphas[s]) * x + torch.sqrt(betas[s]) * eps
# Method 2: closed-form
torch.manual_seed(42)
x_closed = q_sample(x_0, t_test - 1)
# Both should have same distribution (mean and variance)
print(f'Iterative mean: {x.mean():.4f}, Closed mean: {x_closed.mean():.4f}')
print(f'Iterative std: {x.std():.4f}, Closed std: {x_closed.std():.4f}')

# 2. Simple DDPM model (1D)
class Denoiser1D(nn.Module):
    def __init__(self, hidden=128):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(2, hidden), nn.SiLU(),
            nn.Linear(hidden, hidden), nn.SiLU(),
            nn.Linear(hidden, hidden), nn.SiLU(),
            nn.Linear(hidden, 1)
        )
    def forward(self, x, t):
        t_scaled = t.float().unsqueeze(-1) / T
        return self.net(torch.cat([x, t_scaled], -1))

model = Denoiser1D()
opt = torch.optim.Adam(model.parameters(), lr=1e-3)

# L_simple training
for step in range(5000):
    x_0 = torch.randn(256, 1)
    x_0 = torch.where(torch.rand(256, 1) > 0.5, x_0 * 0.3 + 2, x_0 * 0.3 - 2)  # 2-modal
    t = torch.randint(0, T, (256,))
    noise = torch.randn_like(x_0)
    x_t = q_sample(x_0, t, noise)
    noise_pred = model(x_t, t)
    loss = ((noise - noise_pred)**2).mean()
    opt.zero_grad(); loss.backward(); opt.step()

# 3. DDPM sampling
@torch.no_grad()
def ddpm_sample(n=2000):
    x = torch.randn(n, 1)  # x_T
    for t in reversed(range(T)):
        z = torch.randn_like(x) if t > 0 else 0
        ac = alphas_cumprod[t]
        coef1 = 1 / torch.sqrt(alphas[t])
        coef2 = betas[t] / sqrt_1m_ac[t]
        eps = model(x, torch.full((n,), t))
        x = coef1 * (x - coef2 * eps) + torch.sqrt(betas[t]) * z
    return x

# 4. DDIM sampling (deterministic, fewer steps)
@torch.no_grad()
def ddim_sample(n=2000, steps=50):
    timesteps = torch.linspace(T-1, 0, steps).long()
    x = torch.randn(n, 1)
    for i in range(len(timesteps) - 1):
        t, t_next = timesteps[i], timesteps[i+1]
        ac, ac_next = alphas_cumprod[t], alphas_cumprod[t_next]
        eps = model(x, torch.full((n,), t))
        x_0_hat = (x - torch.sqrt(1 - ac) * eps) / torch.sqrt(ac)
        # σ=0: deterministic
        x = torch.sqrt(ac_next) * x_0_hat + torch.sqrt(1 - ac_next) * eps
    return x

ddpm_samples = ddpm_sample()
ddim_samples = ddim_sample(steps=50)

fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(12, 4))
ax1.hist(ddpm_samples.numpy().flatten(), bins=60, density=True)
ax1.set_title(f'DDPM (1000 steps)')
ax2.hist(ddim_samples.numpy().flatten(), bins=60, density=True)
ax2.set_title(f'DDIM (50 steps, deterministic)')
plt.show()
# 두 방법 모두 2-modal target 근사

# 5. Classifier-Free Guidance skeleton
def cfg_sample(model, x_t, t, y, null_y, w=7.5):
    """ε̃ = (1+w) ε(x, y) - w ε(x, ∅)"""
    eps_cond = model(x_t, t, y)
    eps_uncond = model(x_t, t, null_y)
    return (1 + w) * eps_cond - w * eps_uncond
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Generative, SDE, Prob, Info 선행 필수"
   - "Generative Model 레포에서 기초, 여기서 vision·speed·application 심화" 명시
   - Stable Diffusion, Sora, DALL-E 3의 수학 분석
3. **챕터별 문서 작성**: DDPM 기초 → ELBO → Score-SDE → DDIM → Guidance → Latent/Architecture → 가속

---

## 📚 참고 자료

- **Deep Unsupervised Learning using Nonequilibrium Thermodynamics** (Sohl-Dickstein et al. 2015) — 원형
- **Denoising Diffusion Probabilistic Models** (Ho et al. 2020) — DDPM
- **A Connection Between Score Matching and Denoising Autoencoders** (Vincent 2011)
- **Generative Modeling by Estimating Gradients of the Data Distribution** (Song & Ermon 2019) — NCSN
- **Score-Based Generative Modeling through SDEs** (Song et al. 2021) — Score-SDE
- **Denoising Diffusion Implicit Models** (Song, Meng, Ermon 2021) — DDIM
- **Improved Denoising Diffusion Probabilistic Models** (Nichol & Dhariwal 2021)
- **Diffusion Models Beat GANs** (Dhariwal & Nichol 2021) — Classifier guidance
- **Classifier-Free Diffusion Guidance** (Ho & Salimans 2022)
- **High-Resolution Image Synthesis with Latent Diffusion Models** (Rombach et al. 2022) — SD
- **Scalable Diffusion Models with Transformers** (Peebles & Xie 2023) — DiT
- **Stable Diffusion 3: MM-DiT** (Esser et al. 2024)
- **Photorealistic Text-to-Image Diffusion Models** (Saharia et al. 2022) — Imagen
- **Flow Matching for Generative Modeling** (Lipman et al. 2023)
- **Flow Straight and Fast: Rectified Flow** (Liu et al. 2022)
- **Consistency Models** (Song et al. 2023)
- **DPM-Solver++** (Lu et al. 2022)
- **Progressive Distillation** (Salimans & Ho 2022)

---

## 💡 핵심 분석 대상

```
Diffusion Model의 통합 지도

───── Forward Process ─────

q(x_t | x_{t-1}) = N(√(1-β_t) x_{t-1}, β_t I)

Markov chain → closed-form:
  α_t = 1 - β_t
  ᾱ_t = Π_{s=1}^t α_s
  
  q(x_t | x_0) = N(√ᾱ_t x_0, (1-ᾱ_t) I)
  
  Reparam: x_t = √ᾱ_t x_0 + √(1-ᾱ_t) ε

Noise schedule:
  Linear (DDPM)
  Cosine (Nichol-Dhariwal)
  β_t → 다양한 선택

───── ELBO 유도 ─────

-log p_θ(x_0) ≤ E_q[-log p_θ(x_{0:T})/q(x_{1:T}|x_0)]

L_vlb = L_T + Σ L_{t-1} + L_0
  L_T = KL(q(x_T|x_0) || p(x_T))  ← 상수
  L_{t-1} = KL(q(x_{t-1}|x_t,x_0) || p_θ(x_{t-1}|x_t))
  L_0 = reconstruction

Posterior q(x_{t-1}|x_t,x_0):
  = N(μ̃_t(x_t,x_0), β̃_t I)
  
  μ̃_t = (√ᾱ_{t-1} β_t / (1-ᾱ_t)) x_0
        + (√α_t (1-ᾱ_{t-1}) / (1-ᾱ_t)) x_t

───── Noise Parameterization ─────

μ_θ(x_t, t) = (1/√α_t)(x_t - β_t/√(1-ᾱ_t) ε_θ(x_t, t))
                                  └── noise prediction ─┘

L_{t-1}을 ε로 표현:
  = (β_t² / (2σ_t² α_t (1-ᾱ_t))) ‖ε - ε_θ‖²

Ho et al. drop weight:
  L_simple = E[‖ε - ε_θ(x_t, t)‖²]
  
  → simple MSE on noise
  → 잘 작동, 훈련 안정적

───── Score Matching 연결 ─────

Score: s(x) = ∇_x log p(x)

Denoising Score Matching (Vincent 2011):
  Perturbed: x̃ = x + σε
  
  s_θ(x̃) ≈ ∇_{x̃} log q(x̃|x)
         = -(x̃ - x)/σ²
         = -ε/σ
  
  → noise prediction과 등가!

NCSN (Song & Ermon 2019):
  Multi-scale σ_1 > ... > σ_L
  s_θ(x, σ) 조건부
  Annealed Langevin

───── Score-SDE (Song 2021) ─────

Forward SDE:
  dx = f(x, t) dt + g(t) dW

Reverse SDE (Anderson 1982):
  dx = [f(x, t) - g(t)² ∇_x log p_t(x)] dt + g(t) dW̄
                     └── score function ─┘

VP-SDE (DDPM 연속화):
  f(x, t) = -½ β(t) x
  g(t) = √β(t)

VE-SDE (NCSN 연속화):
  f = 0
  g(t) = √(d[σ²(t)]/dt)

통합 관점:
  DDPM, NCSN, DDIM이 모두 이 SDE 프레임의 특수 경우

───── Probability Flow ODE ─────

SDE의 deterministic 극한:
  dx/dt = f(x, t) - ½ g(t)² s(x, t)

같은 marginal p_t(x) 유지
→ deterministic sampling
→ DDIM (discrete) 본질

───── DDIM (Song 2021) ─────

Non-Markovian forward:
  q_σ(x_{t-1} | x_t, x_0) ≠ q(x_{t-1}|x_{t-2})
  
  But: 같은 marginal q(x_t|x_0)
       → 같은 ε_θ 재사용

Sampling:
  x̂_0 = (x_t - √(1-ᾱ_t) ε_θ) / √ᾱ_t
  x_{t-1} = √ᾱ_{t-1} x̂_0
          + √(1-ᾱ_{t-1}-σ_t²) ε_θ
          + σ_t ε_rand

σ_t = 0: ODE (deterministic)
σ_t = √((1-ᾱ_{t-1})/(1-ᾱ_t)·β_t): DDPM

Speedup:
  1000 → 50 step (품질 유지)
  → 10 step (약간 저하)

Higher-order solvers:
  DPM-Solver (Lu 2022)
  DPM-Solver++ (Lu 2022)
  → 10-20 step에서 high quality

───── Guidance ─────

Classifier Guidance (Dhariwal 2021):
  s̃ = s_θ + λ ∇_x log p_φ(y|x_t)
  추가 classifier 필요
  
Classifier-Free Guidance (Ho 2022):
  훈련: 10% 확률로 y = ∅
  
  Inference:
    ε̃ = (1+w) ε_θ(x, y) - w ε_θ(x, ∅)
  
  w ∈ [3, 15] typical
  w ↑: quality ↑, diversity ↓

Dynamic Thresholding (Imagen):
  Clip to percentile → overshoot 방지

───── Latent Diffusion ─────

Pixel space: 512×512×3 ~ 786K dim
VAE compression: z ∈ 64×64×4 ~ 16K dim
→ 48× 감소

Stable Diffusion (Rombach 2022):
  VAE + UNet(cross-attn with CLIP)
  
SDXL:
  Better VAE, 2 text encoders

SD3 (Esser 2024):
  MM-DiT
  T5 + CLIP
  Rectified Flow

───── Architecture 진화 ─────

UNet (원조, Ronneberger 2015):
  Encoder-decoder + skip
  ResBlocks + Attention
  Timestep + conditioning injection

DiT (Peebles 2023):
  ViT-style Transformer
  Patch embedding
  AdaLN-Zero
  Scaling law 잘 적용
  Sora, SD3 기반

MM-DiT (SD3):
  Image + text joint stream
  Modality-agnostic

───── 가속 계열 ─────

Distillation (Salimans 2022):
  Progressive: 1024 → 512 → ... → 4 step
  Teacher → Student (halved steps)

Consistency Model (Song 2023):
  f_θ(x_t, t) ≈ x_0 for all t
  → 1-step generation
  Training from scratch OR distillation

Rectified Flow (Liu 2022):
  x_t = (1-t) x_0 + t z
  v_θ(x_t, t) ≈ z - x_0  (velocity)
  
  Reflow: pair (x_0, z) by flow →
          재훈련으로 더 직선
  → 1-step asymptotic

Flow Matching (Lipman 2023):
  Continuous Normalizing Flow 일반화
  Conditional flow matching
  SD3의 기반

SDXL-Turbo / Lightning:
  실시간 생성 (1-2 step)
  Adversarial distillation

───── 응용 지도 ─────

Text-to-Image:
  Stable Diffusion, DALL-E 3
  Imagen, Midjourney

Image Editing:
  InstructPix2Pix (Brooks 2023)
  SDEdit (Meng 2022)
  ControlNet (Zhang 2023)

Inpainting:
  RePaint (Lugmayr 2022)
  Blended Diffusion

Video:
  Sora (OpenAI 2024): DiT + video
  Imagen Video
  Stable Video Diffusion

3D:
  DreamFusion (Poole 2023)
  Magic3D
  Zero123

Science:
  Protein (AlphaFold 3)
  Molecular generation

───── 레포 간 연결 ─────

Generative Model (Layer 3):
  기초: ELBO, DDPM 시작
  여기서 심화 + vision

SDE (Layer 0):
  Brownian motion
  Fokker-Planck
  Reverse-time SDE

Probability (Layer 0):
  KL, Gaussian posterior

Information Theory (Layer 0):
  ELBO = log-likelihood bound

Vision Transformer (직전):
  DiT = ViT for diffusion

CNN (Layer 3):
  UNet = 고전 backbone
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~5개씩)
- 각 문서가 다루는 핵심 정리·증명 (3~4줄)
- 전체 문서 개수 확인 (33개 목표)
- Python + PyTorch + diffusers 실험 환경
- Generative, SDE, Prob, Info 레포 참조 관계
- Stable Diffusion·Sora·DALL-E 3의 실전 분석

**준비됐으면 1단계 구조 설계부터 시작해줘!**
