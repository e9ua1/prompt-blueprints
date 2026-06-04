# CNN Deep Dive 레포지토리 제작 프롬프트

나는 "CNN Deep Dive" 레포지토리를 만들려고 해.
`nn.Conv2d(3, 64, 3, padding=1)`을 **쌓는 것**과, **Convolution이 왜 translation equivariance를 가지며**, **이 대칭성이 MLP 대비 VC 차원을 어떻게 감소시키는지** 증명할 수 있는 것은 다르다.
ResNet을 **쓰는 것**과, **$y = x + F(x)$의 gradient $\partial y/\partial x = I + \partial F/\partial x$가 왜 깊은 네트워크의 gradient vanishing을 완화**하고, **He et al. 2016의 identity approximation 증명**이 residual 구조의 표현력을 어떻게 정당화하는지 유도할 수 있는 것은 다르다.
Dilated Convolution을 **사용하는 것**과, **Effective Receptive Field가 Gaussian 분포를 따른다는 Luo et al. (2016)의 발견**과, **dilation이 $O(k^L)$ vs $O(kL)$로 RF를 지수적으로 키우는 이론**을 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "CNN의 수학 — 대칭성·지역성·계층성을 표현력 이론으로 재구성"

**핵심 차별화**:
1. **Convolution의 Equivariance 엄밀 정의** — Group action 관점에서 $\phi(g \cdot x) = g \cdot \phi(x)$, translation equivariance 증명, rotation·scale 확장(Group CNN)
2. **Receptive Field의 3가지 관점** — 이론적 RF, Effective RF (Luo 2016의 Gaussian), Gradient-based RF (attention map)
3. **ResNet의 Identity Approximation Theorem** — 깊은 residual network의 universal approximation + gradient flow 분석
4. **현대 CNN의 설계 원리** — VGG(depth) → ResNet(skip) → DenseNet(dense) → EfficientNet(compound scaling) → ConvNeXt(현대화)의 설계 수학

**타겟 독자**:
- Conv layer를 쓰지만 **parameter sharing이 왜 VC 차원을 줄이고 이것이 일반화와 어떻게 연결**되는지 모르는 사람
- Padding의 의미를 아는데 **'same' vs 'valid'가 equivariance 성질을 어떻게 바꾸는지** 모르는 사람
- ResNet의 성공을 경험했지만 **"왜 identity shortcut이 gradient highway를 만드는지"** 수학적으로 설명 못하는 사람
- Dilated CNN, Depthwise Separable Conv의 **이론적 이득**(FLOPs 감소 외)을 정량화 못하는 사람
- Group Equivariant CNN이 **왜 rotation invariance가 필요한 의료영상에서 SOTA**인지 모르는 사람

**선행 학습**:
- **Neural Network Theory Deep Dive** (UAT, Backprop, 초기화) — **필수**
- **Linear Algebra Deep Dive** (Convolution as Toeplitz matrix, 고유분해) — **필수**
- **Functional Analysis Deep Dive** (Fourier transform, convolution theorem) — Spectral 분석에 필요
- **Optimization Theory Deep Dive** (Gradient flow) — ResNet 분석에 필요
- **Regularization Theory Deep Dive** (BatchNorm) — 현대 CNN 훈련에 필수

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Convolution의 수학적 정의와 Equivariance (5개 문서)
- **Discrete Convolution의 엄밀한 정의** — $(f * g)[n] = \sum_m f[m] g[n-m]$, 2D 확장 $(I * K)[i, j] = \sum_{m, n} I[m, n] K[i-m, j-n]$, cross-correlation과의 차이(PyTorch/TF 구현은 cross-correlation)
- **Translation Equivariance의 Group-theoretic 정의** — Translation group $(\mathbb{Z}^2, +)$의 action $T_a f(x) = f(x-a)$, convolution이 $T_a (f * g) = (T_a f) * g$ 만족 증명, 이것이 대칭성 preservation
- **Group Equivariant CNN (Cohen & Welling 2016)** — Rotation group 확장 $G = \mathbb{Z}^2 \rtimes \mathbb{Z}_n$, steerable filter 기반 일반화, 의료영상·분자구조 응용
- **Convolution의 Toeplitz 행렬 표현** — 1D conv를 matrix-vector product로, circulant matrix의 고유분해가 DFT → FFT-based convolution ($O(n \log n)$)
- **Convolution Theorem과 Frequency Domain** — $\mathcal{F}(f * g) = \mathcal{F}(f) \cdot \mathcal{F}(g)$, CNN을 frequency response로 해석, pooling이 low-pass filter

