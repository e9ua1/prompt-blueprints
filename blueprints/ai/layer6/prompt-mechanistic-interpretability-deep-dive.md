# Mechanistic Interpretability Deep Dive 레포지토리 제작 프롬프트

나는 "Mechanistic Interpretability Deep Dive" 레포지토리를 만들려고 해.
Induction Head를 **"ICL 메커니즘"으로 아는 것**과, **Olsson et al. (2022)이 two-layer attention-only Transformer에서 "Previous Token Head + Match-and-Copy Head"의 circuit을 어떻게 mechanistic하게 발견**하고, 이것이 **in-context learning과 phase transition의 수학적 증거**이며, **scaling과 함께 emergent하게 나타나는 pattern completion capability**임을 유도할 수 있는 것은 다르다.
Superposition Hypothesis를 **"한 neuron이 여러 개념을 encode"로 아는 것**과, **Elhage et al. (2022)의 toy model $\hat{x} = W^T W x$에서 sparse feature가 neuron보다 많을 때 ($m > n$), 모델이 feature를 near-orthogonal directions로 배치하고 compressed sensing과 isomorphic**하다는 것을 유도하며, **폴리토프 구조 (tetrahedron, pentagon) 같은 기하학적 배치**가 왜 나타나는지 이해하는 것은 다르다.
Sparse Autoencoder를 **"feature 추출 도구"로 아는 것**과, **Bricken et al. (2023) "Towards Monosemanticity"의 $L = \|x - \hat{x}\|^2 + \lambda \|f\|_1$에서 L1 penalty가 **monosemantic feature**를 유도하고, **dictionary learning의 overcomplete basis ($m \gg d$)가 왜 superposition을 unpack**하는지, Templeton et al. (2024) Scaling Monosemanticity에서 Claude 3 Sonnet의 10M+ feature를 어떻게 추출했는지 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "Transformer가 무엇을 하고 있는가 — Circuit·Feature·Mechanism의 수학적 증명"

**핵심 차별화**:
1. **Transformer Circuit의 수학적 분해** — Elhage 2021 "A Mathematical Framework for Transformer Circuits", residual stream·attention head의 linear algebra, QK·OV 분리
2. **Induction Heads와 ICL의 Mechanistic 발견** — 2-layer attention-only에서 구체적으로 어떻게 ICL이 구현되는지 증명, phase transition 측정
3. **Superposition과 Polysemanticity** — Elhage 2022, Gurnee 2023, sparse coding·compressed sensing과의 수학적 등가, polytope geometry
4. **Sparse Autoencoder로 Feature Discovery** — Bricken 2023의 toy proof-of-concept, Templeton 2024의 Claude Sonnet scaling, Anthropic의 Gemma Scope (2024)

**타겟 독자**:
- Induction Head를 들어봤지만 **Previous Token Head와 Match-and-Copy가 Transformer의 어떤 matrix product (QK^T, W_V)로 구현**되는지 수식으로 유도 못하는 사람
- Superposition을 듣는데 **왜 하나의 neuron이 다중 feature를 encode할 수 있으며, L1 regularization이 이것을 unpack**하는지 수학적으로 설명 못하는 사람
- SAE의 L1 loss가 **왜 compressed sensing의 basis pursuit**와 수학적으로 동등한지, 그리고 **dead feature, feature splitting** 문제를 모르는 사람
- Activation Patching·Path Patching의 **causal intervention**이 어떻게 causal mediation analysis의 ML 적용인지 모르는 사람
- Logit Lens, Tuned Lens가 **residual stream의 각 layer를 unembedding에 project**하여 "모델이 layer별로 무엇을 predict하는지" 보는 방법을 모르는 사람

