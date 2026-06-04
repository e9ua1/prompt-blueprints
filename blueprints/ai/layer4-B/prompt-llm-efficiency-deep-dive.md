# LLM Efficiency Deep Dive 레포지토리 제작 프롬프트

나는 "LLM Efficiency Deep Dive" 레포지토리를 만들려고 해.
LoRA를 **"저랭크 adapter"로 아는 것**과, **Hu et al. (2021)의 $W + \Delta W = W + BA$에서 $B \in \mathbb{R}^{d \times r}, A \in \mathbb{R}^{r \times k}$, $r \ll \min(d, k)$가 훈련 가능 parameter를 $d \cdot k$에서 $r(d+k)$로 줄이고** "intrinsic dimension" 가설 (Aghajanyan 2020)이 왜 이를 정당화하는지 유도할 수 있는 것은 다르다.
QLoRA를 **"4bit 양자화"로 아는 것**과, **Dettmers et al. (2023)의 NF4 (NormalFloat) quantile이 표준 정규분포의 분위수로 설계되어 INT4보다 reconstruction error가 낮은 수학**과, **Double Quantization으로 quantization constant 자체를 또 양자화**하는 아이디어를 이해하는 것은 다르다.
Speculative Decoding을 **"작은 모델로 초안"으로 아는 것**과, **Leviathan et al. (2023)의 accept probability $\min(1, p(x)/q(x))$가 왜 target 분포 $p$를 정확히 보존**하는지 rejection sampling theorem으로 증명할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "LLM 효율화의 수학 — Parameter·Memory·Compute·Latency의 모든 축에서"

**핵심 차별화**:
1. **PEFT 전체 계보와 수학** — LoRA, QLoRA, Adapter, Prefix Tuning, IA³의 수학적 분해
2. **Quantization 이론** — INT8, FP8, INT4, NF4의 이론적 비교, AWQ·GPTQ·SmoothQuant의 reconstruction objective
3. **MoE의 수학** — Top-k routing의 gradient, load balancing, expert capacity, Clark 2022의 scaling law
4. **Speculative Decoding의 rejection sampling 증명** — Draft model로 초안, target model로 verify, 수학적 losslessness

**타겟 독자**:
- LoRA를 쓰지만 **$r$과 $\alpha$(scaling)의 의미**와 target module 선택 기준을 모르는 사람
- 4bit quantization이 왜 NF4로 잘 작동하는지 **quantile-based 설계**의 근거를 모르는 사람
- MoE의 **load balancing loss** $L_{\text{aux}} = \alpha \sum_i f_i P_i$가 왜 expert utilization을 균등하게 만드는지 유도 못하는 사람
- Speculative Decoding이 **속도만 빠르고 품질 동일**이라는 주장의 수학적 근거를 모르는 사람
- Flash Attention 2 (Dao 2023)가 Flash Attention 1 대비 **어떤 work partitioning 최적화**로 2배 빨라졌는지 모르는 사람

**선행 학습**:
- **LLM Pretraining Deep Dive** (Transformer scaling) — **필수**
- **Transformer Deep Dive** (Attention, FFN, KV cache) — **필수**
- **Linear Algebra Deep Dive** (Low-rank decomposition, SVD) — **필수**
- **Information Theory Deep Dive** (Quantization, entropy) — **필수**
- **Optimization Theory Deep Dive** (훈련 memory) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Parameter-Efficient Fine-Tuning (PEFT) 기초 (5개 문서)
- **Full Fine-Tuning의 비용** — 7B LLaMA 한 파라미터 셋 = 28GB (FP32), Adam optimizer state = 56GB 추가, 총 84GB+ (7B 모델에 80GB 이상 필요)
- **PEFT의 분류와 공통 원리** — 기존 weight freeze + 작은 새 parameter 추가, 3가지 계열 (additive, selective, reparameterization), Lialin 2023 survey 분류
- **Intrinsic Dimension 가설 (Aghajanyan 2020)** — "Pre-trained LM은 낮은 intrinsic dimension의 task manifold에 살고 있음", $d_{\text{intrinsic}} \ll d_{\text{full}}$, random projection으로 fine-tuning 가능
- **Adapter Layers (Houlsby 2019)** — FFN 사이에 $W_{\text{down}} \sigma(W_{\text{up}})$ 삽입, $d \times r + r \times d$ params, pre-trained weight 불변, bottleneck dimension $r$ 튜닝
- **Prefix/Prompt Tuning** — Soft prompt $P \in \mathbb{R}^{l \times d}$를 input에 prepend, 훈련 대상은 $P$만, Li & Liang 2021 Prefix Tuning, Lester 2021 Prompt Tuning의 scale dependence