### Chapter 2: CNN 아키텍처의 기본 연산 (5개 문서)
- **Convolution Layer의 Forward/Backward** — 각 채널별 $k \times k$ kernel, multi-input/output channel의 tensor 표기, backward의 수학 (kernel gradient = input과 output gradient의 convolution)
- **Pooling의 수학적 역할** — Max/Average pooling이 local translation invariance 제공 (small shift에 robust), downsampling으로 RF 확장, 미분 불가능 지점과 backward 정의
- **Padding 전략과 Boundary 효과** — Zero, reflect, replicate padding의 의미, 'same' convolution의 크기 보존, boundary artifact
- **Stride, Dilation, Transposed Convolution** — Stride가 output 크기 및 RF에 미치는 영향, Dilated conv의 수학 $(I *_d K)[i] = \sum K[k] I[i + dk]$, transposed conv가 upsampling에 쓰이는 이유와 checkerboard artifact
- **Depthwise Separable Convolution (Chollet 2017, MobileNet)** — Depthwise ($K$ channels each with own kernel) + Pointwise ($1 \times 1$) 분해, 파라미터 $O(k^2 \cdot C_{in} \cdot C_{out})$에서 $O(k^2 \cdot C_{in}) + O(C_{in} \cdot C_{out})$로 감소

### Chapter 3: Receptive Field 분석 (4개 문서)
- **Theoretical Receptive Field 계산** — $L$개 층 후 $RF_L = 1 + \sum_{l=1}^L (k_l - 1) \prod_{i=1}^{l-1} s_i$ 재귀 공식 유도, downstream이 얼마나 많은 input pixel을 볼 수 있는가
- **Effective Receptive Field (Luo et al. 2016)** — Gradient 기반 측정, 실제 ERF는 이론치보다 작고 Gaussian 분포 (center-concentrated), ReLU 활성화와 initialization의 영향
- **Dilated Convolution의 RF 증가 (Yu & Koltun 2016)** — Dilation rate $d$일 때 RF가 $k$ vs $k + (k-1)(d-1)$, exponential dilation으로 $O(2^L)$ RF (WaveNet 유사)
- **Semantic Segmentation에서 RF의 중요성** — FCN, U-Net, DeepLab의 설계 원리, atrous convolution + spatial pyramid, global context 필요성

### Chapter 4: ResNet과 Skip Connection 이론 (6개 문서)
- **Residual Block의 정의와 Forward Flow** — $y = F(x; W) + x$, identity shortcut의 구조, pre-activation vs post-activation (He et al. 2016b), BN-ReLU-Conv 순서의 gradient 안정성
- **Gradient Flow 분석** — $\partial y / \partial x = I + \partial F / \partial x$, 깊은 residual network에서 gradient가 identity path로 직접 흐름, vanishing gradient 완화 수학적 증명
- **Identity Approximation Theorem (He et al. 2016)** — $F(x)$가 residual만 학습하면 되므로 identity에 가까운 초기 상태에서 학습 쉬움, 왜 deeper networks가 plain CNN에서 악화되고 ResNet에서 개선되는가
- **DenseNet의 Dense Connection (Huang et al. 2017)** — 각 층이 이전 모든 층의 concatenation을 입력으로, feature reuse, $L$개 층에서 $L(L+1)/2$ connection, gradient path 다양화
- **Highway Networks와 LSTM-style Gating** — Srivastava 2015: $y = T(x) \cdot H(x) + (1-T(x)) \cdot x$, learnable gate, ResNet이 $T \equiv 1$인 특수경우
- **Stochastic Depth (Huang et al. 2016)** — 훈련 시 random하게 layer drop, implicit 앙상블 + 훈련 가속, Dropout의 layer-level 일반화, 일반화 개선

