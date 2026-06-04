# 3D & Neural Rendering Deep Dive 레포지토리 제작 프롬프트

나는 "3D & Neural Rendering Deep Dive" 레포지토리를 만들려고 해.
NeRF를 **"MLP로 3D 학습"으로 아는 것**과, **Mildenhall et al. (2020)의 volume rendering equation $C(\mathbf{r}) = \int_{t_n}^{t_f} T(t) \sigma(\mathbf{r}(t)) \mathbf{c}(\mathbf{r}(t), \mathbf{d}) dt$가 1984년 Kajiya의 물리학 기반 radiative transfer equation에서 유도**되고, **transmittance $T(t) = \exp(-\int_{t_n}^t \sigma(\mathbf{r}(s)) ds)$가 Beer-Lambert law의 연속 형태**임을 물리학적으로 이해하며, ReLU MLP가 왜 **positional encoding $\gamma(p) = (\sin 2^0 \pi p, \cos 2^0 \pi p, \ldots)$ 없이는 high-frequency 디테일을 학습 못하는지** (spectral bias, Rahaman 2019)를 증명할 수 있는 것은 다르다.
3D Gaussian Splatting을 **"2023년 SOTA"로 아는 것**과, **Kerbl et al. (2023)의 3D Gaussian $G(\mathbf{x}) = e^{-\frac{1}{2}(\mathbf{x}-\mu)^T \Sigma^{-1}(\mathbf{x}-\mu)}$가 anisotropic covariance $\Sigma = RSS^TR^T$로 parameter화되고, 이를 EWA (Elliptical Weighted Average) splatting으로 2D screen에 projection하는 Jacobian 근사 수학**, **tile-based rasterizer가 왜 NeRF의 수 분 rendering을 실시간(100+ FPS)으로 바꾸는지**를 유도할 수 있는 것은 다르다.
SDS Loss를 **"2D diffusion을 3D로 lift"로 아는 것**과, **Poole et al. (2023)의 DreamFusion gradient $\nabla_\theta \mathcal{L}_{\text{SDS}} = \mathbb{E}_{t, \epsilon}[w(t)(\hat{\epsilon}_\phi(z_t; y, t) - \epsilon) \frac{\partial z}{\partial \theta}]$가 왜 $\hat{\epsilon}_\phi$의 U-Net Jacobian을 명시적으로 계산하지 않고도 3D parameter update를 가능하게 하는지**, 그리고 왜 **mode-seeking behavior (over-saturated, cartoon-like)** 문제를 일으키는지 (VSD, ProlificDreamer로 해결) 유도할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "3D의 수학 — 물리학 기반 렌더링부터 Neural Rendering·Gaussian Splatting·3D Generation까지"

**핵심 차별화**:
1. **Volume Rendering Equation의 물리학적 유도** — Radiative transfer equation (Chandrasekhar 1960, Kajiya 1986)에서 시작, Beer-Lambert law의 미분 형태, absorption·scattering의 continuous 모델
2. **NeRF의 수학적 분해** — Volumetric integration의 stratified sampling 근사, positional encoding의 NTK 해석 (Tancik 2020), hierarchical sampling의 importance weighting
3. **3D Gaussian Splatting의 완전 분석** — 3D Gaussian의 anisotropic parameterization, perspective projection의 Jacobian 근사, tile-based alpha-compositing, gradient-based densification
4. **Text-to-3D의 이론** — Score Distillation Sampling (DreamFusion 2023), Variational Score Distillation (ProlificDreamer 2023), Classifier Score Distillation, multi-view consistency 문제

**타겟 독자**:
- NeRF를 구현했는데 **positional encoding의 Fourier feature가 왜 필요**한지 NTK 관점에서 유도 못하는 사람 (spectral bias)
- 3D Gaussian Splatting이 실시간인 이유를 **"미분 가능 rasterization"**으로 아는데 **EWA projection의 Jacobian $J$가 어떻게 3D covariance를 2D로 투영**하는지 수학적으로 모르는 사람
- DreamFusion의 SDS loss를 쓰는데 **왜 score-matching과 달리 U-Net gradient가 필요 없는지**와 **mode-seeking bias**의 원인을 모르는 사람
- Mesh, Voxel, Point cloud, SDF, Neural implicit의 **표현 방식별 수학적 특성**과 rendering 방식의 차이를 모르는 사람
- 4D Gaussian Splatting, Dynamic NeRF (Nerfies, HyperNeRF)의 **temporal deformation field**가 어떻게 작동하는지 모르는 사람