### Chapter 2: LoRA와 그 확장 (6개 문서)
- **LoRA — Low-Rank Adaptation (Hu 2021)** — $W + \Delta W = W + BA$, $B \in \mathbb{R}^{d \times r}, A \in \mathbb{R}^{r \times k}$, $r \ll d$, FFW + Attention weight에 적용, $r = 8$도 충분
- **LoRA의 Initialization과 $\alpha$ Scaling** — $A \sim \mathcal{N}(0, \sigma)$ 작게, $B = 0$ → 초기에 $\Delta W = 0$, $\alpha/r$ scaling factor, $\alpha = 2r$ 권장 (rsLoRA로 $\alpha/\sqrt{r}$ 발전)
- **Target Module 선택** — Original LoRA는 attention만, 현대는 모든 linear에 적용 (All-Linear), MLP LoRA가 더 중요하다는 후속 연구
- **DoRA — Weight Decomposed LoRA (Liu 2024)** — $W = m \cdot V/\|V\|$로 magnitude와 direction 분리, LoRA로 direction만 update, magnitude는 별도 학습, full fine-tuning에 더 가까움
- **Merged vs Unmerged Inference** — 훈련 후 $W' = W + BA$로 merge하면 inference latency 동일, multi-task adapter는 unmerged 유지, LoRA hub
- **Composable LoRA — Task Vectors (Ilharco 2023)** — $\Delta W_{\text{task}}$를 task vector로, 산술 연산으로 task 조합 ($\Delta_{A} + \Delta_B$, $\Delta_A - \Delta_B$), model editing 응용

### Chapter 3: Quantization 이론 (6개 문서)
- **Quantization의 기본 수학** — $x_q = \text{round}(x/s) + z$ (uniform), $s$ scale, $z$ zero-point, symmetric vs asymmetric, per-tensor vs per-channel
- **INT8 Post-Training Quantization** — Absolute max calibration, outlier의 영향 (LLM.int8(), Dettmers 2022), 혼합 정밀도 inference
- **GPTQ — Optimal Brain Compression (Frantar 2023)** — Hessian 기반 weight 업데이트, $\min_{\hat{W}} \|WX - \hat{W}X\|^2$ 각 row별, OBQ/OBS의 LLM 확장, 4bit에서 거의 손실 없음
- **AWQ — Activation-aware Quantization (Lin 2023)** — Activation magnitude가 큰 채널을 보존, $W \to W/s, X \to X \cdot s$로 equivalent transformation 후 quantize, outlier 보호
- **SmoothQuant (Xiao 2023)** — Activation outlier를 weight로 smooth transfer, W8A8 enable, $X \to X/s, W \to W \cdot s$ (AWQ와 반대), 양쪽 quantization 친화적
- **NF4 — NormalFloat 4bit (Dettmers 2023)** — 정규분포 quantile을 quantization level로, $q_i = F^{-1}(i/(2^k - 1))$, weight가 Gaussian이라는 가정 하에 INT4보다 낮은 error

### Chapter 4: QLoRA와 혼합 기법 (4개 문서)
- **QLoRA 전체 (Dettmers 2023)** — Base model을 NF4로 quantize + LoRA adapter는 BF16으로 훈련, gradient는 dequantize → backward → quantize, 65B 모델을 single 48GB GPU로 fine-tune
- **Double Quantization** — Quantization constant (32-bit float) 자체를 또 양자화 (8-bit), $0.5 \cdot 1/64 = 0.008$ bit/param 절약, 4bit+overhead를 거의 순수 4bit로
- **Paged Optimizer** — NVIDIA Unified Memory로 optimizer state를 CPU-GPU 간 이동, OOM 방지, Adam의 32GB state를 효율 관리
- **QLoRA vs LoRA vs Full FT — 성능 비교** — Alpaca, Vicuna 벤치마크에서 성능 gap 미미, memory 16배 절약, Guanaco 33B/65B의 성공