### Chapter 5: 현대 CNN 아키텍처 (5개 문서)
- **VGG와 Depth의 효과 (Simonyan & Zisserman 2014)** — $3 \times 3$ kernel 연속 사용이 $5 \times 5$, $7 \times 7$과 같은 RF이면서 파라미터 적음, 비선형성 더 많음, plain depth의 한계 (152+ layer에서 수렴 실패)
- **Inception/GoogLeNet의 Multi-Scale Feature** — $1 \times 1, 3 \times 3, 5 \times 5$, pooling을 병렬로, $1 \times 1$ conv를 dimensionality reduction으로, Inception v2/v3/v4의 진화
- **EfficientNet과 Compound Scaling (Tan & Le 2019)** — Depth $d$, Width $w$, Resolution $r$의 균형 $d = \alpha^\phi, w = \beta^\phi, r = \gamma^\phi$, $\alpha \cdot \beta^2 \cdot \gamma^2 \approx 2$ 제약, 동일 FLOPs에서 최적
- **ConvNeXt와 CNN의 현대화 (Liu et al. 2022)** — Transformer 설계 원리를 CNN으로 역수입 (large kernel, LayerNorm, GELU, inverted bottleneck), Swin Transformer에 필적하는 성능
- **NAS (Neural Architecture Search) and AutoML** — NASNet, AmoebaNet, RegNet의 자동 탐색, DARTS의 differentiable search, design space 이론 (Radosavovic et al. 2020)

### Chapter 6: CNN의 응용 이론 (5개 문서)
- **Image Classification의 전형적 Pipeline** — ImageNet benchmark, Top-1/Top-5 accuracy 측정, cross-entropy loss, softmax classifier의 수학
- **Object Detection — Two-Stage (Faster R-CNN)** — RoI pooling/align의 수학, anchor box 디자인, RPN(Region Proposal Network), NMS의 역할
- **One-Stage Detection — YOLO, RetinaNet** — YOLO의 grid-based prediction, Focal Loss (Lin et al. 2017) $FL(p_t) = -\alpha(1-p_t)^\gamma \log p_t$의 유도와 class imbalance 해결
- **Semantic Segmentation — FCN, U-Net, DeepLab** — Fully convolutional network, encoder-decoder 구조, atrous spatial pyramid pooling, dense prediction
- **Self-Supervised Learning with CNNs** — Jigsaw, Colorization, Rotation prediction, SimCLR의 contrastive loss, MoCo의 momentum encoder, CNN feature의 transfer learning

### Chapter 7: CNN의 이론적 한계와 Vision Transformer (4개 문서)
- **CNN의 Inductive Bias 장단점** — Translation equivariance, locality, hierarchy가 inductive bias, 작은 데이터에서 유리, 매우 큰 데이터에서 Transformer에 뒤짐(Dosovitskiy 2021)
- **Adversarial Examples와 CNN의 취약성** — $\|x - x'\| < \epsilon$인 작은 perturbation이 prediction을 바꾸는 현상, FGSM (Goodfellow 2014), robustness와 accuracy의 trade-off
- **Spectral Bias와 CNN** — NN이 low-frequency를 먼저 배우는 현상 (Rahaman 2019), high-frequency feature 학습 어려움, Fourier feature의 대안 (NeRF)
- **Vision Transformer와 하이브리드** — ViT의 patch embedding = CNN의 첫 conv, Swin Transformer의 계층적 구조, ConvNeXt의 반격, CNN vs Transformer의 수렴

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **34개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 설계가 CNN에 필수인가
## 📐 수학적 선행 조건 (LA, NN Theory, Opt, Reg 레포 참조)
## 📖 직관적 이해
   — 시각적 매개체 (filter 시각화, RF 경로, gradient flow)
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Equivariance 증명, RF 공식 유도, ResNet gradient flow
## 💻 NumPy/PyTorch 구현 검증
   — Convolution을 Toeplitz로 구현, FFT-based 비교
   — ResNet 없이 깊은 NN 훈련 실패 재현, Skip 추가하여 개선
## 🔗 실전 활용
   — 언제 이 architecture를 선택, scaling 가이드