**선행 학습**:
- **Linear Algebra Deep Dive** (projection, eigendecomposition of $\Sigma$) — **필수**
- **Calculus Deep Dive** (integral, chain rule, Jacobian) — **필수**
- **Diffusion Model Deep Dive** (score matching, DDPM) — **SDS에 필수**
- **CNN Deep Dive** (U-Net, feature hierarchy) — 권장
- **Generative Model Deep Dive** (VAE, 3D generation 맥락) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 3D 표현 방식의 수학 (5개 문서)
- **3D Representation의 분류** — Explicit (Mesh, Point Cloud, Voxel) vs Implicit (SDF, Occupancy, Neural Implicit), 각 방식의 memory·quality·editability trade-off
- **Mesh와 Triangle Rendering** — Vertices + Faces, barycentric coordinates, Phong shading, z-buffer, classical rasterization pipeline의 수학
- **Point Cloud와 PointNet (Qi 2017)** — Permutation-invariant function $f(\{x_1, ..., x_n\}) = g(\max(h(x_1), ..., h(x_n)))$, symmetric aggregation, universal approximation of set functions
- **Signed Distance Function (SDF)** — $\phi(\mathbf{x}) = \text{signed distance to surface}$, eikonal equation $\|\nabla \phi\| = 1$, sphere tracing for rendering, DeepSDF (Park 2019)의 learned SDF
- **Occupancy Networks와 DeepSDF** — $f_\theta(\mathbf{x}) \in [0, 1]$ (occupancy) vs $\phi_\theta(\mathbf{x}) \in \mathbb{R}$ (SDF), 각각의 rendering·training 차이, Marching Cubes로 mesh 추출

