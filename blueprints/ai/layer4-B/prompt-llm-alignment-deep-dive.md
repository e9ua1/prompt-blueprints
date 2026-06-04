# LLM Alignment Deep Dive 레포지토리 제작 프롬프트

나는 "LLM Alignment Deep Dive" 레포지토리를 만들려고 해.
RLHF를 **"인간 피드백으로 훈련"으로 아는 것**과, **Bradley-Terry preference model $P(y_1 \succ y_2 | x) = \sigma(r(x, y_1) - r(x, y_2))$이 reward model을 어떻게 정의**하고 **Christiano et al. (2017)·Ouyang et al. (2022)의 3단계 (SFT → RM → PPO) 파이프라인이 왜 그 순서여야 하는지** 유도할 수 있는 것은 다르다.
DPO를 **"reward model 없이 훈련"으로 아는 것**과, **Rafailov et al. (2023)의 closed-form $\pi^*(y|x) = \pi_{\text{ref}}(y|x) \exp(r(x,y)/\beta)/Z(x)$를 뒤집어 $r(x, y) = \beta \log(\pi^*/\pi_{\text{ref}}) + \beta \log Z$로 reward를 policy로 표현**하고 Bradley-Terry에 대입해 DPO loss $L = -\log \sigma(\beta \log \pi/\pi_{ref} (y_w) - \beta \log \pi/\pi_{ref} (y_l))$을 유도할 수 있는 것은 다르다.
Constitutional AI를 **"헌법으로 가이드"로 아는 것**과, **Bai et al. (2022)의 SL-CAI(비평·수정)와 RLAIF(AI-generated preferences)가 어떻게 RLHF의 scalability 문제를 해결**하고 **self-improvement loop의 이론적 근거**를 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "LLM Alignment의 수학 — RLHF·DPO·CAI를 preference learning의 통합 관점에서"

**핵심 차별화**:
1. **RLHF의 3단계 완전 유도** — SFT의 필요성, RM의 Bradley-Terry 전제, PPO의 KL-constrained objective
2. **DPO의 Reward-Policy Duality** — Optimal policy의 closed-form에서 reward를 policy로 재표현, 증명의 핵심 step마다 분해
3. **Constitutional AI와 RLAIF** — Scalable oversight, self-critique 메커니즘, AI feedback의 신뢰성
4. **Modern Alignment Methods** — IPO, KTO, SimPO, ORPO, GRPO 등 DPO 후계자들의 각기 다른 이론적 가정

**타겟 독자**:
- RLHF pipeline을 구현하는데 **reward hacking이 왜 발생하고 KL penalty가 어떻게 이를 억제**하는지 모르는 사람
- DPO를 쓰지만 **$\beta$ 값이 KL regularization과 어떻게 대응**되는지 수학적으로 설명 못하는 사람
- IPO·KTO·SimPO 중 **어느 것이 어떤 상황에 유리한지** 각 방법의 가정(preference model)을 비교 못하는 사람
- RLHF에서 **PPO가 아닌 A2C·TRPO·REINFORCE로 대체 가능한지**와 각각의 trade-off를 모르는 사람
- Process Reward Model (PRM)이 Outcome Reward Model (ORM)과 **수학적으로 어떻게 다른지** 모르는 사람

**선행 학습**:
- **LLM Pretraining Deep Dive** (Scaling, 훈련) — **필수**
- **Policy Gradient Deep Dive** (REINFORCE, PG Theorem) — **필수**
- **Advanced RL Deep Dive** (PPO, TRPO) — **필수**
- **Bayesian ML Deep Dive** (Bradley-Terry, preference) — 권장
- **Information Theory Deep Dive** (KL divergence) — **필수**

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Alignment 문제의 정식화 (5개 문서)
- **왜 Pretrained LM은 Misaligned인가** — Pretraining objective = next-token prediction, 인간 의도·안전·유용성과 다른 목표, "inner alignment" vs "outer alignment"
- **Preference Learning의 수학적 Framework** — 비교 데이터 $\{(x, y_w, y_l)\}$에서 $y_w \succ y_l$, reward function 추론, revealed preference theory
- **Bradley-Terry Model** — $P(y_1 \succ y_2 | x) = \sigma(r(x, y_1) - r(x, y_2))$, $r$은 hidden utility, logit link, paired comparison의 classic 모델
- **Plackett-Luce와 Ranking 일반화** — $k$개 ranking에서 $P(y_1 > y_2 > \ldots > y_k) = \prod_i \frac{e^{r_i}}{\sum_{j \geq i} e^{r_j}}$, list-wise ranking의 수학
- **Alignment Tax와 Helpfulness-Harmlessness Trade-off** — Askell et al. 2021의 HHH, alignment 후 capability 저하 문제, KL anchor로 완화