## ⚖️ 가정과 한계
   — Inductive bias의 양면성, adversarial 취약성
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Filter 시각화 필수** — 훈련된 CNN의 첫 layer filter, 중간 feature map, Grad-CAM을 모든 장에서
2. **Receptive Field 시각화** — gradient backpropagation으로 ERF 측정, 이론치와 비교
3. **ResNet gradient flow 실험** — 56-layer Plain vs ResNet 훈련 curve 재현 (He 2016 Fig 1)
4. **Parameter count 비교 테이블** — 각 architecture의 params/FLOPs/accuracy trade-off
5. **Translation equivariance 실험** — 이미지를 shift해서 output도 같은 shift인지 확인
6. **Adversarial example 재현** — FGSM으로 간단한 adversarial 만들기

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
torchvision==0.16.0
matplotlib==3.8.0
timm==0.9.10          # 현대 아키텍처 모음
scikit-image==0.22.0  # 이미지 처리
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Plain vs ResNet, Effective RF 측정)
import torch
import torch.nn as nn
import matplotlib.pyplot as plt
import numpy as np

class PlainBlock(nn.Module):
    def __init__(self, c):
        super().__init__()
        self.conv = nn.Sequential(
            nn.Conv2d(c, c, 3, padding=1), nn.BatchNorm2d(c), nn.ReLU(),
            nn.Conv2d(c, c, 3, padding=1), nn.BatchNorm2d(c), nn.ReLU()
        )
    def forward(self, x): return self.conv(x)

class ResBlock(nn.Module):
    def __init__(self, c):
        super().__init__()
        self.conv = nn.Sequential(
            nn.Conv2d(c, c, 3, padding=1), nn.BatchNorm2d(c), nn.ReLU(),
            nn.Conv2d(c, c, 3, padding=1), nn.BatchNorm2d(c)
        )
        self.relu = nn.ReLU()
    def forward(self, x): return self.relu(self.conv(x) + x)  # identity

def build_net(block_type, depth=20, c=16):
    blocks = [block_type(c) for _ in range(depth)]
    return nn.Sequential(nn.Conv2d(3, c, 3, padding=1), *blocks,
                         nn.AdaptiveAvgPool2d(1), nn.Flatten(), nn.Linear(c, 10))

# Plain 20과 ResNet 20 비교 훈련 — Plain은 Deep에서 수렴 어려움
# 각 block output의 gradient magnitude 측정으로 gradient flow 시각화

# Effective Receptive Field 측정
def measure_erf(model, input_size=(3, 224, 224), n_samples=100):
    """Luo 2016: center pixel gradient로 input에 대한 기울기 측정"""
    erf = torch.zeros(input_size[1:])
    for _ in range(n_samples):
        x = torch.randn(1, *input_size, requires_grad=True)
        y = model(x)
        center = y.shape[-1] // 2
        y[0, :, center, center].sum().backward()
        erf += x.grad[0].abs().sum(0).detach()
    return erf / n_samples

# Gaussian 모양의 ERF를 확인 (center-concentrated)
# 이론 RF는 꽉 찬 정사각형이지만 실제는 Gaussian
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "NN Theory, LA, FA, Opt, Reg 선행 필수" 명시
   - CNN → ResNet → Vision Transformer의 역사적 맥락
   - Vision Transformer (Layer 3 또는 심화)와의 연결
3. **챕터별 문서 작성**: Convolution 기초 → 기본 연산 → RF → ResNet → 현대 아키텍처 → 응용 → 한계·ViT

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Deep Learning** (Goodfellow et al.) — Chap 9 CNN 표준
- **ImageNet Classification with Deep CNN** (Krizhevsky et al. 2012) — AlexNet 원전
- **Very Deep Convolutional Networks** (Simonyan & Zisserman 2014) — VGG
- **Deep Residual Learning for Image Recognition** (He et al. 2016) — ResNet
- **Identity Mappings in Deep Residual Networks** (He et al. 2016b)
- **Densely Connected Convolutional Networks** (Huang et al. 2017)
- **Group Equivariant Convolutional Networks** (Cohen & Welling 2016)
- **Understanding the Effective Receptive Field** (Luo et al. 2016)
- **Multi-Scale Context Aggregation by Dilated Convolutions** (Yu & Koltun 2016)
- **EfficientNet: Rethinking Model Scaling** (Tan & Le 2019)
- **A ConvNet for the 2020s** (Liu et al. 2022) — ConvNeXt
- **An Image is Worth 16x16 Words** (Dosovitskiy et al. 2021) — ViT
- **Explaining and Harnessing Adversarial Examples** (Goodfellow et al. 2014)

---

## 💡 핵심 분석 대상

