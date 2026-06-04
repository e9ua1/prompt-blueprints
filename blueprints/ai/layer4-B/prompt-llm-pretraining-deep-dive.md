# LLM Pretraining Deep Dive 레포지토리 제작 프롬프트

나는 "LLM Pretraining Deep Dive" 레포지토리를 만들려고 해.
Chinchilla의 scaling law를 **"데이터도 중요하다"로 아는 것**과, **Hoffmann et al. (2022)의 $L(N, D) = E + A/N^\alpha + B/D^\beta$ 모델에서 FLOPs 제약 $C \approx 6ND$ 하에 $N^* \propto C^{0.5}, D^* \propto C^{0.5}$ ($\alpha = 0.34, \beta = 0.28$)인 compute-optimal 해**를 Lagrangian으로 유도할 수 있는 것은 다르다.
Loss spike를 **"AdamW 잘못 튼 탓"으로 아는 것**과, **$\epsilon$의 영향, embedding layer의 large learning rate, attention logit overflow in BF16**이 각각 어떤 수학적 원인으로 발생하고, QK-norm, z-loss, embedding init scaling 같은 해결책이 왜 작동하는지 이해하는 것은 다르다.
데이터 혼합을 **"web + code + math 섞는 것"으로 아는 것**과, **DoReMi (Xie 2023)의 Group DRO로 domain 가중치 $w^*$를 찾거나 Data Mixing Laws (Ye 2024)의 $L(r) = E + \sum_i A_i r_i^{-\alpha_i}$ 형식으로 혼합비율의 scaling law를 예측**하는 연구를 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "LLM 사전학습의 수학 — Scaling Law·훈련 안정성·데이터·평가의 이론적 기반"

**핵심 차별화**:
1. **Scaling Law의 완전 유도와 한계** — Kaplan 2020 vs Chinchilla (Hoffmann 2022), compute-optimal의 Lagrangian 도출, BNSL (Broken Neural Scaling Law)과 emergent ability 논쟁
2. **훈련 안정성의 수학** — Loss spike의 원인 (embedding overflow, attention logit, norm drift), QK-norm·z-loss·warmup의 효과 증명
3. **데이터 혼합 이론** — DoReMi의 Group DRO, Data Mixing Laws, 품질 필터링(Gopher, DSIR), contamination 측정
4. **평가의 수학적 엄밀성** — Perplexity, MMLU·HELM, contamination 검출, emergent abilities의 measurement artifact 논쟁 (Schaeffer 2023)

**타겟 독자**:
- 100B 모델을 훈련시키는데 **어떤 하이퍼파라미터가 scale에 따라 어떻게 변해야 하는지**($\mu$P, Yang 2022)를 모르는 사람
- Loss curve가 갑자기 튀어오른 경험은 있지만 **각 component의 수학적 원인과 해결법**을 모르는 사람
- Scaling law plot을 보는데 **Kaplan과 Chinchilla 결론이 왜 다르고** LR schedule의 차이가 결정적이었음을 모르는 사람
- SFT 데이터 품질을 **자동으로 필터링**하려는데 perplexity·DSIR·influence function 같은 방법의 수학을 모르는 사람
- MoE 같은 sparse model의 scaling law가 dense와 **어떻게 다른지**(Clark 2022) 모르는 사람