### Chapter 2: RLHF의 3단계 파이프라인 (6개 문서)
- **Supervised Fine-Tuning (SFT)** — Pretrained model을 high-quality demonstration으로 fine-tune, cross-entropy loss $-\mathbb{E}[\log \pi(y|x)]$, 뒤 단계를 위한 "format" 정렬
- **Reward Model 학습** — SFT model에서 초기화 (context 공유), classification head 부착, Bradley-Terry MLE: $L_{\text{RM}} = -\mathbb{E}[\log \sigma(r(x, y_w) - r(x, y_l))]$
- **PPO with KL Penalty** — $\max_\pi \mathbb{E}[r(x, y)] - \beta \text{KL}(\pi \| \pi_{\text{SFT}})$, reward hacking 방지, $\beta$의 역할, InstructGPT의 선택
- **Reward Hacking과 Goodhart's Law** — RM은 proxy, $\pi$가 RM을 exploit (spurious feature) → alignment 실패, KL penalty·reward model ensemble 완화
- **Iterative RLHF — Ring 1, 2, 3** — LLaMA-2의 iterative 접근, 새 데이터 → new RM → new PPO, 각 iteration의 성능 변화
- **Process vs Outcome Reward Model** — ORM: final answer 평가, PRM: reasoning step마다 평가, Lightman et al. 2023 "Let's Verify Step by Step", math·reasoning에서 PRM 우위

### Chapter 3: DPO — Direct Preference Optimization (5개 문서)
- **KL-Constrained Reward Maximization 복습** — RLHF objective $\max_\pi \mathbb{E}_{\pi}[r(x, y)] - \beta \text{KL}(\pi \| \pi_{\text{ref}})$, optimal solution의 존재성
- **Optimal Policy의 Closed Form** — $\pi^*(y|x) = \frac{1}{Z(x)} \pi_{\text{ref}}(y|x) \exp(r(x,y)/\beta)$, exponential tilting, $Z(x) = \sum_y \pi_{\text{ref}}(y|x) \exp(r/\beta)$
- **Reward as Function of Policy** — 위 식 뒤집기 $r(x, y) = \beta \log \frac{\pi^*(y|x)}{\pi_{\text{ref}}(y|x)} + \beta \log Z(x)$, $Z(x)$는 $y$에 무관 (pair 차이에서 상쇠)
- **DPO Loss 유도** — Bradley-Terry + reward reparametrization → $L_{\text{DPO}} = -\mathbb{E}[\log \sigma(\beta \log \frac{\pi(y_w|x)}{\pi_{\text{ref}}(y_w|x)} - \beta \log \frac{\pi(y_l|x)}{\pi_{\text{ref}}(y_l|x)})]$, 직접 policy 최적화
- **DPO의 장점과 한계** — RM 불필요 (simpler), 안정적, 메모리 절약 (1 model 대신 2 model); 한계: BT 가정, offline only, reward hacking의 다른 형태 (length bias)

### Chapter 4: DPO의 후계자들 (5개 문서)
- **IPO — Identity Preference Optimization (Azar 2023)** — DPO의 BT 가정 약화, $L = \mathbb{E}[(h_\pi(x, y_w, y_l) - 1/(2\beta))^2]$, overfitting 감소, deterministic preference에 robust
- **KTO — Kahneman-Tversky Optimization (Ethayarajh 2024)** — Prospect theory 기반, pair 없이 각 sample의 desirable/undesirable만 필요, $L = -\mathbb{E}[\lambda \cdot v(\beta \log \pi/\pi_{\text{ref}})]$
- **SimPO — Reference-Free (Meng 2024)** — $\pi_{\text{ref}}$ 없이 length-normalized, $L = -\mathbb{E}[\log \sigma(\frac{\beta}{|y_w|} \log \pi(y_w) - \frac{\beta}{|y_l|} \log \pi(y_l) - \gamma)]$, margin $\gamma$
- **ORPO — Odds Ratio (Hong 2024)** — SFT + preference 결합, single stage, $L = L_{\text{SFT}} + \lambda \cdot L_{\text{OR}}$, odds ratio $P(y_w)/P(y_l) / (1-P(y_w))/(1-P(y_l))$
- **GRPO — Group Relative Policy Optimization (DeepSeek 2024)** — RM 없이 group 내 상대 순위, $A_i = \frac{r_i - \bar{r}_{\text{group}}}{\sigma_{\text{group}}}$, PPO의 critic 제거, reasoning tasks

