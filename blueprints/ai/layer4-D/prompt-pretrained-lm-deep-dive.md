# Pretrained LM Deep Dive 레포지토리 제작 프롬프트

나는 "Pretrained LM Deep Dive" 레포지토리를 만들려고 해.
BERT를 **"양방향 Transformer"로 아는 것**과, **Devlin et al. (2019)의 MLM에서 15% mask ratio가 왜 그 숫자이고 (10% random + 10% unchanged + 80% [MASK]의 triple noise가 pretraining-finetuning 불일치를 줄이는 수학적 동기)**, **NSP가 왜 RoBERTa (Liu 2019)에서 제거되었는지 ablation 근거**를 이해하는 것은 다르다.
GPT의 자기회귀를 **"next token prediction"으로 아는 것**과, **Radford et al. (2018, 2019, 2020)의 log-likelihood $\sum_t \log p(x_t | x_{<t})$가 causal attention mask로 어떻게 parallel training을 가능케 하고**, **GPT-1→2→3의 scale 진화가 emergent abilities(Wei 2022)와 어떻게 연결**되는지 이해하는 것은 다르다.
In-Context Learning을 **"few-shot prompting"으로 아는 것**과, **Akyürek et al. (2023), von Oswald et al. (2023)의 "attention = learned gradient descent" 해석이 어떻게 linear regression task에서 ICL을 implicit gradient update로 수식화**하는지 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "Pretrained LM의 수학 — BERT·GPT·T5의 objective 분해와 전이학습·ICL 이론"

**핵심 차별화**:
1. **BERT MLM의 정보이론적 분석** — 15% mask ratio 선택, 80/10/10 rule, Span Masking (SpanBERT), MAE의 75%와 대비
2. **GPT Autoregressive의 수학적 특성** — Causal attention mask, teacher forcing, parallel training with triangular mask
3. **Transfer Learning의 이론** — ULMFiT (Howard 2018)의 discriminative fine-tuning + slanted triangular LR + gradual unfreezing, 현대 LLM fine-tuning의 기원
4. **In-Context Learning 해석** — Brown 2020의 경험적 현상, Akyürek·von Oswald 2023의 attention-as-GD 이론, reasoning·CoT emergence

**타겟 독자**:
- BERT·GPT 모두 사용하지만 **encoder-only vs decoder-only vs encoder-decoder 중 어느 task에 어느 것이 적합**한지 수학적으로 설명 못하는 사람
- RoBERTa의 개선점 (static vs dynamic masking, NSP 제거, 더 큰 batch·더 긴 훈련)의 각 기여를 **ablation 수학**으로 이해 못하는 사람
- T5의 "everything is text-to-text"가 **span corruption objective**로 어떻게 unified training을 가능케 하는지 모르는 사람
- ICL이 왜 작동하는지 **implicit gradient descent 해석**의 수식을 못 따라가는 사람
- Chain-of-Thought가 어떤 모델 규모에서 emergent하게 나타나는지 (Wei 2022 Fig 3) 수학적 특성을 모르는 사람

**선행 학습**:
- **Transformer Deep Dive** (Attention, Encoder/Decoder) — **필수**
- **NLP Foundations Deep Dive** (Word2Vec 이후 static embedding 한계) — **필수**
- **LLM Pretraining Deep Dive** (Scaling Law, 훈련) — **필수**
- **Generalization Theory Deep Dive** (Emergent abilities) — **필수**
- **Information Theory Deep Dive** (MLM의 정보이론) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Contextual Embedding의 등장 (4개 문서)
- **Static Embedding의 한계** — Polysemy 문제 ("bank"의 river vs money), context 무시, sentence-level 모델 부재, Word2Vec/GloVe의 ceiling
- **ELMo — Embeddings from Language Models (Peters 2018)** — 양방향 LSTM LM, 각 layer의 weighted sum이 contextual embedding, character-CNN input, feature-based transfer
- **ULMFiT — Universal LM Fine-tuning (Howard 2018)** — LM pretrain → LM fine-tune → task fine-tune 3단계, discriminative fine-tuning (layer별 다른 LR), slanted triangular LR, gradual unfreezing
- **Transformer가 가져온 변화** — Attention Is All You Need (Vaswani 2017) → pretrained LM 폭발, RNN 대비 병렬성·long-range context, BERT·GPT의 전신

