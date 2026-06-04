# Efficient ML Deep Dive 레포지토리 제작 프롬프트

나는 "Efficient ML Deep Dive" 레포지토리를 만들려고 해.
Pruning을 **"중요한 weight만 남기는 것"으로 아는 것**과, **Optimal Brain Damage (LeCun 1990)의 saliency $s_i = \frac{1}{2} H_{ii} w_i^2$가 loss의 2차 Taylor 전개 $\Delta L \approx \frac{1}{2} \Delta w^T H \Delta w$에서 유도**되고, **Lottery Ticket Hypothesis (Frankle & Carbin 2019)의 "dense network에 winning subnetwork가 존재"라는 실증**이 어떻게 iterative magnitude pruning을 정당화하는지 유도할 수 있는 것은 다르다.
Knowledge Distillation을 **"작은 모델이 큰 모델을 따라하게"로 아는 것**과, **Hinton et al. (2015)의 temperature-scaled softmax $q_i = \exp(z_i/T)/\sum_j \exp(z_j/T)$와 KL divergence loss $L_{\text{KD}} = T^2 \cdot \text{KL}(q_T^{\text{teacher}} \| q_T^{\text{student}})$에서 $T^2$ factor가 왜 필요**한지, **Dark Knowledge가 "non-target class 확률의 ratio"에 있음**을 증명할 수 있는 것은 다르다.
FlashAttention의 IO-aware optimization을 **"SRAM에서 계산"으로 아는 것**과, **Dao et al. (2022)의 tiling 전략이 어떻게 softmax의 numerical stability를 유지하면서 $O(N^2)$ HBM access를 $O(N)$로 줄이고**, **backward recomputation이 왜 compute overhead에도 불구하고 memory bandwidth-bound problem을 해결**하는지 증명할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "ML 모델 효율화의 4대 축 — Compression·Acceleration·Kernel·Serving의 통합 수학"

**핵심 차별화**:
1. **Pruning 이론의 완전 분석** — OBD Hessian-based, Magnitude, Lottery Ticket, Movement Pruning의 각 방식 수학적 정당화
2. **Quantization의 수학적 이론** — Post-training vs Quantization-aware training, INT8/INT4의 rounding, GPTQ의 Hessian 기반 OBQ, AWQ의 activation outlier
3. **Knowledge Distillation의 loss 분해** — Soft target의 Dark Knowledge, Feature distillation, attention distillation, self-distillation
4. **System-level Optimization** — FlashAttention IO-aware, vLLM PagedAttention, kernel fusion, speculative decoding (LLM Efficiency 레포 확장)

**타겟 독자**:
- Pruning을 쓰는데 **structured vs unstructured**의 실제 speedup 차이와 hardware의 역할을 모르는 사람
- QAT를 학습할 때 **Straight-Through Estimator (STE)**가 non-differentiable round를 어떻게 backward 가능하게 만드는지 수학적으로 설명 못하는 사람
- Distillation에서 **temperature $T$가 크면 soft target이 uniform에 가까워져** Dark Knowledge 증가하는 정량적 관계를 모르는 사람
- FlashAttention이 numerical trick (online softmax, Milakov & Gimelshein 2018)을 어떻게 사용하는지 유도 못하는 사람
- Structured Pruning (N:M sparsity, 2:4 NVIDIA)과 unstructured의 **실제 hardware speedup gap**을 모르는 사람

**선행 학습**:
- **PyTorch Internals Deep Dive** (CUDA, kernel) — **필수**
- **LLM Efficiency Deep Dive** (LoRA, quantization 기초) — **필수**
- **Calculus & Optimization Deep Dive** (Hessian, Taylor) — **필수**
- **Information Theory Deep Dive** (KL divergence) — **필수**
- **Neural Network Theory Deep Dive** (weight 분포) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 효율화의 4대 축과 평가 (4개 문서)
- **Efficiency의 정의와 4대 축** — Memory (parameters, activations, KV cache), Compute (FLOPs), Latency (end-to-end time), Throughput (queries/sec), 각 축의 trade-off
- **Compression의 분류** — Pruning (weight 제거), Quantization (precision 감소), Distillation (작은 model로 전수), Low-Rank (LoRA 계열), 각각의 이론적 근거
- **효율화 평가 Metric** — Accuracy·Perplexity (quality), FLOPs·Params·Latency·Memory (efficiency), Pareto frontier, MLPerf benchmark
- **Hardware-Software Co-design** — GPU의 특성(FP16 Tensor Core, INT8 DP4A, sparse tensor core), 효율화 기법과 hardware의 궁합