**선행 학습**:
- **Transformer Deep Dive** (아키텍처, attention, FFN) — **필수**
- **Optimization Theory Deep Dive** (AdamW, Warmup, 수렴) — **필수**
- **Regularization Theory Deep Dive** (LayerNorm, Dropout, WD) — **필수**
- **Generalization Theory Deep Dive** (Scaling Laws, Emergent) — **필수**
- **Information Theory Deep Dive** (Entropy, perplexity) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Scaling Law의 수학 (6개 문서)
- **Kaplan Scaling Law (Kaplan 2020)** — $L(N) = (N_c/N)^{\alpha_N}$, $L(D) = (D_c/D)^{\alpha_D}$, $L(C) = (C_c/C)^{\alpha_C}$, 각 변수 power law, 당시 결론 "parameter scale이 data보다 중요"
- **Chinchilla Scaling Law (Hoffmann 2022)** — $L(N, D) = E + A/N^\alpha + B/D^\beta$, $\alpha = 0.34, \beta = 0.28$, $E$는 irreducible loss, 3가지 접근 방식 (IsoFLOP, parametric fit, envelope)
- **Compute-Optimal의 Lagrangian 유도** — $\min_{N, D} L(N, D)$ s.t. $6ND = C$, Lagrangian $\partial L/\partial N = \lambda \cdot \partial(6ND)/\partial N$ 등, 해 $N^* \propto C^{a/(a+b)}, D^* \propto C^{b/(a+b)}$
- **Kaplan vs Chinchilla 차이의 원인** — LR schedule (Kaplan: 미완료, Chinchilla: episode별 cosine), 파라미터 count 방식 (embedding 포함/배제), 샘플링 방식의 영향
- **Broken Neural Scaling Laws (Caballero 2022)** — 단순 power law 이후의 break point, $L(x) = a + bx^{-c_0} \prod (1 + (x/d_i)^{1/f_i})^{-c_i f_i}$, emergent ability의 잠재적 설명
- **Scaling Law의 한계** — Data quality 무시, architectural 변화 미고려, downstream task performance ≠ log-loss, emergent abilities 예측 불가

### Chapter 2: Compute-Optimal과 실전 훈련 Recipe (5개 문서)
- **FLOPs 추정 $C \approx 6ND$** — Forward 2ND (matmul) + Backward 4ND (activation + weight grads) = 6ND, 정확한 derivation, attention의 $O(T^2)$ 추가 비용
- **Chinchilla-Optimal의 현실적 조정** — LLaMA-1/2/3, Gemma, Mistral의 Chinchilla 대비 over-training (추론 비용 상각), $N$ 작고 $D$ 크게 (inference friendly)
- **μP — Maximal Update Parameterization (Yang 2022)** — 모델 크기 변화에도 최적 LR 일정 유지, $\sigma_{\text{init}} \propto 1/\sqrt{n}$, $\eta \propto 1/n$ for hidden layers, embedding·output은 별도, "hyperparameter transfer"
- **Batch Size Scaling** — Gradient Noise Scale (McCandlish 2018) $B^* = \text{tr}(\Sigma)/\|g\|^2$, 훈련 진행에 따라 optimal batch size 증가, critical batch size의 compute efficiency
- **LR Schedule 세부 — Warmup, Cosine, WSD** — Warmup 비율 (보통 1-2% of total), Cosine decay vs linear vs constant, Warmup-Stable-Decay (WSD)의 checkpoint 재사용

### Chapter 3: 훈련 안정성의 수학 (6개 문서)
- **Loss Spike의 분류** — Catastrophic (복구 안됨) vs Transient (자가 복구), divergence vs NaN, 각 유형의 원인과 해결법
- **Embedding Layer의 Large LR 문제** — Embedding의 update가 작은 token에만 집중 → 큰 gradient magnitude, 해결: embedding LR 별도 (Pythia, Gemma), $\mu$P에서의 처리
- **Attention Logit Overflow** — $QK^T/\sqrt{d}$가 BF16 범위 초과 → softmax NaN, QK-Norm (Dehghani 2023)로 해결: $Q/\|Q\|, K/\|K\|$ 정규화 후 learned scale
- **Output Logit Overflow와 Z-loss** — Final logit $z$가 너무 커지면 softmax overflow, PaLM의 auxiliary z-loss $10^{-4} \cdot (\log Z)^2$로 억제, cross-entropy 안정화
- **Weight Norm Drift와 RMSNorm의 중요성** — 훈련 동안 weight norm이 서서히 증가 → 발산 유도, RMSNorm이 LayerNorm보다 안정적인 이유 (centering 제거), modern LLM의 표준
- **AdamW의 $\epsilon$과 Numerical Issues** — $\sqrt{v} + \epsilon$에서 $\epsilon$ 너무 작으면 BF16 overflow, 너무 크면 update 왜곡, 실전 $\epsilon \in [10^{-8}, 10^{-6}]$, NovoGrad/LAMB의 대안