### Chapter 2: BERT — Encoder-Only의 수학 (6개 문서)
- **BERT 아키텍처 (Devlin 2019)** — Bidirectional Transformer encoder, [CLS] + tokens + [SEP], WordPiece tokenization, segment embeddings for pair tasks
- **Masked Language Modeling의 정보이론** — 15% mask ratio의 경험적 최적 ($\sim \log 2$ bit per position), $x \setminus x_M$에서 $x_M$ 복원, Gibbs sampling과 유사
- **80/10/10 Rule의 필요성** — 80% [MASK], 10% random token, 10% unchanged, pretrain-finetune gap 해소 (finetune에는 [MASK] 없음), robustness
- **Next Sentence Prediction (NSP)** — [CLS] 임베딩으로 두 문장 연속성 예측, 긍정 샘플과 부정 샘플 50:50, QA·NLI task 겨냥, 나중에 논란
- **RoBERTa의 개선 (Liu 2019)** — Dynamic masking (매 step 다른 mask), NSP 제거 (improves), 더 큰 batch (8K), 더 긴 훈련 (500K steps), 10배 많은 데이터
- **SpanBERT, DistilBERT, ALBERT** — SpanBERT (Joshi 2020)의 span masking, DistilBERT (Sanh 2019)의 knowledge distillation (40% smaller, 97% 성능), ALBERT의 parameter sharing

### Chapter 3: GPT — Decoder-Only의 계보 (5개 문서)
- **GPT-1 (Radford 2018)** — Generative pretraining + discriminative fine-tuning, decoder-only Transformer, Causal (left-to-right) attention mask, log-likelihood $L = \sum_t \log p(x_t | x_{<t})$
- **Causal Attention Mask의 수학** — Lower triangular mask, $\text{softmax}(QK^T + M)$ where $M_{ij} = -\infty$ for $i < j$, parallel training via triangular matrix
- **GPT-2 (Radford 2019)** — 1.5B parameters (10배 scale), zero-shot task transfer, "language models are unsupervised multitask learners", few-shot ICL의 초기 증거
- **GPT-3 (Brown 2020)** — 175B parameters, in-context learning의 명확한 등장, few-shot·one-shot·zero-shot 정량 비교, prompting paradigm 정착
- **Modern Decoder-Only LLM 계보** — LLaMA, Mistral, Gemma, Qwen, DeepSeek의 아키텍처 공통점 (RMSNorm, SwiGLU, RoPE, GQA), GPT의 계승과 변주

### Chapter 4: T5 — Encoder-Decoder와 Unified Framework (4개 문서)
- **T5 — Text-to-Text Transfer Transformer (Raffel 2020)** — 모든 NLP task를 text-to-text로, encoder-decoder 아키텍처, MNLI·SQuAD·CNN/DM 등 통합 훈련, C4 dataset (750GB)
- **Span Corruption Objective** — 15% token을 variable-length span으로 mask, sentinel token으로 대체, encoder 입력은 corrupted, decoder는 원본 span 예측, BERT MLM의 generative 버전
- **Prefix LM과 UL2 (Tay 2022)** — Bidirectional on prefix + causal on suffix, UL2의 Mixture-of-Denoisers (R-denoiser, S-denoiser, X-denoiser), 다양한 objective 통합
- **Architecture 선택 — Encoder-only vs Decoder-only vs Enc-Dec** — Task 특성에 따른 적합성, Raffel 2020 Table 2 ablation, 현대 LLM이 decoder-only로 수렴한 이유 (ICL, generation quality)

### Chapter 5: Transfer Learning의 이론 (5개 문서)
- **Fine-Tuning vs Linear Probe** — Feature extractor로만 쓰기 vs 전체 파라미터 업데이트, 각 방식의 성능·계산 비교, 언제 어느 방법
- **Discriminative Fine-Tuning (Howard 2018)** — Layer별 learning rate $\eta^l = \eta^{l-1}/2.6$, 상위 layer가 task-specific, 하위 layer가 일반적 feature, catastrophic forgetting 방지
- **Slanted Triangular LR Schedule** — Linear warmup + linear decay, cyclical 변형, 훈련 안정성, LLM 시대의 cosine schedule과 비교
- **Catastrophic Forgetting과 Elastic Weight Consolidation (EWC, Kirkpatrick 2017)** — Fine-tuning 중 pretrained knowledge 손실, Fisher Information으로 중요 parameter 식별, $L = L_{\text{task}} + \lambda \sum_i F_i (\theta_i - \theta^*_i)^2$
- **PEFT와 Fine-tuning 효율 (LLM Efficiency 참조)** — LoRA·Adapter 등으로 효율 확보, full fine-tuning의 memory·overfitting 문제 해결