### Chapter 5: Mixture of Experts (MoE) (5개 문서)
- **MoE의 기본 수식** — Gating $g(x) = \text{Softmax}(W_g x)$, output $y = \sum_i g_i(x) E_i(x)$, Shazeer 2017의 outrageously large NN, Switch Transformer (Fedus 2022)
- **Top-$k$ Routing과 Sparse Activation** — Top-1 (Switch), Top-2 (Mixtral), 각 token이 $k$ expert만 활성화, FFN 교체가 일반적, attention shared
- **Load Balancing Loss** — $L_{\text{aux}} = \alpha \cdot N \sum_i f_i P_i$, $f_i$는 expert $i$에 routed된 token 비율, $P_i$는 gating prob 평균, 둘을 곱해 균등화 유도
- **Expert Capacity와 Token Dropping** — Capacity $C = \lceil k \cdot T/N \rceil \cdot c_f$, $c_f$ capacity factor (보통 1.25), 초과 token drop, DP를 위한 fixed shape
- **MoE Scaling Law (Clark 2022, Krajewski 2024)** — Dense와 다른 scaling, sparsity ratio의 영향, active params vs total params, compute-matched에서 MoE 우위

### Chapter 6: Flash Attention과 Attention 최적화 (5개 문서)
- **Attention의 Memory Bottleneck** — 표준 attention이 $O(T^2)$ 메모리 (HBM), softmax 계산이 HBM-bound, compute/memory 비율 문제
- **Flash Attention 1 (Dao 2022)** — Tiling + recomputation, SRAM에서 block 단위 softmax, backward는 recompute, 정확 (근사 아님), 2-4× 빠름
- **Flash Attention 2 (Dao 2023)** — Work partitioning 개선, causal mask 최적화, backward pass 병렬화, FA1 대비 2× 추가 향상
- **Flash Attention 3 (Shah 2024)** — H100의 asynchrony (TMA, WGMMA) 활용, FP8 지원, Hopper 아키텍처 최적화
- **Alternative — Paged/Ring/Linear Attention** — PagedAttention (vLLM, Kwon 2023)의 KV cache 파편화, Ring Attention (Liu 2023) long context, Mamba의 linear complexity

### Chapter 7: Speculative Decoding과 추론 가속 (4개 문서)
- **Autoregressive Decoding의 Bottleneck** — $T$ tokens 생성에 $T$ forward passes, memory-bound (KV cache read/write), compute underutilization
- **Speculative Decoding (Leviathan 2023, Chen 2023)** — 작은 draft model $q$로 $K$ tokens 초안, target model $p$로 일괄 verify, acceptance prob $\min(1, p(x)/q(x))$
- **Losslessness 증명 — Rejection Sampling** — Accept sample이 $p$ 분포 따름 증명: $P(\text{sample} = x) = q(x) \min(1, p(x)/q(x))$ + reject 시 resample이 결국 $p(x)$
- **Medusa, EAGLE, Lookahead** — Medusa (Cai 2024)의 multiple heads로 draft, EAGLE (Li 2024)의 feature-level draft, Lookahead (Fu 2024)의 n-gram guess, 각 approach의 trade-off

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **35개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 효율화 기법이 실전에 필수인가
## 📐 수학적 선행 조건 (LLM Pretraining, Transformer, LA, Info 참조)
## 📖 직관적 이해
   — Memory·Compute·Latency 감소의 geometric 직관
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — LoRA low-rank, NF4 quantile, Spec Decoding losslessness
## 💻 PyTorch 구현 검증
   — LoRA 바닥부터 구현, QLoRA 재현
   — 간단한 MoE layer, Flash Attention 비교
   — Speculative Decoding pseudocode