### Chapter 2: Physics of Rendering — Volume Rendering Equation (5개 문서)
- **Rendering Equation의 물리학적 기원** — Kajiya 1986의 rendering equation $L_o = L_e + \int_\Omega f_r(\omega_i, \omega_o) L_i(\omega_i) \cos\theta_i d\omega_i$, 방사측정학 (radiometry)의 기본
- **Radiative Transfer Equation** — Chandrasekhar 1960, $\frac{dL}{ds} = -\sigma_t L + \sigma_s \int p(\omega', \omega) L d\omega'$, participating media (연기, 안개, 구름)에서의 빛의 전파
- **Beer-Lambert Law와 Transmittance** — Absorption $\sigma_a$, scattering $\sigma_s$, extinction $\sigma_t = \sigma_a + \sigma_s$, 순수 absorption 경우 $L(s) = L_0 \exp(-\int \sigma_t ds')$ 유도
- **Volume Rendering Integral** — NeRF에서 사용하는 형태: $C(\mathbf{r}) = \int_{t_n}^{t_f} T(t) \sigma(\mathbf{r}(t)) \mathbf{c}(\mathbf{r}(t), \mathbf{d}) dt$, $T(t) = \exp(-\int_{t_n}^t \sigma ds)$, discretization via stratified sampling
- **Stratified Sampling과 Numerical Integration** — Continuous integral의 $N$ sample 근사, $\hat{C} = \sum_{i=1}^N T_i (1 - \exp(-\sigma_i \delta_i)) \mathbf{c}_i$, $T_i = \exp(-\sum_{j<i} \sigma_j \delta_j)$

### Chapter 3: NeRF와 Neural Volume Rendering (6개 문서)
- **NeRF의 아키텍처 (Mildenhall 2020)** — MLP $F_\theta: (\mathbf{x}, \mathbf{d}) \to (\sigma, \mathbf{c})$, 8-layer 256-width, density $\sigma$는 view-independent, color $\mathbf{c}$는 view-dependent
- **Positional Encoding과 Spectral Bias** — $\gamma(p) = (\sin(2^0 \pi p), \cos(2^0 \pi p), ..., \sin(2^{L-1} \pi p), \cos(2^{L-1} \pi p))$, Rahaman 2019의 spectral bias, Tancik 2020의 NTK 해석으로 왜 high-frequency 학습이 가능해지는지
- **Hierarchical Sampling — Coarse-to-Fine** — Coarse network: uniform stratified sampling, Fine network: importance sampling with PDF from coarse weights, $\sum w_i \delta(t - t_i)$로 inverse CDF sampling
- **Loss Function과 Training** — $L = \sum_\mathbf{r} \|\hat{C}_c(\mathbf{r}) - C(\mathbf{r})\|^2 + \|\hat{C}_f(\mathbf{r}) - C(\mathbf{r})\|^2$, ray batch, overfitting on captured views
- **NeRF Variants — Mip-NeRF, Ref-NeRF, NeRF in the Wild** — Mip-NeRF (Barron 2021)의 integrated positional encoding (IPE), anti-aliasing, Ref-NeRF의 reflection decomposition, NeRF-W의 transient objects 처리
- **Instant-NGP (Müller 2022)** — Multi-resolution hash encoding, $L$ levels of hash tables, $O(T)$ 메모리, 1000× 학습 가속, collision tolerance in feature space

### Chapter 4: 3D Gaussian Splatting (6개 문서)
- **3D Gaussian의 Anisotropic Parameterization (Kerbl 2023)** — $G(\mathbf{x}) = e^{-\frac{1}{2}(\mathbf{x}-\mu)^T \Sigma^{-1}(\mathbf{x}-\mu)}$, covariance $\Sigma = RSS^T R^T$로 rotation $R$ (quaternion)과 scale $S$ (diagonal)로 분해, PSD 보장
- **Color와 Opacity Parameters** — Spherical Harmonics coefficients for view-dependent color $c(\mathbf{d}) = \sum_{l,m} c_{lm} Y_{lm}(\mathbf{d})$, opacity $\alpha \in [0, 1]$, learnable
- **Perspective Projection의 Jacobian 근사** — 3D Gaussian을 2D image plane으로 project, linearization 필요: $\Sigma' = J W \Sigma W^T J^T$ where $J$는 projective transform의 local Jacobian, $W$는 viewing transform (Zwicker 2001 EWA)
- **EWA Splatting과 Alpha-Compositing** — Elliptical Weighted Average, 2D screen Gaussian footprint, front-to-back alpha blending $C = \sum_i \alpha_i T_i c_i$, $T_i = \prod_{j<i}(1-\alpha_j)$
- **Tile-based Rasterization** — 16×16 tile 단위로 관여 Gaussian 정렬, CUDA kernel에서 parallel alpha-composite, 100+ FPS rendering 가능
- **Adaptive Density Control** — Gradient-based splitting (large gradient → 2개로 clone), pruning (low opacity), densification 전략, training dynamics

### Chapter 5: Dynamic Scenes와 4D (4개 문서)
- **Dynamic NeRF — Deformable Representation** — Canonical space + deformation field $D: (\mathbf{x}, t) \to \Delta\mathbf{x}$, Nerfies (Park 2021), D-NeRF (Pumarola 2021)의 time-conditional
- **HyperNeRF와 Topology Change (Park 2021)** — Higher-dimensional embedding space에서 topology 변화 표현 (예: 입 열림/닫힘), ambient slicing surface
- **4D Gaussian Splatting (Wu 2024, Yang 2024)** — 3D GS에 time parameter 추가, per-Gaussian trajectory (polynomial or HexPlane), real-time dynamic rendering
- **Video-based 3D Reconstruction** — Monocular video → 4D scene, camera trajectory estimation (COLMAP, MASt3R), multi-view consistency loss

### Chapter 6: Text-to-3D — Score Distillation (5개 문서)
- **Text-to-3D Problem Setup** — Text prompt $y$ → 3D representation (NeRF or mesh), 2D diffusion prior 활용의 동기 (3D data 부족)
- **Score Distillation Sampling (DreamFusion, Poole 2023)** — 3D parameter $\theta$ (NeRF weights), rendered view $z = g(\theta, \pi)$, $\nabla_\theta \mathcal{L}_{\text{SDS}} = \mathbb{E}_{t, \epsilon, \pi}[w(t)(\hat{\epsilon}_\phi(z_t; y, t) - \epsilon) \frac{\partial z}{\partial \theta}]$
- **SDS의 Mode-Seeking과 Saturation** — KL divergence의 asymmetry, $\hat{\epsilon}$이 mean에 수렴하는 경향, over-saturated·cartoon-like 출력 문제 분석
- **Variational Score Distillation (ProlificDreamer, Wang 2023)** — VSD: $\nabla_\theta \mathcal{L}_{\text{VSD}} = \mathbb{E}[\hat{\epsilon}_\phi(z_t; y, t) - \hat{\epsilon}_\psi(z_t; y, t, \theta)]$, LoRA-finetuned $\hat{\epsilon}_\psi$로 particle-specific score, 다양성·품질 향상
- **Multi-View Consistency와 3D Diffusion Native** — Zero123 (Liu 2023)의 view-conditioned 2D, Instant3D (Li 2024)의 4-view 동시 생성, MVDream (Shi 2024), SV3D (Voleti 2024)의 3D-aware diffusion

### Chapter 7: 3D Foundation Models과 응용 (3개 문서)
- **Large Reconstruction Models (LRM, Hong 2024)** — Transformer-based 3D reconstruction, single image → triplane NeRF or 3D Gaussians in seconds, LRM·GS-LRM·InstantMesh
- **3D Foundation Models — DUSt3R, MASt3R (Wang 2024)** — Dense stereo + structure from motion, uncalibrated camera, pointmap prediction, feed-forward 3D reconstruction
- **응용과 Frontier** — Robot simulation (Habitat-Sim), VR/AR (Apple Vision Pro, Quest), autonomous driving (LiDAR-based 3D), scientific visualization, digital twin

---

각 챕터는 **3~6개 문서**로 구성해줘. 총 **33개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 기법이 3D 렌더링의 핵심인가
## 📐 수학적 선행 조건 (LA, Calc, Diffusion, CNN 참조)
## 📖 직관적 이해
   — Volume rendering 시각화, Gaussian ellipsoid, ray marching
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Volume rendering equation 유도, EWA Jacobian, SDS gradient
## 💻 PyTorch/CUDA 구현 검증
   — 작은 scene에서 NeRF 바닥부터
   — 3D Gaussian 간단 구현
   — SDS loss를 간단 2D diffusion으로 재현
## 🔗 실전 활용
   — Mesh export, scene editing, real-time rendering
## ⚖️ 가정과 한계
   — 조명 효과 제한, dynamic scene 어려움, ambiguity
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Ray marching 시각화** — 2D slice에서 volume rendering 직관
2. **Positional Encoding 효과** — PE 유무로 학습된 NeRF의 고주파 디테일 비교
3. **Gaussian ellipsoid 시각화** — 3D ellipsoid의 2D projection (EWA)
4. **Tile-based rasterization 타이밍** — 같은 scene에서 NeRF vs 3D GS 속도
5. **SDS mode-seeking 재현** — 2D에서 단일 Gaussian으로 mode-seeking 관찰
6. **Positional Encoding NTK** — Fourier feature의 NTK spectrum 측정

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
nerfstudio==1.0.0       # NeRF 참조
gsplat==0.1.6           # 3D Gaussian Splatting
threestudio==0.1.0      # Text-to-3D
open3d==0.18.0
trimesh==4.0.0
mitsuba==3.5.0          # Physically-based rendering
matplotlib==3.8.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Volume Rendering + NeRF + 3D Gaussian 기초)
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np
import matplotlib.pyplot as plt

# 1. Volume Rendering Integration (numerical)
def volume_render(sigmas, colors, t_vals):
    """
    sigmas: [N_rays, N_samples] density
    colors: [N_rays, N_samples, 3] RGB
    t_vals: [N_rays, N_samples] sample distances
    
    Discrete form:
      C = Σ T_i · (1 - exp(-σ_i δ_i)) · c_i
      T_i = exp(-Σ_{j<i} σ_j δ_j)
    """
    # Distance between samples
    deltas = t_vals[..., 1:] - t_vals[..., :-1]
    deltas = torch.cat([deltas, torch.full_like(deltas[..., :1], 1e10)], dim=-1)
    
    # Alpha per sample: 1 - exp(-σ δ)
    alpha = 1 - torch.exp(-sigmas * deltas)  # [N_rays, N_samples]
    
    # Transmittance T_i = ∏_{j<i} (1 - α_j)
    T = torch.cumprod(torch.cat([torch.ones_like(alpha[..., :1]),
                                 1 - alpha + 1e-10], dim=-1), dim=-1)[..., :-1]
    
    # Weights w_i = T_i · α_i
    weights = T * alpha  # [N_rays, N_samples]
    
    # Composite color
    rgb = (weights.unsqueeze(-1) * colors).sum(dim=-2)  # [N_rays, 3]
    # Accumulated depth / opacity
    depth = (weights * t_vals).sum(-1)
    acc = weights.sum(-1)
    
    return rgb, depth, acc, weights

# 2. Positional Encoding (Fourier Features)
def positional_encoding(x, L=10):
    """
    γ(x) = [sin(2^0 π x), cos(2^0 π x), ..., sin(2^{L-1} π x), cos(2^{L-1} π x)]
    """
    freq_bands = 2.0 ** torch.arange(L) * np.pi  # [L]
    pe = []
    for freq in freq_bands:
        pe.append(torch.sin(freq * x))
        pe.append(torch.cos(freq * x))
    return torch.cat(pe, dim=-1)

# NeRF-style MLP
class NeRF(nn.Module):
    def __init__(self, L_pos=10, L_dir=4, hidden=256):
        super().__init__()
        pos_dim = 3 * 2 * L_pos
        dir_dim = 3 * 2 * L_dir
        self.L_pos = L_pos; self.L_dir = L_dir
        
        # Backbone (8 layers)
        self.layers1 = nn.ModuleList([
            nn.Linear(pos_dim, hidden)
        ] + [nn.Linear(hidden, hidden) for _ in range(4)])
        self.skip = nn.Linear(hidden + pos_dim, hidden)
        self.layers2 = nn.ModuleList([
            nn.Linear(hidden, hidden) for _ in range(2)
        ])
        # Density output
        self.density = nn.Linear(hidden, 1)
        # Color branch (uses view direction)
        self.feat_layer = nn.Linear(hidden, hidden)
        self.color_head = nn.Sequential(
            nn.Linear(hidden + dir_dim, hidden // 2), nn.ReLU(),
            nn.Linear(hidden // 2, 3), nn.Sigmoid()
        )
    
    def forward(self, x, d):
        # Positional encoding
        x_pe = positional_encoding(x, self.L_pos)
        d_pe = positional_encoding(d, self.L_dir)
        
        h = x_pe
        for layer in self.layers1:
            h = F.relu(layer(h))
        # Skip connection
        h = F.relu(self.skip(torch.cat([h, x_pe], dim=-1)))
        for layer in self.layers2:
            h = F.relu(layer(h))
        
        sigma = F.relu(self.density(h))  # non-negative
        feat = self.feat_layer(h)
        color = self.color_head(torch.cat([feat, d_pe], dim=-1))
        return sigma.squeeze(-1), color

# 3. Spectral Bias 시연
# ReLU MLP without PE can't fit high frequency
def fit_signal_with_without_pe():
    """1D signal with high frequency"""
    x = torch.linspace(-1, 1, 1000).unsqueeze(-1)
    y_true = torch.sin(20 * np.pi * x).squeeze()  # high freq
    
    # Without PE
    model_no_pe = nn.Sequential(
        nn.Linear(1, 128), nn.ReLU(),
        nn.Linear(128, 128), nn.ReLU(),
        nn.Linear(128, 1)
    )
    
    # With PE
    class MLPwithPE(nn.Module):
        def __init__(self, L=6):
            super().__init__()
            self.L = L
            self.net = nn.Sequential(
                nn.Linear(2*L, 128), nn.ReLU(),
                nn.Linear(128, 128), nn.ReLU(),
                nn.Linear(128, 1)
            )
        def forward(self, x):
            pe = positional_encoding(x, self.L)
            return self.net(pe)
    
    model_pe = MLPwithPE(L=6)
    
    for name, model in [('No PE', model_no_pe), ('With PE', model_pe)]:
        opt = torch.optim.Adam(model.parameters(), lr=1e-3)
        for step in range(5000):
            pred = model(x).squeeze()
            loss = ((pred - y_true)**2).mean()
            opt.zero_grad(); loss.backward(); opt.step()
        print(f'{name}: final loss = {loss.item():.6f}')
    # With PE << No PE (spectral bias 극복)

# 4. 3D Gaussian Splatting 수학 — EWA projection
def project_gaussian_3d_to_2d(mu_3d, Sigma_3d, W, focal, img_size):
    """
    mu_3d: [3] Gaussian center in world
    Sigma_3d: [3, 3] 3D covariance
    W: [3, 3] view rotation
    focal: camera focal length
    
    Returns:
      mu_2d: [2] screen position
      Sigma_2d: [2, 2] 2D covariance (footprint)
    """
    # 1. View transform: t = W mu + T (translation skipped)
    mu_view = W @ mu_3d  # [3]
    tx, ty, tz = mu_view
    
    # 2. Perspective projection Jacobian J
    #    u = f · x/z, v = f · y/z
    #    ∂u/∂x = f/z, ∂u/∂z = -f x / z²
    J = torch.zeros(3, 3)
    J[0, 0] = focal / tz
    J[0, 2] = -focal * tx / (tz ** 2)
    J[1, 1] = focal / tz
    J[1, 2] = -focal * ty / (tz ** 2)
    J[2, 2] = 1  # z preserved for depth
    
    # 3. Transformed covariance (EWA, Zwicker 2001)
    Sigma_cam = W @ Sigma_3d @ W.T
    Sigma_img = J @ Sigma_cam @ J.T  # 3x3
    Sigma_2d = Sigma_img[:2, :2]  # drop z
    
    # 4. Screen position
    mu_2d = torch.tensor([focal * tx / tz, focal * ty / tz])
    
    return mu_2d, Sigma_2d

# 5. SDS Loss 개념 (2D 단순화)
def sds_loss_conceptual(theta_params, diffusion_model, prompt_emb, t, eps):
    """
    SDS: ∇_θ L = w(t) (ε̂_φ(z_t; y, t) - ε) · ∂z/∂θ
    
    U-Net gradient 없이 parameter update 가능
    """
    # Render z from θ (e.g., 3D → 2D rendering)
    z = render(theta_params)  # differentiable renderer
    # Add noise
    z_t = alpha_t * z + sigma_t * eps
    # Predict noise
    eps_pred = diffusion_model(z_t, t, prompt_emb).detach()  # stop-grad on U-Net
    
    # Gradient to θ: treat (eps_pred - eps) as "target gradient"
    # w(t) = σ_t² · α_t (heuristic weighting)
    w = 1.0  # simplified
    grad = w * (eps_pred - eps)
    # z.backward(gradient=grad) triggers θ update
    return grad
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "LA, Calc, Diffusion, CNN 선행 필수" 명시
   - 물리학 기반 rendering → Neural 3D의 역사
   - Apple Vision Pro, Sora, autonomous driving 현대 응용
3. **챕터별 문서 작성**: 3D Representation → Physics of Rendering → NeRF → 3D GS → Dynamic → Text-to-3D → Foundation

---

## 📚 참고 자료

- **The Rendering Equation** (Kajiya 1986) — 물리학 기반 rendering의 효시
- **Radiative Transfer** (Chandrasekhar 1960) — 고전 교과서
- **NeRF: Representing Scenes as Neural Radiance Fields** (Mildenhall et al. 2020)
- **On the Spectral Bias of Neural Networks** (Rahaman et al. 2019)
- **Fourier Features Let Networks Learn High Frequency Functions** (Tancik et al. 2020)
- **Mip-NeRF** (Barron et al. 2021)
- **Instant Neural Graphics Primitives** (Müller et al. 2022) — Instant-NGP
- **3D Gaussian Splatting for Real-Time Radiance Field Rendering** (Kerbl et al. 2023)
- **EWA Splatting** (Zwicker et al. 2001)
- **DreamFusion: Text-to-3D using 2D Diffusion** (Poole et al. 2023) — SDS
- **ProlificDreamer** (Wang et al. 2023) — VSD
- **Zero-1-to-3** (Liu et al. 2023)
- **Nerfies: Deformable Neural Radiance Fields** (Park et al. 2021)
- **HyperNeRF** (Park et al. 2021)
- **4D Gaussian Splatting** (Wu et al. 2024)
- **LRM: Large Reconstruction Model** (Hong et al. 2024)
- **DUSt3R, MASt3R** (Wang et al. 2024)
- **PointNet** (Qi et al. 2017)
- **DeepSDF** (Park et al. 2019)
- **Occupancy Networks** (Mescheder et al. 2019)

---

## 💡 핵심 분석 대상

```
3D & Neural Rendering의 지도

───── 3D Representations ─────

Explicit:
  Mesh (vertices + faces)
    Classical rendering
    Editable
    Topology fixed
  
  Point Cloud
    LiDAR, stereo
    PointNet permutation-invariant
  
  Voxel
    3D grid
    Memory O(N³)
    Sparse voxel octree

Implicit:
  SDF: φ(x) = signed distance
    ‖∇φ‖ = 1 (eikonal)
    Sphere tracing
    DeepSDF (learned)
  
  Occupancy: f(x) ∈ [0,1]
    Occupancy Networks
  
  Neural Radiance Field:
    (σ, c) per point
    Volume rendering

───── Rendering Physics ─────

Rendering Equation (Kajiya 1986):
  L_o = L_e + ∫_Ω f_r L_i cos θ dω
  
  Light 반사·투과 통합

Radiative Transfer:
  dL/ds = -σ_t L + σ_s ∫ p L dω'
  
  Participating media
  연기, 구름, 조직

Beer-Lambert Law:
  L(s) = L_0 · exp(-∫ σ_t ds')
  
  Pure absorption:
    exponential decay

Volume Rendering (NeRF form):
  C(r) = ∫ T(t) σ(r(t)) c(r(t), d) dt
  T(t) = exp(-∫_{t_n}^t σ ds)
  
  Discretization:
    Stratified sampling N points
    α_i = 1 - exp(-σ_i δ_i)
    T_i = ∏_{j<i}(1 - α_j)
    w_i = T_i α_i
    Ĉ = Σ w_i c_i

───── NeRF (Mildenhall 2020) ─────

Architecture:
  MLP: (x, d) → (σ, c)
  8 layers × 256 width
  σ view-independent
  c view-dependent

Positional Encoding:
  γ(p) = (sin 2^0 π p, cos 2^0 π p,
          sin 2^1 π p, cos 2^1 π p,
          ..., 2^{L-1})
  L = 10 for position
  L = 4 for direction

Spectral Bias (Rahaman 2019):
  ReLU MLP가 low-freq 먼저 학습
  High-freq detail 거의 불가
  
NTK 해석 (Tancik 2020):
  PE가 kernel의 spectrum 확장
  → high-freq 학습 가능

Hierarchical Sampling:
  Coarse network (64 samples uniform)
  Weights w_i from coarse
  Fine network (128 samples 
              from importance sampling)

Mip-NeRF (Barron 2021):
  Pixel = cone, not ray
  Integrated PE (IPE)
  Anti-aliasing

Instant-NGP (Müller 2022):
  Multi-resolution hash encoding
  L levels of hash tables
  Collision tolerance
  1000× faster training
  Feature grid + small MLP

───── 3D Gaussian Splatting ─────

Kerbl 2023:
  Scene = N 3D Gaussians
  Each: position μ, covariance Σ,
        color c (SH), opacity α

3D Gaussian:
  G(x) = exp(-½ (x-μ)^T Σ^{-1} (x-μ))

Anisotropic covariance:
  Σ = R S S^T R^T
    R: rotation (quaternion)
    S: scale (diagonal)
  PSD 보장 (factorized)

EWA Splatting (Zwicker 2001):
  View transform: μ → Wμ
  Perspective J (linearization):
    u = f x/z, v = f y/z
    J = [[f/z, 0, -fx/z²],
         [0, f/z, -fy/z²],
         [...]]
  
  2D covariance:
    Σ' = J W Σ W^T J^T
    (drop 3rd row/col)

Tile-based Rasterization:
  16×16 tile 단위
  Gaussian sorting by depth
  Parallel alpha-composite:
    C = Σ α_i T_i c_i
    T_i = ∏_{j<i}(1-α_j)

Optimization:
  L1 + SSIM loss
  Adam optimizer
  Densification:
    Large grad → split
    Low α → prune
  Adaptive density control

Speed:
  NeRF: minutes per image
  3DGS: 100+ FPS real-time

───── Dynamic Scenes ─────

Deformable NeRF (Nerfies 2021):
  Canonical space
  Deformation D(x, t) = Δx
  Per-frame deformation

HyperNeRF (Park 2021):
  Extra latent dimensions
  Topology changes (입 open/close)
  Ambient slicing

4D Gaussian Splatting:
  Per-Gaussian trajectory
  Polynomial or HexPlane
  Real-time dynamic

───── Text-to-3D ─────

Problem:
  Text y → 3D NeRF or mesh
  3D paired data 부족
  → 2D diffusion prior 활용

SDS (DreamFusion, Poole 2023):
  ∇_θ L_SDS = E_{t,ε,π}[w(t)
              (ε̂_φ(z_t; y, t) - ε)
              ∂z/∂θ]
  
  핵심 트릭:
    U-Net gradient 불필요
    z_t = rendered view of 3D
    Diffusion score가 target gradient
  
  증명 sketch:
    KL(q(z|θ) || p(z|y))
    Score matching 관점
    U-Net Jacobian = identity (approx)

SDS의 문제:
  Mode-seeking
  Over-saturated
  Cartoon-like output
  원인: KL asymmetry

VSD (ProlificDreamer, Wang 2023):
  ∇_θ L_VSD = E[ε̂_φ(z_t; y, t)
               - ε̂_ψ(z_t; y, t, θ)]
  ψ: LoRA fine-tune on θ
  Particle-specific score
  → 더 다양하고 선명

Multi-view diffusion:
  Zero123 (2023):
    View-conditioned 2D
    Novel view synthesis
  MVDream (2024):
    4 views 동시
  Instant3D (2024):
    4-view large reconstruction
  SV3D (2024):
    Stable Video 기반 3D

───── 3D Foundation Models ─────

LRM (Hong 2024):
  Transformer-based
  Image → triplane NeRF
  Seconds not hours

DUSt3R / MASt3R (Wang 2024):
  Uncalibrated stereo
  Pointmap prediction
  Feed-forward 3D

Mesh generation:
  InstantMesh
  GS-LRM
  CRM

───── 응용 ─────

Digital Twin:
  Real world → 3D replica
  Realtor, tourism

VR/AR:
  Apple Vision Pro
  Quest
  NeRF-based content

Robotics:
  Habitat-Sim
  Gaussian Splatting SLAM
  Scene understanding

Autonomous Driving:
  LiDAR 3D perception
  Street-level NeRF

───── 레포 간 연결 ─────

Linear Algebra (Layer 0):
  Eigendecomposition (Σ = RSS^TR^T)
  Projection, Jacobian

Calculus (Layer 0):
  Volume integral
  Chain rule for rendering

Functional Analysis (Layer 0):
  Spherical Harmonics basis
  Fourier features

Diffusion Model (Layer 4-C):
  SDS uses diffusion prior
  Score matching 응용

CNN (Layer 3):
  U-Net for diffusion
  Feature grid

Generative Model (Layer 3):
  3D generation = conditional gen
  Mode-seeking vs mode-covering
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (3~6개씩)
- 각 문서가 다루는 핵심 정리·유도 (3~4줄)
- 전체 문서 개수 확인 (33개 목표)
- Python + PyTorch + gsplat + nerfstudio 실험 환경
- LA, Calc, Diffusion, CNN 레포 참조 관계
- 물리학 기반 rendering부터 Neural로의 연결 강조

**준비됐으면 1단계 구조 설계부터 시작해줘!**