### Chapter 6: In-Context Learning의 이론 (5개 문서)
- **ICL 현상 (Brown 2020)** — Prompt 내에 $k$개 demo $(x_i, y_i)$ + test $x$, weight update 없이 task 수행, scale에 emergent
- **ICL의 Empirical 성질** — Demo 수에 따른 성능, permutation invariance 위반, label-text correlation의 중요도 (Min 2022 등)
- **Attention as Gradient Descent (Akyürek 2023, von Oswald 2023)** — Linear regression task에서 ICL이 1-step GD와 수학적 동등, induction head mechanism (Olsson 2022), attention pattern 분석
- **Transformers Learn In-Context by Gradient Descent** — von Oswald 공식: attention output이 $\theta \to \theta - \eta \nabla L$ 한 step과 동치, weight matrix가 implicit optimizer 저장
- **Task Vector와 Function Vector (Hendel 2023, Todd 2024)** — ICL 시 특정 hidden state가 task-specific vector encoding, activation steering으로 task 전환 가능, mechanistic interpretability

### Chapter 7: Prompting과 Reasoning의 이론 (4개 문서)
- **Instruction Tuning (FLAN, T0, Wei 2022)** — Natural language instruction으로 multi-task fine-tuning, zero-shot generalization 크게 개선, Super-NaturalInstructions (1616 task)
- **Chain-of-Thought (CoT, Wei 2022)** — "Let's think step by step" 또는 few-shot reasoning demonstration, 특정 scale (∼62B) 이상에서 emergent, reasoning capability 해방
- **Self-Consistency와 Tree of Thoughts** — Wang 2023의 multiple CoT sampling + majority vote, ToT (Yao 2023)의 tree search, reasoning의 계산 확장
- **Emergent Abilities와 Measurement Artifact 논쟁** — Wei 2022의 발견, Schaeffer 2023의 반론 (non-linear metric의 artifact), 실제 capability jump 여부 논쟁

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **33개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 모델이 pretrained LM의 milestone인가
## 📐 수학적 선행 조건 (Transformer, NLP Found, LLM Pretrain, Gen 참조)
## 📖 직관적 이해
   — MLM masking, causal mask, ICL demo 시각화
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — MLM 정보이론, causal mask 수학, ICL as GD
## 💻 PyTorch 구현 검증
   — 작은 BERT/GPT 훈련 (WikiText-103)
   — Fine-tuning 각 기법 비교
   — ICL 실험 재현
## 🔗 실전 활용
   — 언제 어느 아키텍처, 어떤 fine-tune 방법
## ⚖️ 가정과 한계
   — Scale dependency, hallucination
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Attention pattern 시각화** — BERT bidirectional vs GPT causal의 attention heatmap
2. **Masking ratio 실험** — 작은 BERT에서 15% vs 40% vs 75% mask ratio 성능 비교
3. **Fine-tuning 비교** — Linear probe vs full FT vs LoRA on GLUE subset
4. **ICL 곡선** — Demo 수에 따른 성능, model size별 비교
5. **CoT 효과** — 수학 문제에서 CoT 유무의 정확도 차이
6. **Induction head 측정** — Small GPT에서 induction circuit 발견 (Olsson 2022)

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
transformers==4.36.0
datasets==2.16.0
peft==0.7.0
matplotlib==3.8.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (작은 BERT MLM + GPT causal + ICL linear regression)
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np
import matplotlib.pyplot as plt

# 1. Causal attention mask (GPT)
def causal_mask(T):
    """Lower triangular: 1 where i >= j"""
    return torch.tril(torch.ones(T, T))

