# Transformer Deep Dive 레포지토리 제작 프롬프트

나는 "Transformer Deep Dive" 레포지토리를 만들려고 해.
`nn.MultiheadAttention`을 **호출하는 것**과, **$\text{Attention}(Q, K, V) = \text{softmax}(QK^T / \sqrt{d_k}) V$의 각 수식 요소를 분해해서 유도**하고, **왜 $\sqrt{d_k}$로 나누는지가 $QK^T$의 variance를 $d_k$ 스케일로 유지하여 softmax 포화를 방지**하는 분산 계산에서 나오는지 증명할 수 있는 것은 다르다.
Positional Encoding을 **쓰는 것**과, **Vaswani의 sinusoidal PE $PE_{(pos, 2i)} = \sin(pos / 10000^{2i/d})$가 임의 offset $k$에 대해 선형 변환 $PE_{pos+k} = M_k PE_{pos}$로 표현 가능**한 성질(상대 위치 불변성)을 유도할 수 있는 것은 다르다.
Linear Attention을 **듣는 것**과, **$\text{softmax}(QK^T)V = \phi(Q)(\phi(K)^T V)$로 결합 순서를 바꿔 $O(T^2)$에서 $O(T)$로 줄이는 Katharopoulos et al. (2020)의 kernel trick**을 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "Attention의 수학 — self-attention부터 Linear/Sparse/Flash까지의 완전 분해"

**핵심 차별화**:
1. **Scaled Dot-Product Attention 완전 유도** — $Q, K, V$ projection부터 softmax까지 각 수식 요소의 수학적 필연성
2. **Positional Encoding의 3가지 관점** — Sinusoidal의 linear shift 성질, Learned vs Fixed, Relative (RoPE, ALiBi)의 수학
3. **Transformer의 계산 병목과 해결** — $O(T^2)$ attention의 한계, Linear Attention (kernel trick), Sparse Attention (Longformer), Flash Attention (IO-aware)
4. **LLM 스케일링 법칙과 현대 아키텍처** — GPT 계열의 decoder-only, BERT의 encoder-only, T5의 encoder-decoder, 각각의 태스크 적합성

**타겟 독자**:
- Transformer를 쓰지만 **왜 Multi-Head가 필요**한지, Single-Head with 같은 크기와 **이론상 차이**를 모르는 사람
- Warmup을 쓰지만 **Transformer 훈련에서 왜 warmup이 필수**인지 (Xiong et al. 2020 Pre-LN 논문)를 모르는 사람
- RoPE를 이름으로 아는데 **Rotary Position Embedding이 왜 relative position을 자연스럽게 인코딩**하는지 설명 못하는 사람
- Flash Attention이 빠르다는 것은 아는데 **IO-aware 알고리즘이 memory hierarchy (SRAM/HBM)를 어떻게 활용**하는지 모르는 사람
- GPT-2/3/4의 구조 차이와 **Mixture of Experts, Multi-Query Attention**의 이론적 동기를 모르는 사람