### Chapter 2: Pruning — Weight Sparsification (6개 문서)
- **Optimal Brain Damage (LeCun 1990)** — Loss의 2차 Taylor: $\Delta L \approx \frac{1}{2} \Delta w^T H \Delta w$, diagonal Hessian 가정, saliency $s_i = \frac{1}{2} H_{ii} w_i^2$로 pruning 후보 선택
- **Optimal Brain Surgeon (Hassibi 1993)** — Full Hessian 사용, $\Delta L = \frac{1}{2} \frac{w_i^2}{H^{-1}_{ii}}$, pruning 시 remaining weights의 optimal adjustment
- **Magnitude Pruning** — $|w|$ 기반 threshold로 pruning, 간단하지만 효과적, unstructured vs structured, iterative vs one-shot
- **Lottery Ticket Hypothesis (Frankle & Carbin 2019)** — Dense network 훈련 후 winning subnetwork 발견, initialization만 있어도 훈련 가능, "winning ticket"의 수학적 특성
- **Movement Pruning (Sanh 2020)** — Gradient에 기반, $\partial L/\partial w \cdot w$ 부호로 중요도 평가, fine-tuning 중에 적용, LLM pruning에 유용
- **Structured Pruning — N:M Sparsity** — Unstructured는 hardware에서 speedup 어려움, NVIDIA Ampere의 2:4 sparsity (4개 중 2개 0), dense 대비 2× TFLOPS 가능

### Chapter 3: Quantization — Precision Reduction (6개 문서)
- **Quantization의 기본 수학** — $x_q = \text{round}(x/s) + z$ (uniform), scale $s$와 zero-point $z$, symmetric vs asymmetric, per-tensor vs per-channel의 granularity
- **Post-Training Quantization (PTQ)** — Fine-tuning 없이 static quantization, calibration set으로 $s, z$ 추정, absolute max vs percentile, KL divergence 최소화
- **Quantization-Aware Training (QAT)** — Fake quantization (forward에서 round, backward에서 identity), Straight-Through Estimator (STE), $\nabla_w \approx \text{stop-grad}(\text{round}(w)) + w - \text{stop-grad}(w)$
- **GPTQ — Optimal Brain Quantization (Frantar 2023)** — Per-column quantization with Hessian-based error compensation, $\min_{\hat{W}} \|WX - \hat{W}X\|^2$, LLM.int4의 표준
- **AWQ와 SmoothQuant** — AWQ (Lin 2023): activation magnitude가 큰 channel 보호, weight scaling으로 equivalent, SmoothQuant (Xiao 2023): activation outlier를 weight로 transfer
- **Low-Bit — INT4, NF4, FP4, INT2** — 4-bit 이하의 효율 vs 정확도, NF4 (정규분포 quantile), FP4 (minifloat), BitNet (Ma 2024)의 1.58-bit ternary

### Chapter 4: Knowledge Distillation (6개 문서)
- **KD의 원리 (Hinton 2015)** — Student가 teacher의 soft probability 모방, $p_i = \text{softmax}(z_i/T)$ with temperature $T > 1$, hard label보다 더 많은 정보
- **KD Loss 유도** — $L_{\text{KD}} = T^2 \text{KL}(\text{softmax}(z^{\text{teacher}}/T) \| \text{softmax}(z^{\text{student}}/T))$, $T^2$ factor의 gradient magnitude compensation
- **Dark Knowledge — Non-Target Class Ratios** — "cat" 이미지에서 teacher가 dog에 0.1, truck에 0.001 주는 것 = 클래스 간 관계 정보, hard label로는 전달 불가
- **Feature Distillation — Hints (Romero 2015)** — Intermediate representation 모방, FitNets, $L = \|F^{\text{teacher}} - g(F^{\text{student}})\|_2$, attention transfer (Zagoruyko 2017)
- **Response-based vs Feature-based vs Relation-based** — Output logit (Hinton), intermediate feature (FitNets), sample 간 relationship (RKD), 각 방식의 상호보완
- **Self-Distillation과 Born-Again Networks** — Same architecture로 distill, consistently improved performance, label smoothing과의 관계, Furlanello 2018