### Chapter 5: Constitutional AI와 RLAIF (4개 문서)
- **SL-CAI — Supervised Constitutional AI (Bai 2022)** — Model이 자기 output을 비평(self-critique)하고 수정(revise), constitution (일련의 principle), SFT data 자동 생성
- **RLAIF — RL from AI Feedback** — 사람 preference 대신 AI-generated, scalability, RLHF 성능 matching (Lee et al. 2024), bias 우려
- **Self-Rewarding Language Models (Yuan 2024)** — Model이 자신의 judge 역할, iterative DPO with self-generated preferences, 성능 향상 loop
- **Scalable Oversight 이론** — Irving 2018 debate, Christiano 2018 amplification, 약한 supervisor로 강한 model alignment 가능한가 (weak-to-strong generalization)

### Chapter 6: 실전 RLHF 구현 이슈 (5개 문서)
- **Response Length Bias** — RM이 긴 답변 선호 (spurious correlation), $\beta$ 증가·길이 정규화로 완화, DPO의 length exploitation
- **Reward Model Overoptimization** — Gao et al. 2023 "Scaling laws for reward model overoptimization", gold RM과 proxy RM의 차이가 벌어지는 지점, 최적 KL budget
- **Hyperparameter Sensitivity — $\beta$와 LR** — $\beta$ 너무 작음: reward 과도 추구 (hacking), 너무 큼: SFT에 묶임, 실전 $\beta \in [0.01, 0.1]$, LR은 SFT보다 10-100배 작게
- **Distribution Shift와 On-policy vs Off-policy** — PPO (on-policy): 현재 $\pi$ 샘플, DPO (off-policy): 미리 수집된 preference, 각각의 robustness
- **Evaluation — MT-Bench, AlpacaEval, LLM-as-Judge** — Pairwise evaluation, GPT-4 as judge의 한계 (bias), Chatbot Arena Elo, length bias correction

### Chapter 7: Safety와 Adversarial Robustness (4개 문서)
- **Red Teaming과 Jailbreaking** — Perez 2022 "Red Teaming Language Models", 수작업 vs 자동화된 adversarial prompt, jailbreak 분류 (role-play, prompt injection)
- **Adversarial Training for LLMs** — 공격 prompt를 훈련에 포함, constitutional redteaming (Ganguli 2022), robustness-alignment 균형
- **Refusal Mechanism과 Over-refusal** — Safety training의 trade-off, "helpful-refusal" balance, XSTest benchmark
- **Interpretability for Alignment** — Anthropic의 mechanistic interpretability, refusal direction (Arditi 2024), 특정 circuit이 refusal 담당, 편집 가능성

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **34개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 기법이 alignment에 중요한가
## 📐 수학적 선행 조건 (LLM Pretraining, PG, Advanced RL, Info 참조)
## 📖 직관적 이해
   — Reward hacking, KL anchoring 시각화
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — BT model, DPO closed-form 유도, CAI theory
## 💻 PyTorch 구현 검증
   — 작은 모델(1-7B)에서 SFT → RM → PPO 파이프라인
   — DPO 직접 구현, 여러 variant 비교
   — Reward hacking 시각화
## 🔗 실전 활용
   — InstructGPT, LLaMA-2 Chat, Claude 분석
## ⚖️ 가정과 한계
   — Preference consistency, BT validity
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Reward hacking 재현** — 간단한 RM 훈련 후 PPO에서 hacking 관찰
2. **DPO vs PPO 성능** — 같은 preference data로 두 방법 비교
3. **$\beta$ sweep** — KL divergence와 reward trade-off curve
4. **Length bias 측정** — 훈련 후 생성 길이 분포 변화
5. **Constitutional 실험** — 작은 model에서 self-critique 효과
6. **Judge bias** — LLM-as-judge의 positional bias, length bias 검증

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
transformers==4.36.0
trl==0.7.10           # HuggingFace RLHF
peft==0.7.0           # LoRA
datasets==2.16.0
accelerate==0.25.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (DPO 직접 구현)
import torch
import torch.nn.functional as F
from transformers import AutoTokenizer, AutoModelForCausalLM

def compute_dpo_loss(policy_logps, ref_logps, beta=0.1):
    """
    policy_logps: dict with 'chosen' and 'rejected' log-probs
    ref_logps: same for reference model
    """
    # Log-ratio for chosen and rejected
    policy_ratio_w = policy_logps['chosen'] - ref_logps['chosen']
    policy_ratio_l = policy_logps['rejected'] - ref_logps['rejected']
    
    # DPO loss: -log sigmoid(beta * (logr_w - logr_l))
    logits = beta * (policy_ratio_w - policy_ratio_l)
    loss = -F.logsigmoid(logits).mean()
    
    # Metrics
    rewards_w = beta * policy_ratio_w.detach()
    rewards_l = beta * policy_ratio_l.detach()
    reward_acc = (rewards_w > rewards_l).float().mean()
    
    return loss, rewards_w, rewards_l, reward_acc