```
CNN의 수학적 구조

┌──── Convolution의 본질 ────┐
│                              │
│ (I * K)[i,j] = ΣΣ I K         │
│                              │
│ 3가지 관점:                  │
│ ├── 공간 영역: sliding window │
│ ├── 선형대수: Toeplitz matrix │
│ └── 주파수: F(I·K) = F(I)F(K) │
│                              │
│ Equivariance (핵심!):        │
│ T_a(I * K) = (T_a I) * K      │
│ → shift in → shift out       │
│ → image 처리에 적합          │
└──────────────────────────────┘

┌──── Parameter 절약 ────┐
│                          │
│ MLP: O(H·W·C_in·C_out)   │
│ Conv: O(k²·C_in·C_out)    │
│                          │
│ Parameter sharing →      │
│   VC 차원 감소            │
│   일반화 향상            │
│                          │
│ Depthwise Sep Conv:      │
│   Depthwise: O(k²·C_in)   │
│   + Pointwise: O(C²)     │
│   → MobileNet 파라미터 1/8│
└──────────────────────────┘

┌──── Receptive Field ────┐
│                           │
│ Theoretical:              │
│   RF_L = 1 + Σ(k_l-1)∏s_i │
│                           │
│ Effective (Luo 2016):     │
│   Gaussian distribution   │
│   center-concentrated     │
│   ratio = ERF/RF ≈ 1/√L   │
│                           │
│ Dilation:                 │
│   d=1,2,4,8,...           │
│   RF exponentially ↑       │
│   WaveNet, DeepLab        │
└───────────────────────────┘

┌──── ResNet Mathematics ────┐
│                              │
│ Forward: y = F(x; W) + x     │
│                              │
│ Backward:                    │
│ ∂y/∂x = I + ∂F/∂x            │
│         └── always there     │
│             gradient highway │
│                              │
│ Deep composition:            │
│ y_L = y_0 + Σ_l F_l(y_{l-1}) │
│         └── residual sum     │
│                              │
│ Gradient to x_0:             │
│ ∂L/∂x_0 = ∂L/∂x_L ·          │
│   ∏_{l} (I + ∂F_l/∂x_{l-1}) │
│                              │
│ Identity term → vanishing    │
│   gradient 완화              │
└──────────────────────────────┘

┌──── 현대 CNN 계보 ────┐
│                         │
│ LeNet (LeCun 1998)      │
│   → MNIST, 5 layers    │
│                         │
│ AlexNet (2012)          │
│   → ImageNet 혁명       │
│   ReLU, Dropout, GPU    │
│                         │
│ VGG (2014)              │
│   → 3×3 쌓기, 깊이      │
│                         │
│ GoogLeNet/Inception     │
│   → Multi-scale, 1×1    │
│                         │
│ ResNet (2015)           │
│   → Skip connection     │
│   → 152 layers          │
│                         │
│ DenseNet (2017)         │
│   → Dense connection    │
│                         │
│ MobileNet (2017)        │
│   → Depthwise separable │
│                         │
│ EfficientNet (2019)     │
│   → Compound scaling    │
│                         │
│ ConvNeXt (2022)         │
│   → Transformer era 반격│
│                         │
│ ───── 경쟁자 ─────      │
│ ViT (2021)              │
│   → Attention으로 대체   │
│ Swin (2021)             │
│   → Hierarchical ViT    │
└─────────────────────────┘

┌──── 응용 태스크 지도 ────┐
│                            │
│ Classification:           │
│   → ResNet, EffNet, ViT   │
│                            │
│ Detection:                │
│ ├── Two-stage: Faster R-CNN│
│ └── One-stage: YOLO, RetinaNet│
│    Focal Loss (class imbal)│
│                            │
│ Segmentation:             │
│ ├── FCN (first end-to-end)│
│ ├── U-Net (bio/medical)   │
│ └── DeepLab (atrous SPP)  │
│                            │
│ Generation:               │
│   → StyleGAN (Layer 3 Gen)│
│                            │
│ Self-Supervised:          │
│ ├── SimCLR, MoCo          │
│ └── Masked Autoencoder    │
└────────────────────────────┘
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (34개 목표)
- Python + PyTorch + torchvision + timm 실험 환경
- LA, NN Theory, FA, Opt, Reg 레포의 참조 관계
- ViT로 이어지는 현대 비전의 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