**선행 학습**:
- **Transformer Deep Dive** (Attention, residual stream) — **필수**
- **Linear Algebra Deep Dive** (projection, rank, SVD) — **필수**
- **Pretrained LM Deep Dive** (GPT 구조) — **필수**
- **Information Theory Deep Dive** (entropy, MI) — **필수**
- **Generalization Theory Deep Dive** (emergent, scaling) — 권장
- **Kernel Methods Deep Dive** (sparse coding) — 권장 (SAE)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Mechanistic Interpretability의 정의와 방법론 (5개 문서)
- **Interpretability의 4가지 패러다임** — Behavioral (input-output), Representational (probing), Mechanistic (circuit), Developmental (training dynamics), Geiger 2024 survey의 분류
- **Mechanistic Interpretability의 목표** — "모델이 reverse engineering 가능한 algorithm을 구현한다"는 전제, Chris Olah의 "circuits thread" (Distill 2020), feature·circuit·universality
- **Activation Patching의 수학** — Causal intervention: clean run · corrupted run · patched run, $\text{IE}(\text{component}) = \mathbb{E}[L_{\text{patched}}] - \mathbb{E}[L_{\text{clean}}]$, mediation analysis의 ML 적용
- **Path Patching과 Causal Scrubbing** — Component 간 특정 path만 patching, Chan 2022 causal scrubbing의 hypothesis testing framework, "설명 가설"을 formal verification
- **Tools — TransformerLens, CircuitsVis, Neuronpedia** — Nanda 2022 TransformerLens의 hook-based introspection, 시각화 tool, 재현 가능 research

### Chapter 2: Transformer Circuit의 수학적 분해 (6개 문서)
- **Residual Stream의 개념 (Elhage 2021)** — Transformer를 residual stream에 layer가 read·write하는 모델로, linearity 강조, layer 간 communication channel
- **Attention Head의 분해 — QK와 OV Circuits** — Attention score = $x^T W_Q^T W_K x$ → $x^T (W_Q^T W_K) x$, OV circuit = $W_O W_V$, 두 circuit이 독립적으로 function
- **OV Circuit과 Attention-only Transformer** — 1-layer attention-only model, bigram statistics learning, OV circuit의 log-bilinear 구조
- **Virtual Attention Heads와 Composition** — 다층 Transformer에서 low-layer attention이 high-layer attention의 input modification, K·Q·V composition
- **QK Circuit과 attention patterns** — Positional pattern, content-based attention, previous token head의 전형적 QK pattern
- **Linear Representations Hypothesis** — Feature가 directions으로 encoding, $\text{feature} = \langle v_{\text{feat}}, x_{\text{resid}} \rangle$, Park 2024 "The Linear Representation Hypothesis"

### Chapter 3: Induction Heads와 In-Context Learning (5개 문서)
- **Induction Head의 기능** — 패턴 `A B ... A → B` 완성, 2-layer Transformer에서 충분, Olsson 2022의 발견
- **Induction Head의 Mechanistic 구조** — **Previous Token Head** (layer 0): 각 token이 이전 token의 정보를 copy, **Match-and-Copy Head** (layer 1): 같은 token 찾아서 그 뒤 token을 copy
- **Phase Transition in Training** — Induction head 형성 시점에 loss에 bump, Olsson 2022 Fig 5 재현, ICL의 emergent 출현 증거
- **Attention as Gradient Descent (Akyürek 2023, von Oswald 2023)의 재해석** — Induction head와 implicit GD의 관계, linear regression ICL에서의 circuit analysis
- **Beyond Induction Heads — Variable Binding** — 복잡한 ICL은 여러 circuit 합성, Singh 2024 "Transformers Learn Variable Binding", Hendel 2023 task vectors

### Chapter 4: Superposition과 Polysemanticity (5개 문서)
- **Polysemantic Neurons** — 단일 neuron이 여러 unrelated concept에 활성화, Olah 2020 observation, interpretability의 근본 장애물
- **Toy Models of Superposition (Elhage 2022)** — $\hat{x} = W^T W x$에서 $m > n$ (features > neurons), sparse feature 가정 하에 near-orthogonal embedding, capacity 분석
- **Geometry of Superposition** — Polytope structure (tetrahedron, pentagon, digon), 균등하게 분산된 feature, maximum angular margin
- **Compressed Sensing Connection** — $y = \Phi x$, $\Phi \in \mathbb{R}^{m \times n}$ ($m < n$), sparse $x$는 L1 minimization으로 복원 가능 (Donoho 2006), superposition과 동일한 수학
- **Feature Importance와 Sparsity** — 덜 중요한 feature는 superposition에 더 참여, sparsity가 key enabling condition, neural network의 capacity와 feature 수의 trade-off