def get_logp(model, input_ids, labels, attention_mask):
    """Compute log p(y|x) under model"""
    outputs = model(input_ids=input_ids, attention_mask=attention_mask)
    logits = outputs.logits[:, :-1]  # shift
    labels = labels[:, 1:]  # shift
    log_probs = F.log_softmax(logits, dim=-1)
    per_token_logp = log_probs.gather(-1, labels.unsqueeze(-1)).squeeze(-1)
    mask = attention_mask[:, 1:]
    return (per_token_logp * mask).sum(-1)

# DPO vs IPO 비교 (수식 차이)
def ipo_loss(policy_logps, ref_logps, beta=0.1):
    """IPO: (log-ratio_diff - 1/(2β))² """
    policy_ratio_w = policy_logps['chosen'] - ref_logps['chosen']
    policy_ratio_l = policy_logps['rejected'] - ref_logps['rejected']
    h = policy_ratio_w - policy_ratio_l
    loss = ((h - 1/(2*beta))**2).mean()
    return loss

# SimPO (reference-free)
def simpo_loss(policy_logps_w, policy_logps_l, len_w, len_l, beta=2.0, gamma=1.0):
    """Length-normalized, no reference"""
    logits = beta * (policy_logps_w / len_w - policy_logps_l / len_l) - gamma
    loss = -F.logsigmoid(logits).mean()
    return loss

# Bradley-Terry MLE (Reward Model 훈련)
def bt_loss(r_w, r_l):
    """L = -log sigmoid(r_w - r_l)"""
    return -F.logsigmoid(r_w - r_l).mean()

# Reward hacking 시각화
# 1. Reward model 훈련 (작은 데이터셋)
# 2. PPO로 그 RM에 맞춰 정책 최적화
# 3. 생성 길이, repetition, coherence 측정 → degradation 관찰
# 4. KL penalty β 증가로 완화
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "LLM Pretraining, PG, Advanced RL, Info 선행 필수"
   - RLHF → DPO → variants 진화의 수학적 축
   - InstructGPT, LLaMA-2, Claude의 alignment 사례 분석
3. **챕터별 문서 작성**: Preference → RLHF → DPO → DPO variants → CAI → 실전 이슈 → Safety

---

## 📚 참고 자료

- **Deep Reinforcement Learning from Human Preferences** (Christiano et al. 2017) — RLHF 효시
- **Training language models to follow instructions with human feedback** (Ouyang et al. 2022) — InstructGPT
- **Constitutional AI** (Bai et al. 2022)
- **Direct Preference Optimization** (Rafailov et al. 2023) — DPO
- **A General Theoretical Paradigm to Understand Learning from Human Preferences** (Azar et al. 2023) — IPO
- **KTO: Model Alignment as Prospect Theoretic Optimization** (Ethayarajh et al. 2024)
- **SimPO** (Meng et al. 2024)
- **ORPO** (Hong et al. 2024)
- **DeepSeek Math** (Shao et al. 2024) — GRPO
- **Scaling Laws for Reward Model Overoptimization** (Gao et al. 2023)
- **Let's Verify Step by Step** (Lightman et al. 2023) — PRM
- **Self-Rewarding Language Models** (Yuan et al. 2024)
- **Weak-to-Strong Generalization** (Burns et al. 2023)
- **LLaMA 2** (Touvron et al. 2023) — iterative RLHF
- **A Comprehensive Survey of LLM Alignment Techniques** (최신 survey)

---

## 💡 핵심 분석 대상