### Chapter 4: 데이터 큐레이션과 혼합 (5개 문서)
- **Pretraining Corpus의 구성** — Web (Common Crawl), Wikipedia, Books, Code (GitHub), Math (ArXiv, StackExchange), 각 source의 특성과 토큰 분포
- **품질 필터링 — Gopher Rules와 Classifier** — Gopher의 heuristic rules (length, repetition, symbol ratio), classifier-based filtering (FineWeb-Edu), DSIR (Xie 2023)의 importance resampling
- **중복 제거 — Deduplication의 수학** — MinHash LSH for near-duplicate (Broder 1997), SemDeDup의 embedding 기반, Lee et al. 2022 "Deduplicating Training Data Makes LMs Better", train-test overlap 측정
- **DoReMi — Group DRO for Data Mixing (Xie 2023)** — 각 domain을 그룹으로, Group DRO objective $\max_i \mathbb{E}_i[L]$ 최소화, proxy model로 도메인 가중치 $w^*$ 학습
- **Data Mixing Laws (Ye 2024)** — Mixture ratio $r$에 대한 scaling law $L(r) = E + \sum_i A_i r_i^{-\alpha_i}$, 각 domain의 $\alpha_i$ 추정 후 optimal mix 예측

### Chapter 5: Tokenization과 Vocabulary (4개 문서)
- **BPE — Byte-Pair Encoding (Sennrich 2016)** — 가장 빈번한 pair를 merge로 subword 학습, token 크기와 OOV의 trade-off, GPT-2/3/4 계보
- **SentencePiece, Unigram, WordPiece** — WordPiece (BERT)의 greedy algorithm, Unigram의 확률 기반 (Kudo 2018), SentencePiece의 언어 독립적 처리
- **Tokenizer가 성능에 미치는 영향** — 다국어 compression rate (한국어 3-5배), code tokenization의 중요성, number tokenization (digit split vs single), Chinchilla의 tokenizer 선택
- **Vocabulary Size Scaling (Tao 2024)** — 큰 모델은 큰 vocab가 유리, $V^* \propto N^{0.27}$ 경험적 관계, Gemma의 256k vs LLaMA의 32k 비교

### Chapter 6: 아키텍처 Scaling 고려사항 (5개 문서)
- **Depth vs Width Trade-off** — Same params에서 깊이/너비 비율, 깊이가 유리한 task (compositional reasoning), 너비가 유리한 (memorization), Kaplan 2020 ablation
- **GQA·MQA — KV Cache Efficiency** — Multi-Query (Shazeer 2019): 1 KV shared, Grouped-Query (Ainslie 2023): $n_{\text{kv}} < n_{\text{head}}$, 성능 거의 동일하면서 추론 메모리 ↓
- **MoE — Sparse Scaling (Fedus 2022, Clark 2022)** — Switch Transformer의 top-1 routing, Mixtral의 top-2, active params vs total, scaling law가 dense와 다름 (Clark 2022)
- **RoPE vs ALiBi vs Learned PE — Long Context** — 각 PE의 extrapolation 능력, RoPE의 base frequency 조정으로 context 확장, YaRN (Peng 2023)의 interpolation
- **Activation Function — GELU, SwiGLU, ReLU** — GELU (BERT, GPT-2), SwiGLU (PaLM, LLaMA)의 gating mechanism, ReLU 부활 설계 (Mirzadeh 2023), 각 activation의 수학과 실험