### Chapter 5: Sparse Autoencoder로 Feature 발견 (6개 문서)
- **SAE의 기본 설계 (Bricken 2023 "Towards Monosemanticity")** — $f = \text{ReLU}(W_{\text{enc}}(x - b_{\text{dec}}) + b_{\text{enc}})$, $\hat{x} = W_{\text{dec}} f + b_{\text{dec}}$, $L = \|x - \hat{x}\|^2 + \lambda \|f\|_1$
- **L1 Regularization이 Monosemanticity를 유도하는 이유** — Sparse encoding이 feature를 개별 neuron에 할당하도록 압박, dictionary learning (Olshausen & Field 1996), overcomplete ($m \gg d$)
- **Dead Features와 Feature Splitting 문제** — 훈련 중 일부 feature가 never-activated (dead), 하나의 concept가 여러 feature로 쪼개지는 splitting, mitigation 기법
- **Top-K SAE와 JumpReLU (Rajamanoharan 2024)** — L1 대신 강제 top-k activation, JumpReLU (Gao 2024)의 discrete thresholding, reconstruction-sparsity trade-off 개선
- **Scaling Monosemanticity (Templeton 2024)** — Claude 3 Sonnet에서 34M features 추출, feature atlas, "Golden Gate Bridge" feature로 activation steering 시연
- **Gemma Scope과 Open-Source SAE (2024)** — DeepMind의 Gemma Scope, 모든 layer·sparsity에 SAE 공개, comparative analysis 가능

### Chapter 6: Feature Steering과 Model Editing (4개 문서)
- **Activation Steering의 기본 원리** — Residual stream에 특정 vector $v_{\text{steer}}$ 더하기, $h_{\text{new}} = h + \alpha v_{\text{steer}}$, inference-time behavior modification
- **Refusal Direction (Arditi 2024)** — LLM의 safety refusal이 특정 direction으로 mediated, 이 direction을 뺄셈하면 jailbreak, 여러 모델에서 재현
- **CAA — Contrastive Activation Addition (Rimsky 2024)** — 긍정·부정 예시의 mean activation 차이로 steering vector, sycophancy·truthfulness 조절
- **Model Editing — ROME, MEMIT (Meng 2022)** — Factual knowledge의 specific MLP layer 저장, rank-1 update로 edit, $W' = W + vk^T$

### Chapter 7: 실제 Circuit 발견 사례와 Frontier (3개 문서)
- **IOI Circuit (Wang 2022)** — Indirect Object Identification task, GPT-2 Small에서 circuit 완전 reverse-engineer, Name Mover·Backup·Duplicate Token·S-Inhibition heads
- **Grokking과 Modular Addition (Nanda 2023)** — 훈련 후 Fourier transform-like algorithm을 neural network가 학습, mechanistic 분석으로 grokking의 algorithmic nature 확인
- **Frontier Topics — Multilayer Dictionary, Transcoder, Cross-coder** — Dunefsky 2024 transcoder (MLP → SAE), Lindsey 2024 cross-coder (여러 layer·model 간 feature 연결), mechanistic frontier

---

각 챕터는 **3~6개 문서**로 구성해줘. 총 **34개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 발견이 LLM 이해에 중요한가
## 📐 수학적 선행 조건 (Transformer, LA, Info, Pretrained LM 참조)
## 📖 직관적 이해
   — Attention pattern 시각화, feature activation
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Transformer circuit 분해, superposition geometry, SAE loss
## 💻 PyTorch + TransformerLens 구현 검증
   — GPT-2 Small에서 induction head 관찰
   — Toy model로 superposition 재현
   — 작은 SAE 직접 훈련