## 🔗 실전 활용
   — 언제 어느 기법 선택 (학습 vs 추론)
## ⚖️ 가정과 한계
   — Quantization error 누적, MoE instability
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **LoRA rank sweep** — $r$ 변화에 따른 성능·메모리 plot
2. **Quantization error 분석** — NF4 vs INT4 vs FP4의 recon error 비교
3. **MoE load balancing 측정** — 훈련 중 expert utilization 분포
4. **Flash Attention 속도 측정** — sequence length별 standard vs Flash
5. **Speculative acceptance rate** — 다양한 draft model 크기에서 $K$ token 수락률
6. **End-to-end pipeline** — 7B 모델을 QLoRA로 fine-tune → GPTQ 4bit → vLLM 서빙

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
transformers==4.36.0
peft==0.7.0
bitsandbytes==0.41.0
auto-gptq==0.6.0
vllm==0.2.7            # 추론
flash-attn==2.4.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (LoRA 바닥부터 + NF4 quantization + Speculative Decoding)
import torch
import torch.nn as nn
import torch.nn.functional as F

# 1. LoRA 바닥부터
class LoRALinear(nn.Module):
    def __init__(self, in_features, out_features, r=8, alpha=16):
        super().__init__()
        # Base weight (frozen)
        self.base = nn.Linear(in_features, out_features, bias=False)
        for p in self.base.parameters():
            p.requires_grad = False
        # LoRA: A down projection, B up projection
        self.A = nn.Parameter(torch.randn(r, in_features) * 0.01)
        self.B = nn.Parameter(torch.zeros(out_features, r))  # init B=0
        self.scaling = alpha / r
    def forward(self, x):
        # W'x = Wx + alpha/r · B(Ax)
        return self.base(x) + self.scaling * F.linear(F.linear(x, self.A), self.B)

# Parameter count
d, k, r = 4096, 4096, 8
original = d * k
lora = r * (d + k)
print(f'Original: {original}, LoRA: {lora}, Reduction: {original/lora:.1f}x')
# 4096² = 16.7M → 8×(4096+4096) = 65K (250× 감소)

# 2. NF4 quantization
def nf4_levels():
    """Normal Float 4bit: 정규분포 quantile"""
    from scipy.stats import norm
    # 16 levels (4bit)
    # Symmetric around 0
    probs = torch.linspace(0.5/16, 1 - 0.5/16, 16)
    levels = torch.tensor([norm.ppf(p.item()) for p in probs])
    # Normalize to [-1, 1]
    levels = levels / levels.abs().max()
    return levels

nf4 = nf4_levels()
print(f'NF4 levels: {nf4}')
# Compare to INT4: torch.linspace(-1, 1, 16)
int4 = torch.linspace(-1, 1, 16)

# Quantize Gaussian tensor with both
x = torch.randn(10000)
x_norm = x / x.abs().max()

# Find nearest quantization level
def quantize(x, levels):
    # For each x, find closest level
    dists = (x.unsqueeze(-1) - levels.unsqueeze(0)).abs()
    idx = dists.argmin(-1)
    return levels[idx]

x_nf4 = quantize(x_norm, nf4) * x.abs().max()
x_int4 = quantize(x_norm, int4) * x.abs().max()

mse_nf4 = ((x - x_nf4)**2).mean()
mse_int4 = ((x - x_int4)**2).mean()
print(f'NF4 MSE: {mse_nf4:.6f}, INT4 MSE: {mse_int4:.6f}')
# NF4 < INT4 for Gaussian weights

# 3. Speculative Decoding 수학 검증
def speculative_sample(p, q):
    """
    p: target distribution
    q: draft distribution
    Returns: sample from p (exactly)
    """
    x = torch.multinomial(q, 1).item()
    accept_prob = min(1.0, p[x].item() / (q[x].item() + 1e-10))
    if torch.rand(1).item() < accept_prob:
        return x  # accept
    else:
        # Reject: sample from residual (p - q)_+
        residual = torch.clamp(p - q, min=0)
        residual = residual / residual.sum()
        return torch.multinomial(residual, 1).item()

