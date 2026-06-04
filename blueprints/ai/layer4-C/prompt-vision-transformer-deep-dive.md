# Vision Transformer Deep Dive 레포지토리 제작 프롬프트

나는 "Vision Transformer Deep Dive" 레포지토리를 만들려고 해.
ViT의 patch embedding을 **"이미지를 16x16 토큰으로"로 아는 것**과, **Dosovitskiy et al. (2021)이 $x \in \mathbb{R}^{H \times W \times C}$를 $N = HW/P^2$ 개의 $P \times P \times C$ 패치 $\{x_p^i\}$로 reshape하고 linear projection $E \in \mathbb{R}^{P^2 C \times D}$로 임베딩하는 수학**과, 이것이 **patch size $P \times P$의 convolution with stride $P$와 수학적으로 동등**함을 증명할 수 있는 것은 다르다.
DINO를 **"self-supervised ViT"로 아는 것**과, **Caron et al. (2021)의 self-distillation에서 student $s_\theta$가 teacher $t_{\theta'}$을 모방하되 teacher는 $\theta' = \tau \theta' + (1-\tau) \theta$ EMA로 업데이트**되고 **centering + sharpening으로 mode collapse를 방지**하는 수학을 유도할 수 있는 것은 다르다.
MAE를 **"masked autoencoder"로 아는 것**과, **He et al. (2022)의 75% mask ratio가 왜 BERT의 15%보다 훨씬 큰지 (이미지의 pixel redundancy)**, **asymmetric encoder-decoder에서 encoder가 visible patch만 처리해 속도 이득**을 얻는 정보이론적 해석을 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "비전 분야의 Transformer — ViT부터 SSL·Hierarchical·Multimodal까지"

**핵심 차별화**:
1. **ViT 수식 완전 분해** — Patch embedding이 conv와 등가, CLS token의 pooling 해석, position embedding의 역할
2. **Self-Supervised Vision의 3대 패러다임** — Contrastive (SimCLR, MoCo), Self-distillation (DINO, iBOT), Masked Image Modeling (MAE, BEiT)
3. **Hierarchical ViT** — Swin (shifted window), PVT (pyramid), 계층적 feature의 이론적 정당성
4. **Multimodal Vision** — CLIP의 contrastive, BLIP의 caption, Flamingo·LLaVA의 vision-language fusion

**타겟 독자**:
- ViT를 쓰지만 **CNN 대비 inductive bias 부족**을 scale이나 augmentation으로 어떻게 보상하는지 (DeiT, Touvron 2021) 모르는 사람
- CLIP의 contrastive loss $L = -\log \frac{\exp(s_{ii}/\tau)}{\sum_j \exp(s_{ij}/\tau)}$를 쓰지만 **temperature $\tau$의 역할**과 symmetric text/image loss의 이유를 모르는 사람
- MAE의 75% masking이 무슨 근거인지, **SimMIM의 구조적 차이**를 모르는 사람
- DINOv2 (Oquab 2024)가 DINO + iBOT + multi-objective의 집합인데 **각각의 기여도**를 이해 못하는 사람
- Swin Transformer의 **shifted window attention이 어떻게 global receptive field**를 얻는지 유도 못하는 사람

**선행 학습**:
- **Transformer Deep Dive** (self-attention, PE) — **필수**
- **CNN Deep Dive** (convolution, RF, inductive bias) — **필수**
- **Generative Model Deep Dive** (MAE autoencoder) — 권장
- **Regularization Theory Deep Dive** (data augmentation, Mixup) — **필수** (DeiT)
- **Kernel Methods Deep Dive** (contrastive as kernel) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Vision Transformer의 기초 (5개 문서)
- **ViT의 수학적 구조 (Dosovitskiy 2021)** — Input $x \in \mathbb{R}^{H \times W \times C}$ → $N = HW/P^2$ patches, patch embedding $z_0 = [x_{\text{class}}; x_p^1 E; \ldots; x_p^N E] + E_{\text{pos}}$
- **Patch Embedding = Conv2d 등가** — $P \times P$ conv with stride $P$, kernel size $P$가 patch embedding과 동치, 구현 간소화
- **Class Token과 Global Pooling** — [CLS] token $x_{\text{class}}$의 역할, final layer에서 classification head, average pooling과의 비교 (Touvron 2021)
- **Positional Embedding의 변종** — Learned 1D (ViT), 2D (일부 변종), relative (Swin), CPE (ConViT)의 수학적 차이
- **Inductive Bias 부족 문제** — CNN의 translation equivariance·locality 없음, 작은 데이터에서 CNN에 뒤짐, ImageNet-22k pretrain 필요성, DeiT의 해결책

### Chapter 2: 효율적 ViT와 Hierarchical 아키텍처 (5개 문서)
- **DeiT — Data-Efficient ViT (Touvron 2021)** — Distillation token $x_{\text{dist}}$, teacher (CNN) hard labels, ImageNet-1k만으로 경쟁력, 강한 augmentation (Mixup, CutMix, RandAugment)
- **Swin Transformer — Shifted Window (Liu 2021)** — Window-based local self-attention $O(n \cdot w^2)$, shifted window로 cross-window connection, hierarchical feature map (4x, 8x, 16x, 32x downsampling)
- **PVT — Pyramid Vision Transformer (Wang 2021)** — Spatial reduction attention (SRA), 각 stage마다 spatial downsampling, pyramid feature for dense prediction
- **CvT, CoAtNet — CNN-ViT Hybrid** — CvT (Wu 2021)의 convolutional embedding, CoAtNet (Dai 2021)의 conv + attention 층 교차, inductive bias와 scalability 균형
- **MViT, Focal Transformer — Multi-Scale** — Pool-based attention (MViT, Fan 2021), focal self-attention (Yang 2021)의 fine-coarse 혼합

### Chapter 3: Self-Supervised Vision 기초 — Contrastive (5개 문서)
- **Self-Supervised Learning의 3가지 패러다임** — Generative (MAE), Discriminative (contrastive), Self-distillation (DINO), 각 방식의 손실함수 formalism
- **SimCLR — Simple Contrastive Framework (Chen 2020)** — Two augmented views $(\tilde{x}_i, \tilde{x}_j)$, projection head $g(\cdot)$, InfoNCE loss $L = -\log \frac{\exp(\text{sim}(z_i, z_j)/\tau)}{\sum_k \exp(\text{sim}(z_i, z_k)/\tau)}$
- **InfoNCE와 Mutual Information 하한** — $I(X; Y) \geq \log N - L_{\text{InfoNCE}}$, Poole 2019의 MI bound, negative sample 수와 tightness
- **MoCo — Momentum Contrast (He 2020)** — Queue-based negative (large N without large batch), momentum encoder $\theta_k \leftarrow \tau \theta_k + (1-\tau) \theta_q$, v2/v3의 MLP projection 추가
- **BYOL, SimSiam — Without Negatives (Grill 2020, Chen 2021)** — Positive만으로 collapse 안 하는 이유, predictor network의 역할, stop-gradient의 중요성, Tian 2021의 분석

### Chapter 4: Self-Distillation — DINO Family (4개 문서)
- **DINO 알고리즘 (Caron 2021)** — Student $f_{\theta_s}$, Teacher $f_{\theta_t}$, 두 view 모두 입력, $p_t = \text{softmax}(f_t(x)/\tau_t)$, $L = -\sum p_t \log p_s$
- **Teacher EMA와 Collapse 방지** — $\theta_t \leftarrow \lambda \theta_t + (1-\lambda) \theta_s$, centering $c \leftarrow m c + (1-m) \frac{1}{B} \sum f_t$, sharpening $\tau_t < \tau_s$
- **Emergent Properties of DINO** — Attention map이 자연스럽게 object segmentation, $k$-NN classifier로 linear probe 수준 성능, semantic feature의 emergence
- **iBOT과 DINOv2 (Oquab 2024)** — iBOT의 masked image modeling 추가, DINOv2의 multi-objective (DINO + iBOT + KoLeo + SwAV), 1B parameter ViT-g

### Chapter 5: Masked Image Modeling (5개 문서)
- **BEiT — BERT for Image (Bao 2022)** — DALL-E의 discrete VAE로 patch를 token화, masked patch의 token 예측, BERT MLM과 유사 structure
- **MAE — Masked Autoencoder (He 2022)** — 75% mask ratio (NLP의 15% 대비 훨씬 높음), pixel reconstruction, asymmetric encoder-decoder (encoder는 visible만), lightweight decoder
- **MAE의 정보이론적 해석** — 이미지의 높은 redundancy가 high masking 허용, reconstruction objective의 inductive bias, scaling에 따른 linear probe vs fine-tune 성능
- **SimMIM — Simplified MIM (Xie 2022)** — MAE 대비 simpler architecture, small/medium size에서 더 단순한 구현, masking strategy 비교
- **MaskFeat, MVP — Advanced Targets** — Pixel 대신 HOG feature (MaskFeat, Wei 2022), pretrained model feature (MVP, Wei 2022), 더 semantic한 target

### Chapter 6: Multimodal Vision-Language (5개 문서)
- **CLIP — Contrastive Language-Image Pretraining (Radford 2021)** — 400M image-text pairs, dual encoder (ViT + Transformer text), contrastive loss symmetric (image→text + text→image)
- **CLIP의 Zero-Shot Transfer** — Class prompt "a photo of a {cls}", image-text similarity로 classification, 전통적 SSL보다 다양한 task에 transfer
- **BLIP, BLIP-2 — Bootstrapping Captioners (Li 2022, 2023)** — Caption generation + image-text matching, Q-Former 구조, frozen LLM에 vision 주입
- **LLaVA — Visual Instruction Tuning (Liu 2023)** — CLIP vision encoder + projection + LLM (Vicuna/LLaMA), simple linear projection이 MLP보다 효율적, visual instruction dataset
- **Flamingo, GPT-4V, Gemini — Interleaved Vision-Language** — Flamingo (Alayrac 2022)의 gated cross-attention, interleaved image-text 처리, GPT-4V·Gemini의 native multimodal

### Chapter 7: Vision Generation과 Emerging Topics (4개 문서)
- **Vision Generation with Transformers** — Parti (autoregressive), Muse (masked token), 각 방식의 속도-품질 trade-off
- **Scaling Laws for Vision** — Zhai 2022 "Scaling Vision Transformers", data·model scaling의 power law, 데이터 증강과 regularization의 scaling 효과
- **3D Vision — NeRF와 3D Gaussian Splatting** — Neural Radiance Fields (Mildenhall 2020), 3D GS (Kerbl 2023), neural rendering의 수학
- **Video Understanding — TimeSformer, VideoMAE** — Divided space-time attention (TimeSformer), video MAE (Tong 2022), VideoLLaMA, large video transformer

---

각 챕터는 **4~5개 문서**로 구성해줘. 총 **33개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 기법이 vision transformer에 중요한가
## 📐 수학적 선행 조건 (Transformer, CNN, Generative, Reg 참조)
## 📖 직관적 이해
   — Patch grid, attention map 시각화
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Patch = conv 등가, InfoNCE MI bound, DINO collapse
## 💻 PyTorch 구현 검증
   — ViT 바닥부터, Swin window attention
   — SimCLR, DINO, MAE 각각 간단 재현
   — CLIP zero-shot 실험
## 🔗 실전 활용
   — 어느 SSL이 어느 task에 유리
## ⚖️ 가정과 한계
   — ViT의 inductive bias 부족
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Attention map 시각화** — ViT의 각 layer·head attention visualization (CLS→patches)
2. **Patch size sweep** — $P = 8, 16, 32$의 trade-off
3. **DINO self-segmentation** — 훈련 후 attention map이 object mask 역할
4. **MAE reconstruction** — 75% mask에서 reconstruction quality
5. **CLIP zero-shot** — ImageNet class에 대한 zero-shot accuracy
6. **Swin window vs ViT global** — 같은 compute에서 accuracy 비교

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
torchvision==0.16.0
timm==0.9.10
transformers==4.36.0
matplotlib==3.8.0
clip-anytorch==2.5.2   # CLIP 참조
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (ViT 바닥부터 + Patch = Conv 검증)
import torch
import torch.nn as nn
import torch.nn.functional as F

class PatchEmbed(nn.Module):
    """Patch embedding as conv"""
    def __init__(self, img_size=224, patch_size=16, in_chans=3, embed_dim=768):
        super().__init__()
        self.img_size = img_size; self.patch_size = patch_size
        self.num_patches = (img_size // patch_size) ** 2
        # 핵심: Conv2d with stride = kernel_size = patch_size
        self.proj = nn.Conv2d(in_chans, embed_dim,
                              kernel_size=patch_size, stride=patch_size)
    
    def forward(self, x):
        x = self.proj(x)  # [B, D, H/P, W/P]
        return x.flatten(2).transpose(1, 2)  # [B, N, D]

# Equivalent "naive" implementation
def patch_embed_naive(x, E, patch_size):
    """x: [B, C, H, W], E: [P²C, D]"""
    B, C, H, W = x.shape
    P = patch_size
    patches = x.unfold(2, P, P).unfold(3, P, P)  # [B, C, H/P, W/P, P, P]
    patches = patches.contiguous().view(B, C, -1, P, P)
    patches = patches.permute(0, 2, 1, 3, 4).reshape(B, -1, C * P * P)
    return patches @ E

# Verify equivalence
B, C, H, W, P = 2, 3, 224, 224, 16
D = 768
x = torch.randn(B, C, H, W)
conv_embed = PatchEmbed(H, P, C, D)
y_conv = conv_embed(x)
# E from conv weight
E = conv_embed.proj.weight.view(D, -1).T  # [P²C, D]
bias = conv_embed.proj.bias
y_naive = patch_embed_naive(x, E, P) + bias
print(f'Max diff: {(y_conv - y_naive).abs().max():.2e}')  # near 0

class ViTBlock(nn.Module):
    def __init__(self, dim, n_heads):
        super().__init__()
        self.norm1 = nn.LayerNorm(dim)
        self.attn = nn.MultiheadAttention(dim, n_heads, batch_first=True)
        self.norm2 = nn.LayerNorm(dim)
        self.mlp = nn.Sequential(
            nn.Linear(dim, 4*dim), nn.GELU(),
            nn.Linear(4*dim, dim)
        )
    
    def forward(self, x):
        # Pre-LN (Xiong 2020)
        xn = self.norm1(x)
        x = x + self.attn(xn, xn, xn, need_weights=False)[0]
        x = x + self.mlp(self.norm2(x))
        return x

class ViT(nn.Module):
    def __init__(self, img_size=224, patch_size=16, num_classes=1000,
                 dim=768, depth=12, n_heads=12):
        super().__init__()
        self.patch_embed = PatchEmbed(img_size, patch_size, 3, dim)
        N = self.patch_embed.num_patches
        self.cls_token = nn.Parameter(torch.zeros(1, 1, dim))
        self.pos_embed = nn.Parameter(torch.zeros(1, N+1, dim))
        self.blocks = nn.ModuleList([ViTBlock(dim, n_heads) for _ in range(depth)])
        self.norm = nn.LayerNorm(dim)
        self.head = nn.Linear(dim, num_classes)
    
    def forward(self, x):
        B = x.shape[0]
        x = self.patch_embed(x)  # [B, N, D]
        cls = self.cls_token.expand(B, -1, -1)
        x = torch.cat([cls, x], dim=1)
        x = x + self.pos_embed
        for blk in self.blocks: x = blk(x)
        x = self.norm(x)
        return self.head(x[:, 0])  # CLS token

# DINO loss skeleton
def dino_loss(student_out, teacher_out, center, tau_s=0.1, tau_t=0.04):
    """Self-distillation with centering + sharpening"""
    student = F.log_softmax(student_out / tau_s, dim=-1)
    # Teacher: center + sharpen + stop-grad
    teacher = F.softmax((teacher_out - center) / tau_t, dim=-1).detach()
    return -(teacher * student).sum(-1).mean()

# MAE masking
def random_masking(x, mask_ratio=0.75):
    B, N, D = x.shape
    L = int(N * (1 - mask_ratio))  # visible
    noise = torch.rand(B, N)
    ids_shuffle = torch.argsort(noise, dim=1)
    ids_keep = ids_shuffle[:, :L]
    x_visible = torch.gather(x, 1, ids_keep.unsqueeze(-1).expand(-1, -1, D))
    return x_visible, ids_shuffle

# InfoNCE (SimCLR style)
def info_nce(z1, z2, tau=0.1):
    """z1, z2: [B, D] two views"""
    B = z1.shape[0]
    z1 = F.normalize(z1, dim=-1)
    z2 = F.normalize(z2, dim=-1)
    logits = z1 @ z2.T / tau
    labels = torch.arange(B)
    return F.cross_entropy(logits, labels)
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Transformer, CNN, Generative, Reg 선행 필수"
   - ViT → SSL → Multimodal의 진화 축
   - CLIP 기반 foundation model 지도
3. **챕터별 문서 작성**: ViT 기초 → Hierarchical → Contrastive → DINO → MAE → Multimodal → Generation

---

## 📚 참고 자료

- **An Image is Worth 16x16 Words** (Dosovitskiy et al. 2021) — ViT
- **Training data-efficient image transformers & distillation** (Touvron et al. 2021) — DeiT
- **Swin Transformer** (Liu et al. 2021)
- **PVT** (Wang et al. 2021)
- **CoAtNet** (Dai et al. 2021)
- **A Simple Framework for Contrastive Learning** (Chen et al. 2020) — SimCLR
- **Momentum Contrast for Unsupervised Visual Representation Learning** (He et al. 2020) — MoCo
- **Bootstrap Your Own Latent** (Grill et al. 2020) — BYOL
- **Emerging Properties in Self-Supervised ViT** (Caron et al. 2021) — DINO
- **DINOv2** (Oquab et al. 2024)
- **iBOT** (Zhou et al. 2022)
- **BEiT** (Bao et al. 2022)
- **Masked Autoencoders Are Scalable Vision Learners** (He et al. 2022) — MAE
- **SimMIM** (Xie et al. 2022)
- **Learning Transferable Visual Models From Natural Language Supervision** (Radford et al. 2021) — CLIP
- **BLIP, BLIP-2** (Li et al. 2022, 2023)
- **LLaVA** (Liu et al. 2023)
- **Flamingo** (Alayrac et al. 2022)

---

## 💡 핵심 분석 대상

```
Vision Transformer의 지도

───── ViT 기초 (Dosovitskiy 2021) ─────

Input: x ∈ ℝ^{H×W×C}
Patches: N = HW/P², each P×P×C

Patch Embedding:
  x_p^i ∈ ℝ^{P²C}
  z_0 = [x_cls; x_p^1 E; ...; x_p^N E] + E_pos
  E ∈ ℝ^{P²C × D}

핵심: Patch embedding = Conv2d(stride=P)
   nn.Conv2d(C, D, P, stride=P)
   → flatten and transpose

Inductive Bias 부족:
  Translation equivariance X
  Locality X
  → large data 필요
  → ImageNet-22k pretrain

DeiT 해결 (Touvron 2021):
  Distillation token from CNN teacher
  Strong augmentation (Mixup, CutMix, RandAug)
  ImageNet-1k만으로 경쟁력

───── Hierarchical ViT ─────

Swin (Liu 2021):
  Window-based local attn O(nW²)
  Shifted window로 cross-window 정보
  Hierarchical features (4 stages)
  Dense prediction (detection, seg)

PVT (Wang 2021):
  Spatial Reduction Attention
  Pyramid feature map

CvT / CoAtNet:
  Conv + Attention 혼합
  초기에 conv (local), 후반에 attn (global)

───── Self-Supervised Learning ─────

Three paradigms:
  1. Contrastive (SimCLR, MoCo)
  2. Self-distillation (BYOL, DINO)
  3. Masked Image Modeling (MAE, BEiT)

InfoNCE Loss:
  L = -log exp(s_{ij}/τ) / Σ_k exp(s_{ik}/τ)
  
  MI bound (Poole 2019):
    I(X;Y) ≥ log N - L_InfoNCE
  
  Temperature τ: sharpness control
  Negative count N: tightness

MoCo (He 2020):
  Queue for negatives (large N)
  Momentum encoder:
    θ_k ← τ θ_k + (1-τ) θ_q
  v3: MLP projection, no queue

BYOL / SimSiam:
  No negatives!
  Predictor + stop-grad
  Tian 2021: SG가 collapse 방지

───── DINO (Caron 2021) ─────

Student-Teacher self-distillation:
  Student: f_θ_s
  Teacher: f_θ_t (EMA)
  
  p_t = softmax((f_t(x) - c) / τ_t)
        └── centering + sharpening
  p_s = softmax(f_s(x') / τ_s)
  
  L = -p_t^T log p_s

Teacher EMA:
  θ_t ← λ θ_t + (1-λ) θ_s

Collapse 방지:
  Centering c ← m c + (1-m) mean(f_t)
  Sharpening τ_t < τ_s
  → 분포가 너무 uniform/one-hot 되지 않게

Emergent properties:
  Attention → object segmentation
  k-NN ~ linear probe 성능

DINOv2 (Oquab 2024):
  DINO + iBOT (MIM) + KoLeo + SwAV
  Foundation model for vision

───── Masked Image Modeling ─────

BEiT (Bao 2022):
  DALL-E dVAE → discrete tokens
  Mask patches, predict tokens
  BERT MLM 유사

MAE (He 2022):
  Key insight: 75% masking
  (NLP는 15%, vision은 redundancy 높음)
  
  Asymmetric encoder-decoder:
    Encoder: visible patches만 (빠름)
    Decoder: lightweight, all patches
  
  Pixel reconstruction loss
  Linear probe < Fine-tune (contrastive와 반대)

SimMIM (Xie 2022):
  Simpler architecture
  Small/medium size 유리

MaskFeat / MVP:
  Pixel 대신 HOG / pretrained feature
  더 semantic

───── Multimodal (Vision-Language) ─────

CLIP (Radford 2021):
  400M image-text pairs
  Dual encoder:
    Image: ViT
    Text: Transformer
  
  Contrastive loss (symmetric):
    L_img2txt + L_txt2img
  
  Zero-shot classification:
    prompt: "a photo of {cls}"
    similarity(image_emb, text_emb)

BLIP (Li 2022):
  Caption generation + ITM
  Bootstrapping from noisy data

BLIP-2 (Li 2023):
  Q-Former (learnable queries)
  Frozen vision + frozen LLM
  Efficient vision-LLM fusion

LLaVA (Liu 2023):
  CLIP encoder + projection + LLM
  Visual instruction tuning
  Simple but effective

Flamingo (Alayrac 2022):
  Gated cross-attention
  Interleaved image-text
  Few-shot learning

Modern: GPT-4V, Gemini, Claude 3
  Native multimodal
  Strong visual reasoning

───── Emerging Topics ─────

Vision Generation:
  Parti: autoregressive
  Muse: masked token
  MaskGIT: fast generation

Scaling Laws (Zhai 2022):
  Vision ViT의 power law
  Augmentation과 regularization

3D:
  NeRF (Mildenhall 2020)
  3D Gaussian Splatting (Kerbl 2023)
  Neural rendering

Video:
  TimeSformer: divided space-time attn
  VideoMAE (Tong 2022)
  VideoLLaMA, VideoGPT

───── 레포 간 연결 ─────

Transformer (Layer 3):
  Core attention mechanism

CNN (Layer 3):
  Inductive bias 비교
  Conv as baseline

Generative Model (Layer 3):
  MAE = masked autoencoder
  CLIP contrastive = InfoNCE

Regularization (Layer 2):
  DeiT의 augmentation
  Mixup, CutMix

Kernel Methods (Layer 1):
  Contrastive = kernel-like
  InfoNCE와 SVM

Object Detection (다음):
  DETR, DINO-DETR
  Vision backbone
  
Diffusion (다음):
  DiT, Stable Diffusion
  UNet 대체
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~5개씩)
- 각 문서가 다루는 핵심 정리·기법 (3~4줄)
- 전체 문서 개수 확인 (33개 목표)
- Python + PyTorch + timm + CLIP 실험 환경
- Transformer, CNN, Generative, Reg 레포 참조 관계
- Detection, Diffusion으로 이어지는 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