```
LLM Alignment의 지도

───── Preference Learning ─────

Data: {(x, y_w, y_l)} with y_w ≻ y_l

Bradley-Terry (1952):
  P(y_1 ≻ y_2) = σ(r(x, y_1) - r(x, y_2))
  Hidden utility r
  Pairwise comparison

Plackett-Luce (ranking):
  P(y_1 > y_2 > ... > y_k) = Π e^{r_i}/Σ_{j≥i} e^{r_j}

───── RLHF Pipeline ─────

Stage 1: SFT
  Pretrained → fine-tune on demos
  L = -E[log π(y|x)]
  Purpose: format 정렬

Stage 2: Reward Model
  Init from SFT + head
  L_RM = -E[log σ(r(x,y_w) - r(x,y_l))]
  Bradley-Terry MLE

Stage 3: PPO
  max_π E[r(x,y)] - β KL(π || π_SFT)
  
  왜 KL?
    Reward hacking 방지
    Distribution shift 억제
    Pretrained knowledge 유지

Optimal policy:
  π*(y|x) ∝ π_SFT(y|x) · exp(r(x,y)/β)

───── Reward Hacking ─────

RM = proxy of human preference
π가 RM을 exploit
→ spurious features
→ actual preference 저하

대책:
  ├── KL penalty (β 증가)
  ├── RM ensemble
  ├── Iterative RLHF (LLaMA-2)
  └── Process RM (step-level)

Gao 2023 scaling law:
  gold RM과 proxy RM 괴리가
  KL에 따라 커짐
  → optimal KL budget 존재

───── PRM vs ORM ─────

ORM (Outcome): final answer만 평가
PRM (Process): 각 reasoning step 평가

Lightman 2023:
  MATH benchmark에서 PRM >> ORM
  step-level feedback 유리
  reward model 훈련 어려움

───── DPO 유도 ─────

Step 1: RLHF optimal
  π*(y|x) = (1/Z) π_ref(y|x) exp(r/β)

Step 2: Invert for r
  log π*/π_ref = r/β - log Z
  r(x,y) = β log(π*/π_ref) + β log Z(x)

Step 3: Bradley-Terry with r
  r_w - r_l = β log(π*/π_ref)(y_w) 
            - β log(π*/π_ref)(y_l)
  (log Z는 x에 의존, y_w-y_l에서 소거)

Step 4: DPO loss
  L = -E[log σ(β log π/π_ref(y_w) 
              - β log π/π_ref(y_l))]

장점:
  RM 불필요
  안정적 (PPO보다)
  메모리 절약

한계:
  BT 가정
  Offline
  길이 편향

───── DPO 후계자 ─────

IPO (Azar 2023):
  BT 가정 완화
  L = E[(h - 1/(2β))²]
  deterministic pref에 robust

KTO (Ethayarajh 2024):
  Prospect theory
  desirable/undesirable (no pair)
  L = -E[λ v(β logr)]

SimPO (Meng 2024):
  No reference model
  Length-normalized
  L = -E[log σ((β/|y_w|)logπ(y_w) 
                - (β/|y_l|)logπ(y_l) - γ)]

ORPO (Hong 2024):
  SFT + pref 결합
  single stage
  Odds ratio

GRPO (DeepSeek 2024):
  Group 내 상대 순위
  No critic
  Reasoning에 특화

───── Constitutional AI ─────

SL-CAI:
  1. Model generates harmful response
  2. Self-critique (constitution-based)
  3. Revise response
  4. SFT on (prompt, revised)

RLAIF:
  AI-generated preferences
  scalability ↑
  bias 우려

Self-Rewarding (Yuan 2024):
  Model = policy + judge
  Iterative DPO
  자기 개선 loop

Scalable Oversight:
  Debate (Irving 2018)
  Amplification (Christiano 2018)
  Weak-to-strong (Burns 2023)

───── 실전 이슈 ─────

Length bias:
  RM/DPO가 긴 답변 선호
  spurious correlation

Overoptimization (Gao 2023):
  KL budget 초과하면
  gold performance 저하

Hyperparameters:
  β ∈ [0.01, 0.1]
  LR: SFT의 1/10~1/100

Distribution shift:
  PPO (on-policy): robust but slow
  DPO (off-policy): fast but brittle

Evaluation:
  MT-Bench, AlpacaEval
  Chatbot Arena Elo
  LLM-as-judge (GPT-4)
    positional bias
    length bias

───── Safety ─────

Red Teaming:
  Manual + automated
  Jailbreak techniques

Adversarial Training:
  Attack in training

Refusal:
  trade-off vs helpfulness
  over-refusal 문제

Interpretability:
  Anthropic circuits
  refusal direction (Arditi 2024)
  편집 가능

───── 레포 간 연결 ─────

LLM Pretraining (직전):
  Base model 전제

Policy Gradient (Layer 4-A):
  REINFORCE, log-trick

Advanced RL (Layer 4-A):
  PPO, KL-constrained

Bayesian ML (Layer 1):
  Bradley-Terry = logistic
  Preference ~ posterior

Information Theory (Layer 0):
  KL divergence
  Cross-entropy
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명 (3~4줄)
- 전체 문서 개수 확인 (34개 목표)
- Python + PyTorch + TRL 실험 환경
- LLM Pretraining, PG, Advanced RL, Info 레포 참조 관계
- Efficiency·Inference·Safety로 이어지는 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