def causal_attention(Q, K, V):
    T = Q.size(-2)
    d = Q.size(-1)
    scores = Q @ K.transpose(-2, -1) / d**0.5
    mask = causal_mask(T).unsqueeze(0).unsqueeze(0)
    scores = scores.masked_fill(mask == 0, -1e9)
    attn = F.softmax(scores, dim=-1)
    return attn @ V

# 2. MLM masking (BERT)
def bert_mask(tokens, mask_id, vocab_size, ratio=0.15):
    """80/10/10 rule"""
    B, T = tokens.shape
    labels = tokens.clone()
    # Sample mask positions
    prob = torch.rand(B, T)
    mask_positions = prob < ratio
    
    # 80% [MASK]
    mask_token = mask_positions & (torch.rand(B, T) < 0.8)
    # 10% random
    random_token = mask_positions & (torch.rand(B, T) < 0.5) & ~mask_token
    # 10% unchanged (남은 것)
    
    tokens_new = tokens.clone()
    tokens_new[mask_token] = mask_id
    tokens_new[random_token] = torch.randint(0, vocab_size, ((random_token).sum(),))
    
    # Labels: only predict masked positions
    labels[~mask_positions] = -100  # ignored in loss
    return tokens_new, labels

# 3. ICL as Gradient Descent (1D linear regression)
# Task: f(x) = w x, prompt: [(x_1, y_1), (x_2, y_2), ..., x_test]
# Test: Transformer가 implicit하게 w를 학습하는가?

class SimpleTransformerForICL(nn.Module):
    def __init__(self, dim=16, n_heads=2, n_layers=4):
        super().__init__()
        self.input = nn.Linear(2, dim)  # (x, y) pair
        self.blocks = nn.ModuleList([
            nn.TransformerEncoderLayer(dim, n_heads, dim*4, batch_first=True)
            for _ in range(n_layers)
        ])
        self.out = nn.Linear(dim, 1)
    def forward(self, xy_seq):
        """xy_seq: [B, L, 2], first L-1 are (x,y) pairs, last is (x_test, 0)"""
        h = self.input(xy_seq)
        for blk in self.blocks: h = blk(h)
        return self.out(h[:, -1])

def icl_task(batch_size=256, L=20):
    """Generate ICL data for linear regression"""
    w = torch.randn(batch_size, 1)  # hidden param
    x = torch.randn(batch_size, L, 1) * 0.5
    y = x * w.unsqueeze(-1)
    # Last input: y=0 (to predict)
    xy = torch.cat([x, y], dim=-1)
    xy[:, -1, 1] = 0  # remove y_test
    y_test = y[:, -1].squeeze(-1)
    return xy, y_test

model = SimpleTransformerForICL()
opt = torch.optim.Adam(model.parameters(), lr=1e-3)

# Train on ICL task
losses = []
for step in range(3000):
    xy, y_true = icl_task()
    y_pred = model(xy).squeeze(-1)
    loss = ((y_pred - y_true)**2).mean()
    opt.zero_grad(); loss.backward(); opt.step()
    losses.append(loss.item())

# Compare to optimal GD solution
# After k demos, optimal w_hat = (Σ x_i y_i) / (Σ x_i²)
# Transformer should learn similar mapping

# 4. Discriminative Fine-tuning (Howard 2018)
def discriminative_lr(base_lr, n_layers, factor=2.6):
    """Layer별 다른 LR: 상위가 큰 LR"""
    return [base_lr / factor**(n_layers - l - 1) for l in range(n_layers)]

# 5. CoT prompt example
cot_prompt = """Q: There are 5 apples. I eat 2. How many left?
A: Let's think step by step.
- Start with 5 apples.
- Eat 2 apples.
- 5 - 2 = 3.
So the answer is 3.

Q: There are 7 birds. 3 fly away and 2 more arrive. How many birds?
A: Let's think step by step."""

# 작은 model에서는 CoT 효과 미미, 62B+에서 emergent
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Transformer, NLP Foundations, LLM Pretrain, Generalization 선행 필수"
   - BERT-GPT-T5를 통합 비교
   - ICL·CoT 등 emergent phenomena
3. **챕터별 문서 작성**: Contextual 등장 → BERT → GPT → T5 → Transfer → ICL → Prompting

---

## 📚 참고 자료