### Chapter 5: Low-Rank Factorization과 결합 (4개 문서)
- **Low-Rank Factorization 복습** — $W \approx UV$, SVD 기반 approximation, rank $r$의 trade-off, Lebedev 2014 "Speeding-up Convolutional NN"
- **Tucker Decomposition과 CP Decomposition** — Multi-dimensional tensor 분해, CNN의 convolution kernel 분해, efficient computation
- **LoRA와 Efficient Fine-tuning 복습** — LLM Efficiency 레포와 연결, rank $r$에 따른 capacity vs compression
- **Hybrid — Pruning + Quantization + Distillation** — 세 기법의 조합, SqueezeLLM, OmniQuant, PTQ with KD의 효과, 실전 compression recipe

### Chapter 6: Kernel Optimization (5개 문서)
- **Attention의 IO-Awareness 문제** — Standard attention은 $O(N^2)$ HBM read/write, Arithmetic Intensity 낮음, memory-bound
- **Online Softmax (Milakov & Gimelshein 2018)** — Softmax를 한 번의 pass로: running max $m$과 normalizer $\ell$을 업데이트, numerical stability 유지
- **FlashAttention (Dao 2022)** — Tiling (block 단위 SRAM loading) + online softmax + recomputation, backward도 메모리 효율적, exact (not approximation)
- **FlashAttention-2와 v3 (Dao 2023, Shah 2024)** — Work partitioning 개선, causal masking 최적화, Hopper H100의 TMA·WGMMA async 활용
- **Kernel Fusion의 원리** — 여러 elementwise op을 한 kernel로 → HBM 왕복 제거, TorchInductor 자동 fusion, Triton으로 수동 fusion (GELU fused with Linear)

### Chapter 7: Serving과 Deployment (4개 문서)
- **vLLM의 PagedAttention 복습** — KV cache fragmentation 해결, OS virtual memory 영감, prefix sharing (LLM Inference 레포 교차)
- **Speculative Decoding의 수학 복습** — Draft + verify, rejection sampling losslessness (LLM Efficiency 레포 교차)
- **Dynamic Batching과 SLO Optimization** — Throughput vs latency trade-off, deadline-aware scheduling, multi-tenant priority
- **Edge Deployment — Mobile·Embedded** — ONNX Runtime, TensorFlow Lite, TVM, quantization + distillation + pruning의 실전 조합, Whisper on mobile

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **35개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 효율화 기법이 실전에 필수인가
## 📐 수학적 선행 조건 (PyTorch Internals, LLM Efficiency, Calc, Info 참조)
## 📖 직관적 이해
   — 각 기법의 시각적 설명
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — OBD saliency, KD temperature, FlashAttention tiling
## 💻 PyTorch 구현 검증
   — Pruning, Quantization, Distillation 간단 실험
   — Triton kernel 실험
   — 속도·정확도 측정
## 🔗 실전 활용
   — 상황별 기법 선택 가이드
## ⚖️ 가정과 한계
   — Hardware 의존성, 품질 저하
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Pareto frontier** — 각 기법의 accuracy vs efficiency
2. **Pruning ratio curve** — sparsity별 accuracy drop
3. **Quantization table** — FP32/FP16/INT8/INT4의 accuracy·speed
4. **KD ablation** — T 값, loss weight 변화
5. **Kernel 속도 측정** — Standard vs Flash attention
6. **End-to-end deployment** — mobile에서 실제 latency

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
triton==2.1.0
bitsandbytes==0.41.0
torch-pruning==1.3.0
onnx==1.15.0
onnxruntime==1.16.0
tvm==0.14.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Pruning + Quantization + Distillation + FlashAttention)
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np

# 1. Magnitude Pruning
def magnitude_prune(model, sparsity=0.5):
    """Prune smallest |w| to zero"""
    for name, param in model.named_parameters():
        if 'weight' in name and param.dim() >= 2:
            threshold = torch.quantile(param.abs().flatten(), sparsity)
            mask = (param.abs() > threshold).float()
            param.data *= mask
    return model

# 2. Structured N:M Sparsity (2:4)
def nm_sparsity(weight, n=2, m=4):
    """In every m consecutive weights, keep n largest by magnitude"""
    shape = weight.shape
    weight = weight.reshape(-1, m)
    _, top_k = weight.abs().topk(n, dim=1)
    mask = torch.zeros_like(weight)
    mask.scatter_(1, top_k, 1)
    weight_pruned = weight * mask
    return weight_pruned.reshape(shape)