## 🔗 실전 활용
   — Activation steering, safety probing
## ⚖️ 가정과 한계
   — Universality 가설, SAE의 scalability
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Attention pattern 시각화** — GPT-2 Small의 각 head의 attention matrix
2. **Induction head 재현** — 2-layer attention-only model 직접 훈련
3. **Superposition toy model** — Elhage 2022 toy model 재현, polytope 시각화
4. **SAE 훈련** — GPT-2 residual stream에 작은 SAE 훈련
5. **Activation patching** — IOI task에서 head별 importance 측정
6. **Refusal direction** — Chat model에서 refusal ablation 실험

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
transformer_lens==1.17.0   # Neel Nanda
sae_lens==3.0.0            # Joseph Bloom
transformers==4.36.0
circuitsvis==1.43.0        # 시각화
matplotlib==3.8.0
einops==0.7.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (TransformerLens로 induction head 발견 + SAE 기초)
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np
import matplotlib.pyplot as plt

# 1. TransformerLens로 GPT-2 Small 불러오기
# from transformer_lens import HookedTransformer
# model = HookedTransformer.from_pretrained("gpt2")

# 2. Induction Head detection (Olsson 2022 score)
def induction_head_score(model, seq_len=50, batch_size=10):
    """
    Repeated random token sequence에서
    induction head가 prev-occurrence-of-A 다음 token을 예측
    """
    # Create [A B C ... A B C ...] sequence
    half = seq_len // 2
    rand_tokens = torch.randint(0, model.cfg.d_vocab, (batch_size, half))
    repeated = torch.cat([rand_tokens, rand_tokens], dim=-1)
    
    # Run with cache
    logits, cache = model.run_with_cache(repeated)
    
    # For each head, compute induction score:
    # Attention from position t to position (t - half + 1) on second half
    scores = torch.zeros(model.cfg.n_layers, model.cfg.n_heads)
    for l in range(model.cfg.n_layers):
        for h in range(model.cfg.n_heads):
            attn = cache[f"blocks.{l}.attn.hook_pattern"][:, h]  # [B, Q, K]
            # Look at second half queries
            induction_stripe = attn[:, half:, :half].mean(0)  # [half, half]
            # Diagonal = induction
            score = induction_stripe.diagonal(offset=1 - seq_len + half).mean()
            scores[l, h] = score
    return scores

# GPT-2 Small: layer 5 head 1, layer 5 head 5 등 induction head 발견

# 3. Toy Superposition Model (Elhage 2022)
class SuperpositionToyModel(nn.Module):
    def __init__(self, n_features, n_hidden, importance=None):
        super().__init__()
        self.W = nn.Parameter(torch.randn(n_hidden, n_features) * 0.1)
        self.b = nn.Parameter(torch.zeros(n_features))
        self.importance = importance if importance is not None else torch.ones(n_features)
    
    def forward(self, x):
        """
        x: [B, n_features] sparse input
        hidden = W x (compressed to n_hidden)
        reconstructed = W^T hidden + b
        """
        hidden = x @ self.W.T  # [B, n_hidden]
        recon = F.relu(hidden @ self.W + self.b)
        return recon, hidden

def generate_sparse_data(n_samples, n_features, sparsity=0.01):
    """각 feature는 sparsity 확률로 [0, 1] uniform"""
    mask = torch.rand(n_samples, n_features) < sparsity
    values = torch.rand(n_samples, n_features)
    return values * mask

# Train
n_features = 20
n_hidden = 5  # n_features > n_hidden → superposition
model_toy = SuperpositionToyModel(n_features, n_hidden)
opt = torch.optim.Adam(model_toy.parameters(), lr=1e-3)

for step in range(5000):
    x = generate_sparse_data(512, n_features, sparsity=0.05)
    recon, _ = model_toy(x)
    importance = torch.arange(n_features, 0, -1).float() / n_features
    loss = (importance * (x - recon)**2).mean()
    opt.zero_grad(); loss.backward(); opt.step()