- **Deep contextualized word representations** (Peters et al. 2018) — ELMo
- **Universal Language Model Fine-tuning for Text Classification** (Howard & Ruder 2018) — ULMFiT
- **Improving Language Understanding by Generative Pre-Training** (Radford et al. 2018) — GPT-1
- **Language Models are Unsupervised Multitask Learners** (Radford et al. 2019) — GPT-2
- **Language Models are Few-Shot Learners** (Brown et al. 2020) — GPT-3
- **BERT: Pre-training of Deep Bidirectional Transformers** (Devlin et al. 2019)
- **RoBERTa: A Robustly Optimized BERT** (Liu et al. 2019)
- **SpanBERT** (Joshi et al. 2020)
- **ALBERT** (Lan et al. 2020)
- **DistilBERT** (Sanh et al. 2019)
- **Exploring the Limits of Transfer Learning with T5** (Raffel et al. 2020)
- **UL2: Unifying Language Learning Paradigms** (Tay et al. 2022)
- **Finetuned Language Models Are Zero-Shot Learners** (Wei et al. 2022) — FLAN
- **Multitask Prompted Training Enables Zero-Shot Task Generalization** (Sanh et al. 2022) — T0
- **Chain-of-Thought Prompting Elicits Reasoning** (Wei et al. 2022)
- **Self-Consistency Improves Chain of Thought** (Wang et al. 2023)
- **Tree of Thoughts** (Yao et al. 2023)
- **What learning algorithm is in-context learning?** (Akyürek et al. 2023)
- **Transformers learn in-context by gradient descent** (von Oswald et al. 2023)
- **Emergent Abilities of Large Language Models** (Wei et al. 2022)
- **Are Emergent Abilities a Mirage?** (Schaeffer et al. 2023)
- **In-context Learning and Induction Heads** (Olsson et al. 2022)
- **Overcoming catastrophic forgetting in neural networks** (Kirkpatrick et al. 2017) — EWC

---

## 💡 핵심 분석 대상