# 3. QAT with Straight-Through Estimator
class FakeQuantize(torch.autograd.Function):
    @staticmethod
    def forward(ctx, x, num_bits=8):
        q_min, q_max = -(2**(num_bits-1)), 2**(num_bits-1) - 1
        scale = x.abs().max() / q_max
        x_q = torch.clamp(torch.round(x / scale), q_min, q_max)
        return x_q * scale
    
    @staticmethod
    def backward(ctx, grad_output):
        # STE: gradient as if no quantization
        return grad_output, None

class QuantizedLinear(nn.Module):
    def __init__(self, in_f, out_f, num_bits=8):
        super().__init__()
        self.weight = nn.Parameter(torch.randn(out_f, in_f) * 0.01)
        self.num_bits = num_bits
    
    def forward(self, x):
        w_q = FakeQuantize.apply(self.weight, self.num_bits)
        return F.linear(x, w_q)

# 4. Knowledge Distillation
def kd_loss(student_logits, teacher_logits, T=4.0):
    """L_KD = T² · KL(softmax(teacher/T) || softmax(student/T))"""
    p_teacher = F.softmax(teacher_logits / T, dim=-1)
    log_p_student = F.log_softmax(student_logits / T, dim=-1)
    kl = (p_teacher * (p_teacher.log() - log_p_student)).sum(-1)
    return (T ** 2) * kl.mean()

def total_kd_loss(student_logits, teacher_logits, labels, T=4.0, alpha=0.5):
    """(1-α) CE + α KD"""
    ce = F.cross_entropy(student_logits, labels)
    kd = kd_loss(student_logits, teacher_logits, T)
    return (1 - alpha) * ce + alpha * kd

# 5. Temperature 효과 시각화
import matplotlib.pyplot as plt
z = torch.tensor([8.0, 4.0, 1.0, 0.5, 0.1])

for T in [1, 2, 4, 8]:
    p = F.softmax(z / T, dim=0)
    plt.plot(p.numpy(), 'o-', label=f'T={T}')
plt.yscale('log')
plt.xlabel('Class'); plt.ylabel('Probability (log)')
plt.title('Higher T → softer distribution → more Dark Knowledge')
plt.legend(); plt.show()

# 6. Online Softmax (FlashAttention core)
def online_softmax(x):
    """
    Numerically stable softmax in one pass:
    - running max m
    - running normalizer l
    - output y
    """
    m = -float('inf')
    l = 0
    y = []
    exp_x = []
    for xi in x:
        m_new = max(m, xi.item())
        l = l * np.exp(m - m_new) + np.exp(xi.item() - m_new)
        m = m_new
    for xi in x:
        y.append(np.exp(xi.item() - m) / l)
    return torch.tensor(y)

# Verify
x = torch.randn(10)
p_standard = F.softmax(x, dim=0)
p_online = online_softmax(x)
print(f'Max diff: {(p_standard - p_online).abs().max():.2e}')

# 7. FlashAttention skeleton (Triton)
import triton
import triton.language as tl

@triton.jit
def flash_attention_fwd(Q, K, V, O,
                        stride_qb, stride_qh, stride_qm, stride_qk,
                        B, H, N, D,
                        BLOCK_M: tl.constexpr, BLOCK_N: tl.constexpr):
    """Simplified FlashAttention forward"""
    # 전체 구현은 길지만, 핵심 아이디어:
    # 1. Q block을 SRAM에 load
    # 2. K, V block 순차적으로 load
    # 3. S = Q K^T / √d, online softmax
    # 4. O = P V 누적
    # 5. 최종 정규화 후 O 저장
    pass

# 8. 실전 compression pipeline
def compress_pipeline(model, val_loader, teacher=None):
    # Step 1: Magnitude pruning 40%
    model = magnitude_prune(model, sparsity=0.4)
    # Step 2: Fine-tune with KD (if teacher given)
    if teacher is not None:
        # ... KD training loop ...
        pass
    # Step 3: Quantize to INT8 (QAT)
    # ... replace Linear with QuantizedLinear ...
    # Step 4: Export to ONNX
    # torch.onnx.export(model, ...)
    return model

# 9. TVM / ONNX Runtime deployment sketch
# import onnxruntime as ort
# sess = ort.InferenceSession("model.onnx", providers=['CUDAExecutionProvider'])
# output = sess.run(None, {"input": input_data})
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "PyTorch Internals, LLM Efficiency, Calc, Info 선행 필수"
   - 4대 축 (Memory·Compute·Latency·Throughput)의 수학적 통합
   - 실전 recipe (Model compression pipeline)