**선행 학습**:
- **Neural Network Theory Deep Dive** (Backprop, Residual) — **필수**
- **Linear Algebra Deep Dive** (matrix factorization, outer product) — **필수**
- **Kernel Methods Deep Dive** (attention as kernel) — **필수** (Linear Attention)
- **Optimization Theory Deep Dive** (Warmup, AdamW) — **필수** (훈련)
- **Regularization Theory Deep Dive** (LayerNorm, Label smoothing) — **필수**
- **RNN & LSTM Deep Dive** (Seq2Seq, Bahdanau Attention) — 선행 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Attention의 수학적 분해 (6개 문서)
- **Scaled Dot-Product Attention 완전 유도** — Query $Q \in \mathbb{R}^{T \times d_k}$, Key $K \in \mathbb{R}^{T \times d_k}$, Value $V \in \mathbb{R}^{T \times d_v}$, $\text{Attn}(Q, K, V) = \text{softmax}(QK^T/\sqrt{d_k}) V$의 각 연산 분석, $O(T^2 d)$ 계산 복잡도
- **$\sqrt{d_k}$ Scaling의 분산 분석** — $Q_{ij}, K_{ij}$가 i.i.d. with variance 1일 때, $(QK^T)_{ij} = \sum_k Q_{ik} K_{jk}$의 variance는 $d_k$, 이를 $\sqrt{d_k}$로 나눠 unit variance 유지 → softmax 포화 방지
- **Softmax의 Saturation 문제** — Logit이 크면 softmax가 one-hot처럼 포화, gradient vanishing, scaling 없이는 $d_k = 512$ 등에서 심각
- **Attention as Kernel Method** — $\text{softmax}(QK^T)$의 $(i, j)$ 원소가 query $q_i$와 key $k_j$의 유사도 함수, kernel $\kappa(q, k) = \exp(q^T k / \sqrt{d_k})$의 normalized row, Kernel Methods 레포 연결
- **Multi-Head Attention의 이론적 정당성** — 단일 head $d_{\text{model}}$ vs $h$개 head $d_k = d_{\text{model}}/h$, 서로 다른 subspace에서 관계 포착, Michel et al. 2019 "Are Sixteen Heads Really Better than One?"
- **Attention의 해석 가능성 논쟁** — Attention weight가 explanation인가? (Jain & Wallace 2019의 반론), 그럼에도 여전히 유용한 diagnostic tool