```
Pretrained LM의 지도

───── Static → Contextual ─────

Static (W2V, GloVe):
  한 단어 = 한 벡터
  Polysemy 불가

ELMo (Peters 2018):
  Bi-LSTM LM
  Layer별 weighted sum
  Feature-based

ULMFiT (Howard 2018):
  LM pretrain → LM FT → Task FT
  Discriminative LR
  Slanted Triangular
  Gradual Unfreezing

→ Transformer로 폭발

───── BERT (Encoder-only) ─────

Devlin 2019:
  Bidirectional Transformer
  [CLS] x_1 ... x_n [SEP]
  WordPiece tokenization

MLM (Masked Language Modeling):
  15% tokens mask
  80% [MASK]
  10% random
  10% unchanged
  
  왜 15%?
    너무 작음: signal 부족
    너무 큼: context 손실
    경험적 최적
  
  왜 80/10/10?
    Finetune에 [MASK] 없음
    Pretrain-finetune gap 해소

NSP (Next Sentence):
  [CLS] 임베딩으로 pair
  50/50 positive/negative
  
  RoBERTa: NSP 제거해도 OK
    → unnecessary

RoBERTa (Liu 2019):
  ├── Dynamic masking
  ├── No NSP
  ├── 8K batch
  └── 10× more data

SpanBERT (2020):
  Span-level masking
  Span Boundary Objective

DistilBERT (Sanh 2019):
  Knowledge distillation
  40% smaller, 97% perf

───── GPT (Decoder-only) ─────

GPT-1 (Radford 2018):
  117M params
  Causal attention
  L = Σ log p(x_t | x_<t)
  
  Causal mask:
    M_{ij} = -∞ if i < j
    → triangular matrix
    Parallel training

GPT-2 (Radford 2019):
  1.5B params (10× ↑)
  Zero-shot task
  "multitask learner"

GPT-3 (Brown 2020):
  175B params (100× ↑)
  ICL emergent
  Few-shot paradigm

Modern LLMs:
  LLaMA, Mistral, Gemma...
  Decoder-only 수렴
  RMSNorm, SwiGLU, RoPE, GQA

───── T5 (Encoder-Decoder) ─────

Raffel 2020:
  Text-to-text unified
  모든 task를 text 생성으로

Span Corruption:
  15% token → variable-length span
  Sentinel tokens (<extra_id_0>, ...)
  Decoder: 원본 span 예측

Pre-training variants:
  T5: span corruption
  BART: denoising autoencoder
  UL2: Mixture-of-Denoisers
    R-denoiser, S-denoiser, X-denoiser

Architecture 선택:
  Encoder-only: classification
  Decoder-only: generation, ICL
  Enc-Dec: seq2seq (번역, 요약)
  → 현대는 decoder-only 우세

───── Transfer Learning 이론 ─────

Fine-tuning vs Linear probe:
  Linear: feature 고정, 빠름, 일반화
  Full FT: 성능↑, overfitting 위험

Discriminative LR (Howard 2018):
  η^l = η^{l-1} / 2.6
  상위 layer > 하위 layer

Slanted Triangular:
  Linear warmup + linear decay
  → LLM: cosine schedule

Catastrophic Forgetting:
  FT 중 pretrained 지식 손실
  
EWC (Kirkpatrick 2017):
  L = L_task + λ Σ F_i (θ - θ*)²
  F = Fisher Information
  중요 파라미터 pinning

Modern PEFT:
  LoRA, Adapter
  → LLM Efficiency 레포

───── In-Context Learning ─────

Phenomenon (Brown 2020):
  Prompt: [demo_1, demo_2, ..., test]
  No weight update
  → task 수행

Empirical properties:
  Demo 수 ↑ → 성능 ↑ (saturating)
  Permutation에 민감
  Label text 중요 (Min 2022)

Theoretical (Akyürek 2023, von Oswald 2023):
  Linear regression task에서
  Attention = 1-step GD
  
  Weight matrix가 implicit optimizer

Induction Head (Olsson 2022):
  Previous-token head + Match-and-copy
  Two-layer Transformer에서 mechanistic discovery
  ICL의 기초 circuit

Task/Function Vector (Hendel 2023):
  ICL 시 특정 hidden state = task encoding
  Activation steering으로 task 전환

───── Prompting & Reasoning ─────

Instruction Tuning:
  FLAN, T0
  Natural language instructions
  1616 tasks
  Zero-shot generalization ↑

Chain-of-Thought (Wei 2022):
  "Let's think step by step"
  Few-shot reasoning
  Emergent at ~62B+
  수학, symbolic reasoning

Self-Consistency (Wang 2023):
  Multiple CoT sampling
  Majority vote
  추가 compute → 추가 성능

Tree of Thoughts (Yao 2023):
  Reasoning as tree search
  BFS/DFS, backtrack
  복잡한 문제

Emergent Abilities:
  Wei 2022: scale threshold
  Schaeffer 2023: metric artifact
  논쟁 진행 중

───── 아키텍처 선택 지도 ─────

┌─────────────┬──────────────┐
│ Task        │ 권장 아키텍처 │
├─────────────┼──────────────┤
│ Classification│ Encoder-only │
│   (감정, NER) │ (BERT)       │
│             │              │
│ Generation  │ Decoder-only │
│   (chat, code)│ (GPT, LLaMA) │
│             │              │
│ Translation │ Enc-Dec      │
│ Summariz.   │ (T5, BART)   │
│             │              │
│ ICL·Prompt  │ Decoder-only │
│   (few-shot)│ (GPT-3+)     │
└─────────────┴──────────────┘

───── 레포 간 연결 ─────

Transformer (Layer 3):
  Self-attention 기초
  Positional encoding

NLP Foundations (직전):
  Word2Vec 이후
  Tokenization (BPE)

LLM Pretraining (Layer 4-B):
  Scale up + recipe
  Emergent abilities

LLM Alignment (Layer 4-B):
  Pretrained base 가정
  SFT, RLHF, DPO

LLM Efficiency (Layer 4-B):
  LoRA로 fine-tune
  Parameter-efficient

Generalization (Layer 2):
  Emergent ability 이론적 기반
  Scale → capability
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명 (3~4줄)
- 전체 문서 개수 확인 (33개 목표)
- Python + PyTorch + HF 실험 환경
- Transformer, NLP Found, LLM Pretrain, Gen 레포 참조 관계
- LLM Alignment·Efficiency로 이어지는 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