# Empirical verification: 결과가 p 분포 따름
V = 10
p = torch.softmax(torch.randn(V), dim=0)
q = torch.softmax(torch.randn(V), dim=0)

N = 100000
samples = torch.tensor([speculative_sample(p, q) for _ in range(N)])
emp_p = torch.bincount(samples, minlength=V).float() / N

import matplotlib.pyplot as plt
plt.bar(range(V), p.numpy(), alpha=0.5, label='Target p')
plt.bar(range(V), emp_p.numpy(), alpha=0.5, label='Speculative samples')
plt.title('Speculative Decoding preserves target distribution')
plt.legend(); plt.show()
# 두 분포 일치 → losslessness 검증
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "LLM Pretraining, Transformer, LA, Info 선행 필수"
   - 4축 효율화 (Parameter·Memory·Compute·Latency)
   - 실전 recipe (QLoRA+Flash+vLLM)
3. **챕터별 문서 작성**: PEFT → LoRA → Quantization → QLoRA → MoE → Flash → Speculative

---

## 📚 참고 자료

- **LoRA: Low-Rank Adaptation of Large Language Models** (Hu et al. 2021)
- **QLoRA: Efficient Finetuning of Quantized LLMs** (Dettmers et al. 2023)
- **DoRA** (Liu et al. 2024)
- **Parameter-Efficient Transfer Learning** (Houlsby et al. 2019) — Adapter
- **Prefix-Tuning** (Li & Liang 2021)
- **Intrinsic Dimensionality Explains Effectiveness of LM Fine-tuning** (Aghajanyan 2020)
- **LLM.int8()** (Dettmers et al. 2022)
- **GPTQ** (Frantar et al. 2023)
- **AWQ** (Lin et al. 2023)
- **SmoothQuant** (Xiao et al. 2023)
- **Switch Transformer** (Fedus et al. 2022)
- **Mixtral of Experts** (Jiang et al. 2024)
- **ST-MoE: Designing Stable and Transferable Sparse Expert Models** (Zoph 2022)
- **FlashAttention** (Dao et al. 2022)
- **FlashAttention-2** (Dao 2023)
- **PagedAttention / vLLM** (Kwon et al. 2023)
- **Fast Inference from Transformers via Speculative Decoding** (Leviathan et al. 2023)
- **Accelerating Large Language Model Decoding with Speculative Sampling** (Chen et al. 2023)
- **Medusa** (Cai et al. 2024)
- **EAGLE** (Li et al. 2024)

---

## 💡 핵심 분석 대상