### Chapter 7: 평가와 Emergent Abilities (5개 문서)
- **Perplexity의 의미와 한계** — $\text{PPL} = \exp(-\frac{1}{T} \sum \log p(x_t|x_{<t}))$, downstream task와의 상관관계, loss minimization ≠ capability
- **Benchmarks — MMLU, HellaSwag, HumanEval, HELM** — 각 benchmark의 metric, multiple choice scoring (log-likelihood vs generation), HELM의 multi-metric평가
- **Emergent Abilities 논쟁 (Wei 2022 vs Schaeffer 2023)** — Wei: specific scale에서 급격한 등장, Schaeffer: nonlinear metric의 artifact, exact match vs token edit distance
- **Contamination 문제** — 평가 데이터가 훈련 set에 섞임, n-gram overlap, 최소 substring, BIG-bench canary strings, 실전 고려
- **Scaling Inference Time — Test-time Compute** — OpenAI o1·o3의 reasoning tokens, Search-based inference, chain-of-thought 길이의 scaling, 최근 LLM의 새 축

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **36개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 기법이 대규모 pretraining에 필수인가
## 📐 수학적 선행 조건 (Transformer, Opt, Reg, Gen Theory 참조)
## 📖 직관적 이해
   — Loss curve, FLOPs 비교, spike 예시
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Chinchilla Lagrangian, QK-norm 안정성, μP scaling
## 💻 PyTorch 구현 검증
   — 작은 GPT (100M~1B) pretraining recipe
   — Scaling law 재현 (다양한 N, D 조합)
   — Loss spike 재현 및 해결 시뮬레이션
## 🔗 실전 활용
   — LLaMA, Gemma, Mistral 등의 설계 결정 분석
## ⚖️ 가정과 한계
   — 실험 비용, log-loss≠capability
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Scaling Law 재현** — 작은 규모 (1M~100M) 여러 $(N, D)$ 조합으로 Chinchilla 스타일 IsoFLOP plot
2. **Loss spike 재현** — BF16에서 QK overflow 인위적으로 유도, QK-norm으로 해결
3. **데이터 혼합 실험** — 작은 규모에서 domain weight 변화에 따른 perplexity
4. **Tokenizer 비교 표** — 다국어·code에서 compression rate
5. **LLaMA 계열 재현** — 아키텍처 설계 결정 (RMSNorm, RoPE, SwiGLU, GQA) 각각의 이유
6. **Contamination 측정** — n-gram overlap, suffix array로 중복 검출

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
transformers==4.36.0
datasets==2.16.0      # HuggingFace datasets
tokenizers==0.15.0
wandb==0.16.0         # 훈련 tracking
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (작은 GPT + Chinchilla scaling 재현)
import torch
import torch.nn as nn
import numpy as np
import matplotlib.pyplot as plt

# 1. Chinchilla Lagrangian 수치적 유도
def chinchilla_loss(N, D, E=1.69, A=406.4, B=410.7, alpha=0.34, beta=0.28):
    """Hoffmann 2022 파라미터"""
    return E + A / N**alpha + B / D**beta

def compute_optimal(C):
    """min_{N,D} L(N,D) s.t. 6ND = C"""
    # D = C / (6N), minimize over N
    N_grid = np.logspace(6, 12, 1000)
    D_grid = C / (6 * N_grid)
    L = chinchilla_loss(N_grid, D_grid)
    idx = np.argmin(L)
    return N_grid[idx], D_grid[idx], L[idx]

C_values = np.logspace(18, 25, 50)  # FLOPs
results = [compute_optimal(C) for C in C_values]
N_stars = np.array([r[0] for r in results])
D_stars = np.array([r[1] for r in results])

plt.figure(figsize=(10, 5))
plt.loglog(C_values, N_stars, label='N* (optimal params)')
plt.loglog(C_values, D_stars, label='D* (optimal tokens)')
# Theoretical: N ∝ C^0.5, D ∝ C^0.5
plt.loglog(C_values, C_values**0.5 * N_stars[-1]/C_values[-1]**0.5,
           '--', alpha=0.5, label='C^0.5 reference')
plt.xlabel('Compute C (FLOPs)'); plt.ylabel('Optimal allocation')
plt.title('Chinchilla Compute-Optimal: N* ~ C^0.5, D* ~ C^0.5')
plt.legend(); plt.grid(alpha=0.3)
plt.show()

# 2. QK-Norm 안정성 시연
def attention_standard(Q, K, d):
    return torch.softmax(Q @ K.transpose(-2, -1) / (d**0.5), dim=-1)