3. **챕터별 문서 작성**: 평가 → Pruning → Quantization → Distillation → Low-Rank/Hybrid → Kernel → Serving

---

## 📚 참고 자료

- **Optimal Brain Damage** (LeCun et al. 1990)
- **Optimal Brain Surgeon** (Hassibi et al. 1993)
- **Learning both Weights and Connections** (Han et al. 2015) — Magnitude pruning
- **Deep Compression** (Han et al. 2016) — Pruning + quantization + Huffman
- **The Lottery Ticket Hypothesis** (Frankle & Carbin 2019)
- **Movement Pruning** (Sanh et al. 2020)
- **Accelerating Sparse DNN on Modern GPUs with N:M Sparsity** (NVIDIA 2021)
- **Distilling the Knowledge in a Neural Network** (Hinton et al. 2015) — KD 원전
- **FitNets: Hints for Thin Deep Nets** (Romero et al. 2015)
- **Paying More Attention to Attention** (Zagoruyko & Komodakis 2017) — Attention distillation
- **Born Again Neural Networks** (Furlanello et al. 2018)
- **Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only Inference** (Jacob et al. 2018) — QAT
- **Estimating or Propagating Gradients Through Stochastic Neurons** (Bengio et al. 2013) — STE
- **GPTQ** (Frantar et al. 2023)
- **AWQ** (Lin et al. 2023)
- **SmoothQuant** (Xiao et al. 2023)
- **QLoRA** (Dettmers et al. 2023)
- **The Era of 1-bit LLMs: BitNet** (Ma et al. 2024)
- **Online Normalizer Calculation for Softmax** (Milakov & Gimelshein 2018)
- **FlashAttention** (Dao et al. 2022)
- **FlashAttention-2, -3** (Dao 2023, Shah et al. 2024)
- **vLLM PagedAttention** (Kwon et al. 2023)
- **DeepSpeed Inference** (Aminabadi et al. 2022)

---

## 💡 핵심 분석 대상