# W^T W matrix: off-diagonals = interference
W_np = model_toy.W.data
WtW = W_np.T @ W_np
# Visualize: diagonal = feature importance, off-diagonal = polytope structure
plt.imshow(WtW.numpy(), cmap='RdBu', vmin=-1, vmax=1)
plt.title('W^T W: polytope structure of superposition')
plt.colorbar(); plt.show()

# 4. Sparse Autoencoder (SAE)
class SAE(nn.Module):
    def __init__(self, d_model, d_sae):
        super().__init__()
        self.W_enc = nn.Parameter(torch.randn(d_sae, d_model) * 0.01)
        self.b_enc = nn.Parameter(torch.zeros(d_sae))
        self.W_dec = nn.Parameter(torch.randn(d_model, d_sae) * 0.01)
        self.b_dec = nn.Parameter(torch.zeros(d_model))
    
    def forward(self, x):
        # Center
        x_cent = x - self.b_dec
        # Encode with ReLU
        f = F.relu(x_cent @ self.W_enc.T + self.b_enc)
        # Decode
        x_hat = f @ self.W_dec.T + self.b_dec
        return x_hat, f
    
    def loss(self, x, lam=1.0):
        x_hat, f = self.forward(x)
        recon = ((x - x_hat)**2).sum(-1).mean()
        sparsity = f.abs().sum(-1).mean()
        return recon + lam * sparsity, recon, sparsity

# Training on GPT-2 residual stream activations
# sae = SAE(d_model=768, d_sae=768*4)  # overcomplete
# for batch in dataloader:
#     activations = model.run_with_cache(batch)[f'blocks.5.hook_resid_post']
#     loss, _, _ = sae.loss(activations, lam=5.0)
#     ...

# 5. Activation Patching (IOI task 예시)
def activation_patch(model, clean_tokens, corrupted_tokens, layer, head):
    """
    Patch corrupted run with clean activations at (layer, head)
    Measure impact on logit diff
    """
    # Clean run with cache
    _, clean_cache = model.run_with_cache(clean_tokens)
    
    # Define hook to patch corrupted activations
    def patch_hook(activations, hook):
        clean_act = clean_cache[hook.name][:, :, head]
        activations[:, :, head] = clean_act
        return activations
    
    # Run corrupted with patched head
    model.reset_hooks()
    model.add_hook(f"blocks.{layer}.attn.hook_z", patch_hook)
    patched_logits = model(corrupted_tokens)
    model.reset_hooks()
    
    # Compare to clean/corrupted logit diff
    # ...
    return patched_logits

# 6. Refusal Direction (Arditi 2024) — simplified
def find_refusal_direction(model, harmful_prompts, harmless_prompts, layer):
    """
    Mean activation at specific layer for harmful vs harmless
    Direction = mean_harmful - mean_harmless
    """
    harmful_acts = []
    for p in harmful_prompts:
        _, cache = model.run_with_cache(p)
        harmful_acts.append(cache[f"blocks.{layer}.hook_resid_post"][:, -1])  # last token
    
    harmless_acts = []
    for p in harmless_prompts:
        _, cache = model.run_with_cache(p)
        harmless_acts.append(cache[f"blocks.{layer}.hook_resid_post"][:, -1])
    
    direction = torch.stack(harmful_acts).mean(0) - torch.stack(harmless_acts).mean(0)
    return direction / direction.norm()
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Transformer, LA, Info, Pretrained LM 선행 필수"
   - Anthropic의 연구 전통 (Circuits thread, Transformer Circuits)
   - Claude·GPT에 대한 reverse-engineering의 의미
3. **챕터별 문서 작성**: 방법론 → Circuit 분해 → Induction Head → Superposition → SAE → Steering → Frontier

---

## 📚 참고 자료