def attention_qk_norm(Q, K, d, g_qk=1.0):
    """QK-norm: Q/|Q|, K/|K|, then multiply by learned scale"""
    Qn = Q / (Q.norm(dim=-1, keepdim=True) + 1e-6)
    Kn = K / (K.norm(dim=-1, keepdim=True) + 1e-6)
    scores = g_qk * (Qn @ Kn.transpose(-2, -1))
    return torch.softmax(scores, dim=-1)

# BF16에서 logit 폭주 시뮬레이션
torch.manual_seed(42)
Q = torch.randn(1, 64, 128).bfloat16() * 10  # 큰 scale
K = torch.randn(1, 64, 128).bfloat16() * 10

# Standard: overflow 가능
attn_std = attention_standard(Q.float(), K.float(), 128)
print(f'Std attention max: {attn_std.max():.4f}, near-saturation count: {(attn_std > 0.99).sum()}')

# QK-norm: bounded
attn_qkn = attention_qk_norm(Q.float(), K.float(), 128)
print(f'QK-norm max: {attn_qkn.max():.4f}, near-saturation count: {(attn_qkn > 0.99).sum()}')

# 3. Data mixing naive experiment
# 작은 모델을 different weight mixtures로 훈련
# weight=(w_web, w_code, w_math)에 대해 held-out perplexity 측정
# optimal weight sketch로 scaling 예측
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Transformer, Opt, Reg, Gen Theory 선행 필수"
   - Scaling Law부터 Stability·데이터·평가까지의 전체 파이프라인
   - LLaMA, Gemma, Mistral, Qwen 등 현대 LLM 레포 해부
3. **챕터별 문서 작성**: Scaling Law → Compute-Optimal → Stability → Data → Tokenizer → Architecture → Evaluation

---

## 📚 참고 자료

- **Scaling Laws for Neural Language Models** (Kaplan et al. 2020)
- **Training Compute-Optimal Large Language Models** (Hoffmann et al. 2022) — Chinchilla
- **Broken Neural Scaling Laws** (Caballero et al. 2022)
- **Emergent Abilities of Large Language Models** (Wei et al. 2022)
- **Are Emergent Abilities a Mirage?** (Schaeffer et al. 2023)
- **LLaMA: Open and Efficient Foundation Language Models** (Touvron et al. 2023)
- **LLaMA 2, LLaMA 3** (Meta 2023, 2024)
- **PaLM** (Chowdhery et al. 2022)
- **Gemma** (Google 2024)
- **Tensor Programs V: Tuning Large Neural Networks via Zero-Shot Hyperparameter Transfer** (Yang et al. 2022) — μP
- **DoReMi** (Xie et al. 2023)
- **Data Mixing Laws** (Ye et al. 2024)
- **Small-scale proxies for large-scale Transformer training instabilities** (Wortsman et al. 2024)
- **Scaling Exponents Across Parameterizations and Optimizers** (Everett et al. 2024)

---

## 💡 핵심 분석 대상