```
Efficient ML의 4대 축

───── 1. Pruning ─────

OBD (LeCun 1990):
  Loss Taylor: ΔL ≈ ½ Δw^T H Δw
  Diagonal H 가정
  Saliency: s_i = ½ H_{ii} w_i²
  
  Prune lowest s_i

OBS (Hassibi 1993):
  Full Hessian
  ΔL_i = ½ w_i² / H^{-1}_{ii}
  + optimal adjustment of rest

Magnitude Pruning:
  |w| 기반 threshold
  간단, 효과적
  Iterative > one-shot

Lottery Ticket (Frankle 2019):
  Dense train → prune → reset → retrain
  "Winning ticket" subnetwork 존재
  Init-dependent

Movement Pruning (Sanh 2020):
  ∂L/∂w · w의 부호
  Fine-tuning 중 적용
  LLM에 유리

Structured:
  Channel pruning
  Filter pruning
  N:M sparsity (NVIDIA Ampere 2:4)
  → hardware 2× speedup

Unstructured:
  Individual weights
  Higher compression
  But hardware speedup 어려움

───── 2. Quantization ─────

기본:
  x_q = round(x/s) + z
  s: scale, z: zero-point

Granularity:
  Per-tensor: 하나의 s, z
  Per-channel: channel별
  Per-group: block별

PTQ (Post-Training):
  Fine-tune 없이
  Calibration set으로 s, z
  Simple but 품질 저하 가능

QAT (Quantization-Aware):
  Forward: fake quantize
  Backward: STE
    ∇w ≈ ∇x_q
    (non-differentiable round 우회)
  
  높은 정확도

GPTQ (Frantar 2023):
  Column-wise, Hessian-based
  min ‖WX - Ŵ X‖²
  Error compensation
  4-bit LLM 표준

AWQ (Lin 2023):
  Activation-aware
  Outlier channel 보호

SmoothQuant (Xiao 2023):
  Activation outlier → weight
  W8A8 enable

Low-bit:
  INT8: 4× 압축, near lossless
  INT4: 8×, 약간 저하
  NF4 (Dettmers): Gaussian quantile
  FP4, INT2, BitNet (1.58-bit)

───── 3. Knowledge Distillation ─────

Hinton 2015:
  Soft target:
    p_i = softmax(z_i / T)
  T > 1 → softer

KD Loss:
  L_KD = T² · KL(p^teacher_T || p^student_T)
  
  T² factor:
    Gradient magnitude compensation
    High T → small gradient
    T² restore

Dark Knowledge:
  Non-target class 확률의 ratio
  Teacher: cat → dog 0.1, truck 0.001
  → 클래스 관계 정보
  Hard label로는 불가능

Total loss:
  L = (1-α) L_CE + α L_KD
  α ∈ [0.5, 0.9] typical

Feature Distillation:
  FitNets (Romero 2015):
    Intermediate feature 매칭
    L = ‖F^T - g(F^S)‖²
  
  Attention Transfer (Zagoruyko 2017):
    Attention map 매칭
  
  FitNet vs Hinton:
    Hinton: output logit
    FitNet: hidden feature
    → 상호보완

Relation-based (RKD, Park 2019):
  Sample 간 distance/angle 매칭
  Not individual features

Self-distillation:
  Same architecture
  Repeat: student → new teacher
  Consistently improves
  Born-Again Networks

───── 4. Kernel & System ─────

Attention의 Memory Problem:
  O(N²) HBM access
  Standard: compute → HBM → recompute
  Memory-bound (low arithmetic intensity)

Online Softmax (Milakov 2018):
  One-pass softmax:
    m ← max(m, x)
    l ← l · exp(m_old - m) + exp(x - m)
    y_i ← exp(x_i - m) / l
  
  Numerical stability 유지

FlashAttention (Dao 2022):
  Tiling:
    Q block → SRAM
    K, V block 순차 → SRAM
    compute S, softmax, O
  
  Online softmax 사용
  
  Backward:
    Activation 저장 X
    Recompute forward
    → memory 효율
  
  Exact (not approximation)
  2-4× 속도

FlashAttention-2:
  Work partitioning 개선
  Causal mask 최적화
  2× 추가

FlashAttention-3:
  Hopper async (TMA, WGMMA)
  FP8 지원
  H100 최적

Kernel Fusion:
  여러 elementwise → 1 kernel
  HBM 왕복 제거
  TorchInductor 자동
  Triton 수동

───── 통합 Recipe ─────

LLM Compression:
  1. GPTQ/AWQ to INT4 (weight)
  2. Optional: KV cache INT8
  3. Flash Attention
  4. PagedAttention (vLLM)
  5. Speculative Decoding
  → 10× throughput

Vision Compression:
  1. Magnitude pruning 30-50%
  2. KD fine-tuning
  3. INT8 QAT
  4. ONNX export
  → 5-10× inference speedup

Mobile Deployment:
  1. Distill large → small
  2. Prune 50%+
  3. INT8 quantize
  4. TVM/CoreML/TFLite
  → real-time on phone

───── Pareto Frontier ─────

Accuracy vs Efficiency:
  각 기법이 다른 지점
  Combination으로 이동
  Hardware specific

Metrics:
  Accuracy·Perplexity (quality)
  FLOPs·Params (model)
  Latency·Throughput (system)
  Memory (HBM, VRAM)

MLPerf:
  Standardized benchmark
  Training and inference

───── Hardware Co-design ─────

GPU features:
  FP16 Tensor Core: 16× FP32
  BF16 Tensor Core: 같은 BF16 지원
  INT8 DP4A: 4× FP16
  Sparse Tensor Core (Ampere+): 2:4 sparsity 2×
  
Mixed precision recipe:
  Weights FP16/BF16
  Activations FP16/BF16
  Master weights FP32
  Gradient FP16 + loss scale
  Optimizer FP32

───── 레포 간 연결 ─────

PyTorch Internals (앞):
  CUDA kernel, Triton
  Mixed precision

LLM Efficiency (Layer 4-B):
  LoRA, QLoRA, MoE
  Speculative decoding

Calculus (Layer 0):
  Hessian, Taylor (OBD)

Information Theory (Layer 0):
  KL (KD loss)
  Entropy (quantization level)

NN Theory (Layer 2):
  Weight 분포
  Pruning sensitivity

Distributed Training (직전):
  Large model training
  Compression으로 deploy

MLOps (다음):
  Monitoring compressed models
  A/B test efficiency vs quality
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·기법 (3~4줄)
- 전체 문서 개수 확인 (35개 목표)
- Python + PyTorch + Triton + ONNX 실험 환경
- PyTorch Internals, LLM Efficiency, Calc, Info 레포 참조 관계
- MLOps로 이어지는 deployment의 실전

**준비됐으면 1단계 구조 설계부터 시작해줘!**