```
LLM Efficiency의 4대 축

───── 1. Parameter (훈련 효율) ─────

Full FT 비용:
  7B FP32 = 28GB weights
  + 56GB Adam state
  = 84GB+ (A100 80GB 불가)

PEFT 분류:
  ├── Additive: Adapter, LoRA
  ├── Selective: BitFit, layer-wise
  └── Reparameterization: LoRA, DoRA

LoRA (Hu 2021):
  W + ΔW = W + BA
  B ∈ ℝ^{d×r}, A ∈ ℝ^{r×k}
  
  Param reduction:
    O(dk) → O(r(d+k))
    d=k=4096, r=8: 250× ↓
  
  Init:
    A ~ N(0, σ), B = 0
    → ΔW = 0 at start
  
  Scaling:
    output = Wx + (α/r) BAx
    α = 2r 권장 (rsLoRA: α/√r)

Intrinsic Dim (Aghajanyan 2020):
  Pretrained LM의 task manifold
  낮은 d_intrinsic에 존재
  → LoRA의 이론적 근거

DoRA (Liu 2024):
  W = m · V/‖V‖
  LoRA for direction, m separate
  → full FT에 가까운 성능

Task Vectors (Ilharco 2023):
  ΔW = W_ft - W_pt
  산술 연산 (add/subtract)
  model editing

───── 2. Memory (양자화) ─────

Quantization 기본:
  x_q = round(x/s) + z
  s: scale, z: zero-point

INT8 (Dettmers 2022):
  LLM.int8()
  outlier 채널 분리 (FP16)
  mixed precision

GPTQ (Frantar 2023):
  min ‖WX - Ŵ X‖²
  Hessian-based update
  OBQ/OBS의 LLM 확장
  4bit ~ full precision

AWQ (Lin 2023):
  Activation-aware
  큰 activation 채널 보존
  W/s, X·s transformation

SmoothQuant (Xiao 2023):
  Activation outlier → weight
  W8A8 가능
  X/s, W·s (AWQ와 반대)

NF4 (Dettmers 2023):
  q_i = Φ^{-1}(i/(2^k - 1))
  Gaussian quantile
  INT4보다 낮은 recon error
  (Gaussian weight 가정)

QLoRA (Dettmers 2023):
  Base: NF4 (frozen)
  LoRA: BF16 (trainable)
  Double Quant: 상수도 양자화
  Paged Optimizer: CPU offload
  
  → 65B on 48GB GPU

───── 3. Compute (MoE) ─────

MoE (Shazeer 2017):
  y = Σ g_i(x) E_i(x)
  g(x) = softmax(W_g x)

Top-k routing:
  Switch (top-1, Fedus 2022)
  Mixtral (top-2)
  Expert = FFN only

Load Balancing Loss:
  L_aux = α · N · Σ f_i P_i
    f_i: frac tokens routed to i
    P_i: avg gating prob
  
  CV minimization
  균등 utilization 유도

Expert Capacity:
  C = ⌈k · T/N⌉ · c_f
  c_f ∈ [1, 2] capacity factor
  초과 token drop

Scaling Law (Clark 2022):
  Dense와 다름
  Active ≠ Total params
  sparsity ratio 영향

───── 4. Latency (추론 가속) ─────

Attention의 Memory Bottleneck:
  O(T²) HBM 사용
  Softmax compute/mem 비율 낮음

Flash Attention 1 (Dao 2022):
  Tiling + recomputation
  SRAM에서 block softmax
  Exact, not approx
  2-4× 빠름

Flash Attention 2 (Dao 2023):
  Work partitioning 개선
  Causal mask 최적화
  FA1 대비 2× 추가

Flash Attention 3 (Shah 2024):
  Hopper (H100) 최적화
  TMA, WGMMA async
  FP8 지원

PagedAttention (vLLM):
  KV cache fragmentation 해결
  OS page 유추
  고 throughput 서빙

Speculative Decoding (Leviathan 2023):
  Draft q로 K tokens
  Target p로 일괄 verify
  
  Accept: min(1, p(x)/q(x))
  Reject: residual (p-q)_+ 에서 sample

Losslessness 증명:
  P(sample = x) = q(x) · min(1, p(x)/q(x))
                + reject · (p-q)_+
                = p(x)
  Rejection sampling theorem

Variants:
  Medusa: multiple heads
  EAGLE: feature-level
  Lookahead: n-gram cache

───── 통합 Recipe ─────

훈련:
  QLoRA (NF4 + LoRA + BF16)
  + Flash Attention 2
  + Gradient Accumulation
  + Paged Optimizer

추론:
  GPTQ 4bit weights
  + Flash Attention 3
  + vLLM (PagedAttention)
  + Speculative Decoding
  + Tensor Parallel

───── 레포 간 연결 ─────

LLM Pretraining (앞):
  기본 아키텍처·scale

Transformer (Layer 3):
  Attention, FFN, KV cache

Linear Algebra (Layer 0):
  SVD, low-rank
  Kronecker product

Information Theory (Layer 0):
  Entropy, quantization
  Rate-distortion

Optimization (Layer 2):
  Adam state memory
  Gradient accumulation

LLM Inference (다음):
  Serving system
  PagedAttention, 
  Continuous batching
  vLLM architecture
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·기법 (3~4줄)
- 전체 문서 개수 확인 (35개 목표)
- Python + PyTorch + HF + vLLM 실험 환경
- LLM Pretraining, Transformer, LA, Info 레포 참조 관계
- Inference·Serving으로 이어지는 현대 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