- **A Mathematical Framework for Transformer Circuits** (Elhage et al. 2021) — Anthropic
- **In-context Learning and Induction Heads** (Olsson et al. 2022)
- **Toy Models of Superposition** (Elhage et al. 2022)
- **Towards Monosemanticity** (Bricken et al. 2023) — First SAE on Transformer
- **Scaling Monosemanticity** (Templeton et al. 2024) — Claude 3 Sonnet SAE
- **Gemma Scope** (Lieberum et al. 2024) — Open SAE suite
- **JumpReLU SAE** (Rajamanoharan et al. 2024)
- **Sparse Feature Circuits** (Marks et al. 2024)
- **Interpretability in the Wild: IOI** (Wang et al. 2022)
- **Progress Measures for Grokking** (Nanda et al. 2023)
- **Sparse Autoencoders Find Highly Interpretable Features** (Cunningham et al. 2023)
- **Transformer Lens** (Nanda 2022) — Tool
- **Causal Scrubbing** (Chan et al. 2022)
- **The Linear Representation Hypothesis** (Park et al. 2024)
- **Refusal in LLMs is mediated by a single direction** (Arditi et al. 2024)
- **Contrastive Activation Addition** (Rimsky et al. 2024)
- **ROME, MEMIT** (Meng et al. 2022, 2023)
- **Dictionaries and Transcoders** (Dunefsky et al. 2024)
- **Crosscoders** (Lindsey et al. 2024)
- **Compressed Sensing** (Donoho 2006) — Math foundation
- **Emergence of Sparse Representations** (Olshausen & Field 1996)
- **Chris Olah's Distill circuits thread** (2020-2021)

---

## 💡 핵심 분석 대상