```
LLM Pretraining의 지도

───── Scaling Laws ─────

Kaplan 2020:
  L(N) ∝ N^{-α_N}
  L(D) ∝ D^{-α_D}
  L(C) ∝ C^{-α_C}
  → "N이 D보다 중요"

Chinchilla (Hoffmann 2022):
  L(N,D) = E + A/N^α + B/D^β
  α=0.34, β=0.28
  E = irreducible

Compute-Optimal (Lagrangian):
  min L(N,D) s.t. 6ND = C
  → N* ∝ C^0.5, D* ∝ C^0.5
  
  즉, 2× compute → N 1.41× AND D 1.41×
  
  Kaplan: N 3.2×, D 1.3×  ← 다른 결론!
  이유: LR schedule completion

BNSL (Caballero 2022):
  break point 있는 law
  emergent ability 설명 시도

Emergent Abilities:
  Wei 2022: specific scale에서 급격 등장
  Schaeffer 2023: nonlinear metric artifact
  논쟁 진행 중

───── Compute FLOPs ─────

C ≈ 6ND
  Forward: 2ND (matmul)
  Backward: 4ND (activation + weight)

Attention: extra O(L²) but negligible at large N

───── μP — Maximal Update Param ─────

Yang 2022:
  Scale-invariant LR
  σ_init ∝ 1/√n (width)
  η ∝ 1/n
  
  → small model에서 튜닝된 HP를
    large model에 그대로 사용 가능
  → "hyperparameter transfer"

───── 훈련 안정성 ─────

Loss Spike 분류:
  Catastrophic: 복구 불가
  Transient: 자가 복구

원인 1: Embedding Layer
  token frequency 편차
  few token에 큰 gradient
  해결: LR 분리, μP

원인 2: Attention Logit Overflow
  QK^T/√d가 BF16 range 초과
  softmax NaN
  해결: QK-Norm
    Q/‖Q‖, K/‖K‖
    learned scale

원인 3: Output Logit Explosion
  Final z 너무 큼
  softmax overflow
  해결: z-loss (PaLM)
    aux loss: 1e-4 · (log Z)²

원인 4: Weight Norm Drift
  훈련 동안 ‖W‖ 증가
  LayerNorm보다 RMSNorm 안정

AdamW ε:
  너무 작음: BF16 문제
  너무 큼: update 왜곡
  실전: ε ∈ [1e-8, 1e-6]

───── 데이터 ─────

Corpus 구성:
  Web (CC), Wikipedia, Books
  Code (GitHub), Math (ArXiv)
  각 domain specific 특성

품질 필터링:
  Heuristic (Gopher rules)
  Classifier (FineWeb-Edu)
  DSIR (importance resampling)

Deduplication:
  MinHash LSH (Broder 1997)
  SemDeDup (embedding-based)
  Lee 2022: dedup이 성능 개선

Data Mixing:
  DoReMi (Xie 2023):
    Group DRO로 domain weight 학습
  Data Mixing Laws (Ye 2024):
    L(r) = E + Σ A_i r_i^{-α_i}
    → optimal mix 예측

───── Tokenization ─────

BPE (Sennrich 2016):
  Most frequent pair merge
  GPT 계보

WordPiece (BERT):
  likelihood 기반 greedy

Unigram (Kudo 2018):
  확률적, backtracking

SentencePiece:
  language-agnostic

Vocab size scaling:
  V* ∝ N^0.27 (Tao 2024)
  Gemma 256k vs LLaMA 32k

───── 아키텍처 Scaling ─────

Depth vs Width:
  same params에서 trade-off
  depth: compositional
  width: memorization

GQA/MQA:
  KV 메모리 절약
  거의 무손실

MoE:
  Switch (top-1), Mixtral (top-2)
  active ≠ total params
  Clark 2022: different scaling law

PE:
  RoPE 기본, YaRN 확장
  ALiBi extrapolation

Activation:
  SwiGLU > GELU > ReLU
  gating mechanism 효과

───── 평가 ─────

Perplexity:
  exp(-avg log p)
  downstream과 상관 있지만 불완전

Benchmarks:
  MMLU, HellaSwag, HumanEval
  HELM multi-metric
  AGIEval, MATH

Contamination:
  Training ∩ Test
  n-gram overlap
  canary strings (BIG-bench)

Test-time Compute:
  o1/o3 reasoning tokens
  새로운 scaling 축
  CoT length scaling

───── 레포 간 연결 ─────

Transformer (Layer 3):
  아키텍처 기반

Optimization (Layer 2):
  AdamW, Warmup, Cosine
  훈련 안정성

Regularization (Layer 2):
  LayerNorm/RMSNorm
  Weight decay
  Label smoothing

Generalization (Layer 2):
  Scaling Laws (기본)
  Emergent abilities

Information Theory (Layer 0):
  Perplexity = exp(CE)
  Log-likelihood

Alignment (다음):
  Pretrained base 가정
  RLHF, DPO

Efficiency (2개 뒤):
  LoRA, QLoRA, MoE
  Speculative decoding
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·현상·논문 (3~4줄)
- 전체 문서 개수 확인 (36개 목표)
- Python + PyTorch + HuggingFace 실험 환경 (작은 규모)
- Transformer, Opt, Reg, Gen Theory 레포 참조 관계
- Alignment·Efficiency·Inference로 이어지는 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