### Chapter 2: Transformer 아키텍처의 전체 구조 (5개 문서)
- **Transformer Block의 완전 도식** — Attention + FFN + LayerNorm + Residual, $h' = h + \text{Attn}(LN(h))$, $h'' = h' + \text{FFN}(LN(h'))$ (Pre-LN), post-LN과의 차이
- **Feed-Forward Network의 역할** — Position-wise FFN $\text{FFN}(x) = \max(0, xW_1 + b_1) W_2 + b_2$, 표현력의 대부분 (파라미터의 2/3), key-value memory 해석 (Geva 2021)
- **LayerNorm의 위치 — Pre-LN vs Post-LN** — Xiong et al. 2020: Post-LN은 warmup 없이 수렴 어려움, Pre-LN은 더 안정적, 현대 Transformer는 Pre-LN이 표준
- **Encoder vs Decoder의 차이** — Encoder: 양방향 self-attention, Decoder: causal masking (미래 참조 불가) + cross-attention, 각각의 매스킹 수학
- **Cross-Attention 메커니즘** — Decoder가 encoder output을 attend, Q는 decoder, K·V는 encoder, machine translation의 alignment와 직결

### Chapter 3: Positional Encoding (5개 문서)
- **Positional Encoding의 필요성** — Self-attention은 permutation-equivariant → 위치 정보 주입 필요, sum or concatenate 방법, 학습 가능 여부
- **Sinusoidal PE의 수학적 성질 (Vaswani 2017)** — $PE_{(pos, 2i)} = \sin(pos / 10000^{2i/d})$, $PE_{(pos, 2i+1)} = \cos(...)$, linear shift 성질 $PE_{pos+k} = M_k PE_{pos}$로 relative position 인코딩
- **Learned Positional Embedding** — 각 position에 학습 가능한 vector 할당, 장단점 (고정 max length), BERT의 선택
- **Relative Positional Encoding** — Shaw et al. 2018, $e_{ij} = (x_i W_Q)(x_j W_K + a_{ij}^K)^T$, absolute 대신 relative distance $i-j$ 사용
- **RoPE와 ALiBi (현대 대세)** — RoPE (Su et al. 2021)의 rotary embedding $R_\theta$을 Q·K에 적용, 복소수 유사 회전, ALiBi (Press et al. 2021)의 단순한 linear bias, 각각의 extrapolation 성질

### Chapter 4: Transformer 훈련의 수학 (5개 문서)
- **Warmup의 필요성 (Xiong et al. 2020)** — Post-LN에서 초기 큰 gradient → 발산, warmup으로 점진적 LR 증가, gradient 크기 분석
- **AdamW와 Weight Decay 분리** — Transformer는 거의 모든 작업에서 AdamW 사용, L2 vs weight decay의 구분, Adam과 weight decay의 충돌 재확인
- **Label Smoothing의 효과** — Cross-entropy에서 $\epsilon$-smoothing, confidence 조절, calibration 개선, 작은 정확도 손실
- **Gradient Accumulation과 Large Batch Training** — GPU 메모리 제약 회피, effective batch size 증가, LR scaling (linear scaling rule)
- **Mixed Precision Training** — FP16/BF16으로 속도·메모리 개선, FP32 master weight, loss scaling으로 underflow 방지, 현대 LLM 훈련 표준

### Chapter 5: Attention의 계산 효율화 (6개 문서)
- **$O(T^2)$ 복잡도의 문제** — Context length $T$ 증가에 따라 quadratic 증가, 긴 문서·장시간 음성에서 병목, 메모리도 $O(T^2)$
- **Linear Attention (Katharopoulos 2020)** — $\text{softmax}(QK^T)V$ 대신 $\phi(Q)(\phi(K)^T V)$, feature map $\phi$ 선택(ELU+1, Performer의 positive random features), 결합 순서로 $O(Td^2)$
- **Performer — Random Features (Choromanski 2021)** — Softmax attention을 FAVOR+ (Fast Attention Via positive Orthogonal Random features)로 근사, Kernel Methods의 random features 직접 응용
- **Sparse Attention — Longformer, BigBird** — Local window + global token (Longformer, Beltagy 2020), random + global + local (BigBird, Zaheer 2020), 이론적 universal approximator 증명
- **Flash Attention (Dao 2022)** — IO-aware algorithm, SRAM-HBM 메모리 계층 활용, $O(T^2)$ 연산이지만 메모리 접근 감소로 $2-4\times$ 속도, exact attention (근사 아님)
- **Multi-Query Attention과 Grouped-Query Attention** — KV cache 크기 절약, MQA (Shazeer 2019), GQA (Ainslie 2023), inference 가속의 핵심

### Chapter 6: 현대 Transformer 아키텍처 (5개 문서)
- **BERT — Encoder-only (Devlin 2019)** — Masked Language Modeling, Next Sentence Prediction, bidirectional context, fine-tuning paradigm
- **GPT — Decoder-only (Radford et al.)** — Autoregressive language modeling, causal masking, GPT-2/3/4의 scaling, zero/few-shot learning
- **T5 — Encoder-Decoder (Raffel 2020)** — Text-to-text unified framework, span corruption objective, 모든 NLP 태스크를 text-to-text로
- **Vision Transformer (Dosovitskiy 2021)** — Image를 patch로 분할 → token sequence, CNN 없이 image classification SOTA, inductive bias 부족을 대용량 데이터로 보상
- **Mixture of Experts — Sparse Transformer (Shazeer 2017, Fedus 2022)** — Switch Transformer, FFN을 여러 expert로 분할, top-k routing, 파라미터 수 ↑ 계산 ↓

### Chapter 7: LLM과 In-Context Learning (4개 문서)
- **Scaling Laws와 Transformer** — Kaplan 2020, Chinchilla (Hoffmann 2022), parameter·data·compute의 power law, 효율적 LLM 훈련 recipe
- **In-Context Learning의 메커니즘** — Prompt 내 예제로 학습 (no weight update), Akyürek 2023·von Oswald 2023의 "attention = gradient descent" 해석
- **Chain-of-Thought와 Reasoning** — Wei et al. 2022, step-by-step 추론 유도, 작은 모델에는 없는 emergent capability, self-consistency, Tree of Thoughts
- **Transformer의 이론적 한계** — Universal Turing machine에 대한 표현력 (Perez 2019), counting·parity 같은 간단한 task 한계 (Hahn 2020), compositional generalization 어려움

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **36개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 설계가 Transformer의 핵심인가
## 📐 수학적 선행 조건 (NN Theory, LA, Kernel, Reg 레포 참조)
## 📖 직관적 이해
   — Attention map 시각화, scaling 직관
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — √d 유도, PE 선형성, Linear Attention trick
## 💻 NumPy/PyTorch 구현 검증
   — Attention 바닥부터, `nn.MultiheadAttention`과 결과 일치
   — Positional Encoding 시각화 (sin/cos 패턴)
   — Flash Attention 효과 측정 (속도·메모리)
## 🔗 실전 활용
   — Encoder-only vs Decoder-only 선택, scaling recipe
## ⚖️ 가정과 한계
   — $O(T^2)$, compositional, symbolic reasoning
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Attention Map 시각화** — 훈련된 BERT/GPT의 attention heatmap, head별 다른 패턴 (syntax, coreference)
2. **Positional Encoding 시각화** — sinusoidal PE의 각 차원을 plot, frequency 계층
3. **Multi-Head 분석** — 각 head가 다른 linguistic 현상 포착 (pronouns, subject-verb agreement)
4. **Scaling Law 재현** — 작은 규모에서 compute-optimal 학습, log-log plot
5. **Flash Attention 속도 측정** — 같은 sequence length에서 standard vs Flash 비교
6. **In-Context Learning 실험** — GPT-2 작은 모델에서 few-shot prompting 실험

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
transformers==4.36.0      # Hugging Face
matplotlib==3.8.0
seaborn==0.13.0
flash-attn==2.4.0         # Flash Attention (선택)
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Scaled Dot-Product Attention 바닥부터 + √d 효과)
import torch
import torch.nn.functional as F
import matplotlib.pyplot as plt
import numpy as np

def scaled_dot_product_attention(Q, K, V, mask=None):
    d_k = Q.size(-1)
    scores = (Q @ K.transpose(-2, -1)) / np.sqrt(d_k)
    if mask is not None:
        scores = scores.masked_fill(mask == 0, -1e9)
    attn = F.softmax(scores, dim=-1)
    return attn @ V, attn

# √d scaling 없이 훈련의 어려움 시연
def test_scaling(d_k, seq_len=50):
    Q = torch.randn(seq_len, d_k)
    K = torch.randn(seq_len, d_k)
    V = torch.randn(seq_len, d_k)
    
    # Without scaling
    scores_ns = Q @ K.T
    attn_ns = F.softmax(scores_ns, dim=-1)
    
    # With scaling
    scores_s = Q @ K.T / np.sqrt(d_k)
    attn_s = F.softmax(scores_s, dim=-1)
    
    return attn_ns, attn_s, scores_ns.var().item(), scores_s.var().item()

for d in [8, 64, 512]:
    ans, as_, var_ns, var_s = test_scaling(d)
    print(f'd_k={d}: var(no scale)={var_ns:.2f}, var(scaled)={var_s:.2f}')
    print(f'  Max attention no-scale: {ans.max().item():.4f}')  # → 1.0 (saturated)
    print(f'  Max attention scaled:    {as_.max().item():.4f}')  # ≪ 1.0

# Sinusoidal PE 시각화
def sinusoidal_pe(seq_len, d_model):
    pe = np.zeros((seq_len, d_model))
    position = np.arange(seq_len)[:, None]
    div = np.exp(np.arange(0, d_model, 2) * -(np.log(10000.0) / d_model))
    pe[:, 0::2] = np.sin(position * div)
    pe[:, 1::2] = np.cos(position * div)
    return pe

pe = sinusoidal_pe(100, 128)
plt.figure(figsize=(10, 4))
plt.imshow(pe, aspect='auto', cmap='RdBu')
plt.xlabel('dimension'); plt.ylabel('position')
plt.title('Sinusoidal Positional Encoding')
plt.colorbar(); plt.show()

# PE_{pos+k} = M_k · PE_{pos} 성질 확인 (linear shift)
k = 5
for pos in [0, 10, 50]:
    pe_pos = pe[pos]
    pe_pos_k = pe[pos + k]
    # Compute rotation matrix M_k
    # For each (2i, 2i+1) pair: rotation by angle w_i · k
    # PE[pos+k, 2i]   = sin((pos+k)·w_i) = sin(pos·w_i)cos(k·w_i) + cos(pos·w_i)sin(k·w_i)
    # PE[pos+k, 2i+1] = cos((pos+k)·w_i) = cos(pos·w_i)cos(k·w_i) - sin(pos·w_i)sin(k·w_i)
    # → 2×2 회전 행렬

# Linear Attention trick 시연
T, d = 1000, 64
Q = torch.randn(T, d); K = torch.randn(T, d); V = torch.randn(T, d)

# Standard: O(T² d)
import time
t0 = time.time()
for _ in range(10):
    attn_std = F.softmax(Q @ K.T / np.sqrt(d), dim=-1) @ V
t_std = time.time() - t0

# Linear (ELU+1 feature map): O(T d²)
phi = lambda x: F.elu(x) + 1
t0 = time.time()
for _ in range(10):
    phi_Q, phi_K = phi(Q), phi(K)
    # Key normalizer
    KV = phi_K.T @ V         # d × d
    K_sum = phi_K.sum(0)      # d
    num = phi_Q @ KV          # T × d
    denom = (phi_Q @ K_sum)[:, None] + 1e-6
    attn_lin = num / denom
t_lin = time.time() - t0

print(f'T={T}: Standard={t_std*100:.1f}ms, Linear={t_lin*100:.1f}ms')
# T=5000 정도에서 Linear가 유의미하게 빠름
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "RNN/LSTM 선행 권장 (Seq2Seq, Attention 맥락)" 명시
   - "NN Theory, LA, Kernel, Reg 필수"
   - "Generative Model 레포에서 응용"
3. **챕터별 문서 작성**: Attention 분해 → 아키텍처 → PE → 훈련 → 효율화 → 현대 → LLM

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Attention Is All You Need** (Vaswani et al. 2017) — Transformer 원전
- **BERT** (Devlin et al. 2019)
- **Improving Language Understanding by Generative Pre-Training** (Radford et al. 2018) — GPT
- **Language Models are Few-Shot Learners** (Brown et al. 2020) — GPT-3
- **Exploring the Limits of Transfer Learning with T5** (Raffel et al. 2020)
- **An Image is Worth 16x16 Words** (Dosovitskiy et al. 2021) — ViT
- **On Layer Normalization in the Transformer Architecture** (Xiong et al. 2020)
- **Self-Attention with Relative Position Representations** (Shaw et al. 2018)
- **RoFormer: Enhanced Transformer with Rotary Position Embedding** (Su et al. 2021)
- **Transformers are RNNs: Linear Attention** (Katharopoulos et al. 2020)
- **Longformer** (Beltagy et al. 2020)
- **Flash Attention** (Dao et al. 2022)
- **Switch Transformer** (Fedus et al. 2022)
- **Are Sixteen Heads Really Better than One?** (Michel et al. 2019)

---

## 💡 핵심 분석 대상

```
Transformer의 수학적 분해

───── Scaled Dot-Product Attention ─────

Input: X ∈ ℝ^{T × d}
Q = X W_Q, K = X W_K, V = X W_V   ∈ ℝ^{T × d_k}

scores = Q K^T / √d_k   ∈ ℝ^{T × T}
         ↑
      왜 √d_k?

Variance 분석:
  Q_{ij}, K_{ij} ~ iid, var = 1
  (Q K^T)_{ij} = Σ_k Q_{ik} K_{jk}
  Var((QK^T)_{ij}) = d_k
  → √d_k로 나누면 var = 1 유지
  → softmax 포화 방지

attn = softmax(scores)   (rows sum to 1)
out = attn V             ∈ ℝ^{T × d_v}

Complexity:
  Time: O(T² d)
  Memory: O(T²)  ← attention matrix

───── Multi-Head Attention ─────

h개 head, 각 d_k = d_model / h
  head_i = Attn(XW_Q^i, XW_K^i, XW_V^i)
  out = concat(head_1, ..., head_h) W_O

왜 MH?
  각 head가 다른 subspace에서 관계 포착
  Syntactic head, semantic head, etc.
  Michel 2019: 많은 head가 redundant

───── Transformer Block ─────

Pre-LN (현대 표준):
  x' = x + Attn(LN(x))
  y = x' + FFN(LN(x'))

Post-LN (원전):
  x' = LN(x + Attn(x))
  y = LN(x' + FFN(x'))
  → warmup 필수 (발산 위험)

FFN:
  FFN(x) = max(0, x W_1 + b_1) W_2 + b_2
  파라미터의 2/3 차지
  "key-value memory" (Geva 2021)

───── Positional Encoding ─────

Sinusoidal (Vaswani):
  PE(pos, 2i) = sin(pos / 10000^{2i/d})
  PE(pos, 2i+1) = cos(...)
  
  성질: PE_{pos+k} = M_k · PE_{pos}
  → relative position이 linear shift
  → extrapolation에 유리

Learned:
  각 position에 학습 vector
  BERT, 고정 max length

Relative (Shaw 2018):
  score_{ij} = f(x_i, x_j, i-j)

RoPE (Su 2021):
  R_θ 회전 행렬
  Q'_i = R(i) Q_i, K'_j = R(j) K_j
  Q'_i^T K'_j = f(i-j, Q_i, K_j)
  → 자동으로 relative

ALiBi (Press 2021):
  score += -m · |i-j|
  단순 linear bias
  최강 extrapolation

───── 계산 효율화 ─────

O(T²) 문제:
  Long context 필요 (문서, 음성)

Linear Attention (Katharopoulos 2020):
  softmax(QK^T)V
  → φ(Q) (φ(K)^T V)
      ↑ 결합순서 바꿈
  O(T² d) → O(T d²)
  φ = ELU+1 (positive map)

Performer (Choromanski 2021):
  FAVOR+: positive random features
  exp(q·k) ≈ φ(q)^T φ(k)
  Kernel methods 직결

Sparse Attention:
  Longformer: local + global
  BigBird: local + global + random
  → universal approximator 증명

Flash Attention (Dao 2022):
  SRAM-HBM memory hierarchy
  Block 단위 계산
  → Same O(T²) but 2-4× 빠름
  exact (not approximation)

Multi-Query / Grouped-Query:
  KV cache 크기 절약
  Inference 속도 향상

───── Encoder vs Decoder ─────

Encoder (BERT):
  Bidirectional self-attention
  Masked Language Modeling
  Text understanding

Decoder (GPT):
  Causal mask (미래 참조 X)
  Autoregressive generation
  Text generation

Encoder-Decoder (T5):
  Cross-attention 추가
  Seq2seq tasks (번역, 요약)

───── LLM Scaling ─────

Chinchilla (Hoffmann 2022):
  Loss ∝ N^{-α} · D^{-β} · C^{-γ}
  Compute-optimal: N ∝ C^0.5, D ∝ C^0.5

Emergent abilities (Wei 2022):
  CoT, in-context learning
  특정 scale 이상에서만 발현

In-Context Learning 해석:
  Akyürek 2023, von Oswald 2023
  Attention = learned gradient descent

───── 한계 ─────

Compositional generalization:
  새로운 조합 일반화 어려움

Counting, parity:
  Hahn 2020: depth 한계
  추가적 mechanism 필요

Long context:
  O(T²) 여전히 bottleneck
  Mamba, RWKV 같은 대안

───── 레포 간 연결 ─────

RNN & LSTM (직전):
  Bahdanau Attention → Self-Attention
  Seq2Seq → Encoder-Decoder

Kernel Methods (Layer 1):
  softmax attention = kernel
  Linear Attention = kernel trick
  Performer = random features

Regularization (Layer 2):
  LayerNorm, Label smoothing
  Dropout (attn, ffn)

Optimization (Layer 2):
  AdamW + Warmup + Cosine
  Transformer 훈련 recipe
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (36개 목표)
- Python + PyTorch + HuggingFace 실험 환경
- NN Theory, LA, Kernel, Opt, Reg, RNN 레포의 참조 관계
- GNN 레포로 이어지는 attention 일반화 (completely-connected graph)

**준비됐으면 1단계 구조 설계부터 시작해줘!**