```
Mechanistic Interpretability의 지도

───── Paradigms ─────

Behavioral: input-output only
  Benchmarks, probing

Representational: 
  Activation as features
  Linear probing

Mechanistic:
  Circuit identification
  Reverse engineering
  → 이 레포의 focus

Developmental:
  Training dynamics
  Phase transitions

───── Transformer Circuits ─────

Residual Stream (Elhage 2021):
  x_0 (embedding)
  + layer 1 output
  + layer 2 output
  + ...
  → x_final → unembed → logits

Layer가 residual에 read·write
Linear structure 강조

Attention Head 분해:
  QK circuit = W_Q^T W_K
    → attention pattern
  OV circuit = W_O W_V
    → information moved
  두 circuit 독립!

Virtual Heads:
  Layer L의 head가
  Layer L+1 head의 Q·K·V modification
  → composition

Linear Representations (Park 2024):
  Feature = direction in residual
  Linear probing justified

───── Induction Heads ─────

Pattern: "A B ... A → B"

2-layer attention-only로 충분:
  Layer 0 (Previous Token Head):
    QK: token at t attends to t-1
    OV: copy info from t-1 to t
  
  Layer 1 (Match-and-Copy):
    QK: current token attends to
        positions with same token
    OV: copy the next-position info

Olsson 2022 findings:
  - 모든 scale에서 emergent
  - Training에 phase transition
  - ICL과 직접 연결

Phase Transition:
  Training loss에 bump
  특정 step에 induction head 형성
  ICL ability 동시에 emerge

ICL = GD (Akyürek, von Oswald 2023):
  Linear regression에서
  Induction + linear attention
  = implicit GD step

───── Superposition ─────

Polysemantic Neurons:
  단일 neuron이 여러 unrelated
  concept에 활성화
  Olah 2020 observation

Toy Model (Elhage 2022):
  x̂ = W^T W x, m features > n hidden
  Sparse x 가정
  
  Optimal: near-orthogonal W columns
  Polytope structure:
    Tetrahedron (4 features, 3d)
    Pentagon (5 features, 2d)
    ...
  Equal margin 분포

Compressed Sensing 연결:
  y = Φ x (Φ: m×n, m<n, sparse x)
  L1 minimization으로 복원
  (Donoho 2006)
  
  Superposition과 동일 수학!

Feature Importance:
  덜 중요한 feature가 
  더 많이 superpose
  Sparsity가 key enabling

───── Sparse Autoencoders ─────

기본 SAE (Bricken 2023):
  Encode: f = ReLU(W_enc(x - b_dec) + b_enc)
  Decode: x̂ = W_dec f + b_dec
  Loss: ‖x - x̂‖² + λ ‖f‖_1

Monosemanticity:
  L1 sparsity → feature가 
  개별 neuron에 할당
  Each SAE feature = 특정 concept

Dictionary Learning 전통:
  Olshausen & Field 1996
  Overcomplete basis (m ≫ d)
  Sparse coding

Overcomplete:
  d_sae = 2-32× d_model
  Claude Sonnet: 34M features!

Problems:
  Dead features (never activate)
  Feature splitting (concept 여러 개로)

Solutions:
  Top-K SAE:
    Force top-k activations
    No dead features
  
  JumpReLU (Gao 2024):
    Discrete threshold
    Better sparsity-recon

Scaling (Templeton 2024):
  Claude 3 Sonnet
  Feature atlas:
    "Golden Gate Bridge"
    "Deception"
    "Python code"
    ...
  34M interpretable features

Gemma Scope (2024):
  Open SAE suite
  모든 layer, sparsity
  Comparative research 가능

───── Feature Steering ─────

Activation Steering:
  h_new = h + α · v_steer
  Inference-time behavior change

Refusal Direction (Arditi 2024):
  Harmful prompt - harmless prompt
  Mean activation difference
  이 direction 뺄 때 → jailbreak

CAA (Rimsky 2024):
  Contrastive examples (pos vs neg)
  Mean diff = steering vector
  Sycophancy, truthfulness 조절

Model Editing:
  ROME (Meng 2022):
    Factual knowledge locate
    Specific MLP layer 저장
    Rank-1 update: W' = W + vk^T
  
  MEMIT (Meng 2023):
    Multiple edits
    Batched updates

───── 실제 Circuit 사례 ─────

IOI Circuit (Wang 2022):
  Task: "John gave X to Mary. Mary gave Y to"
  GPT-2 Small 전체 reverse-engineered
  
  Components:
    Duplicate Token Heads
    S-Inhibition Heads
    Name Mover Heads
    Backup Name Movers
  
  Complete circuit diagram

Grokking (Nanda 2023):
  Modular arithmetic
  Pre-grokking: memorize
  Post-grokking: generalize
  
  Mechanistic finding:
    Fourier transform-like algorithm
    Network discovered "math"
  
  Progress measures:
    Trig identities
    Gradient symmetries

Variable Binding (Singh 2024):
  Complex ICL = composition
  여러 circuit 합성

───── Frontier ─────

Transcoder (Dunefsky 2024):
  SAE가 activation space, but
  Transcoder는 layer → layer
  MLP output 분해

Cross-coder (Lindsey 2024):
  여러 layer의 feature 연결
  Model 간 feature 비교
  Universality 테스트

Scalable Interpretability:
  Claude Sonnet scale
  수백만 feature
  Automatic feature labeling (by LLM)

Safety Applications:
  Refusal mechanism
  Deception detection
  Biosafety, CBRN concept detection
  Alignment research 직접 기여

───── 레포 간 연결 ─────

Transformer (Layer 3):
  Residual stream
  Attention, FFN
  Core subject

Linear Algebra (Layer 0):
  Projection, rank
  Low-rank decomposition (LoRA와 연결)

Information Theory (Layer 0):
  Entropy
  Mutual information (feature quality)

Pretrained LM (Layer 4-D):
  GPT 구조가 기본
  BERT vs GPT 차이

Kernel Methods (Layer 1):
  Sparse coding (dictionary learning)
  SAE의 기반

Generalization (Layer 2):
  Emergent abilities
  Scaling → circuit emergence

LLM Pretraining (Layer 4-B):
  Phase transition in training
  μP와의 관계
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (3~6개씩)
- 각 문서가 다루는 핵심 발견·증명 (3~4줄)
- 전체 문서 개수 확인 (34개 목표)
- Python + TransformerLens + SAE Lens 실험 환경
- Transformer, LA, Info, Pretrained LM 레포 참조 관계
- Anthropic·Neel Nanda·Chris Olah의 circuit research 맥락

**준비됐으면 1단계 구조 설계부터 시작해줘!**
