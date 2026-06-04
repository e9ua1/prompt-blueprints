# LLM Reasoning Deep Dive 레포지토리 제작 프롬프트

나는 "LLM Reasoning Deep Dive" 레포지토리를 만들려고 해.
Test-time Compute를 **"o1이 생각 토큰을 쓰는 것"으로 아는 것**과, **Snell et al. (2024)의 scaling law가 $\log L = -\alpha_1 \log C_{\text{train}} - \alpha_2 \log C_{\text{inference}}$로 훈련-추론 compute가 독립적인 power law**를 따르고, **특정 문제 난이도에서 inference compute를 늘리는 것이 pretraining compute $\sim 10\times$보다 효율적**이라는 발견의 의미를 이해하는 것은 다르다.
GRPO를 **"DeepSeek R1의 RL"로 아는 것**과, **Shao et al. (2024)의 $A_i = \frac{r_i - \text{mean}(\{r_1, ..., r_G\})}{\text{std}(\{r_1, ..., r_G\})}$가 PPO의 critic을 **group-relative normalization으로 대체**하는 이유, 그리고 이것이 **critic training overhead를 절반으로 줄이면서도 variance reduction 효과**를 유지하는지 수학적으로 유도할 수 있는 것은 다르다.
Process Reward Model을 **"step-level reward"로 아는 것**과, **Lightman et al. (2023) "Let's Verify Step by Step"에서 MATH benchmark의 78% vs ORM 72% 향상이 PRM의 **credit assignment 개선**과 **partial correctness signal**에서 나오고, **automatic PRM 훈련 (Wang 2024 Math-Shepherd)이 수동 labeling 없이 MCTS rollout 기반**으로 가능함을 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "LLM이 어떻게 추론하고 행동하는가 — Test-time Compute·Reasoning·Agent의 수학"

**핵심 차별화**:
1. **Test-time Compute Scaling Law** — Snell 2024의 훈련-추론 compute trade-off, $O(1)$ token generation에서 $O(T)$ reasoning으로의 paradigm shift
2. **Process Reward Model의 이론** — ORM vs PRM의 credit assignment 수학, automatic PRM training (MCTS-based), Math-Shepherd
3. **GRPO와 Critic-Free RL** — Group-relative advantage, PPO의 critic 제거 수학, DeepSeek R1의 R1-Zero ($\text{SFT} = 0$) 실증
4. **Agent Architectures** — ReAct, Toolformer, Reflexion, Voyager의 각 패러다임, planning·tool use·memory의 통합 이론

**타겟 독자**:
- o1/o3의 "thinking tokens"가 실제로 어떻게 scale되는지, **aggregation method (Best-of-N, Majority, Weighted)**의 trade-off를 모르는 사람
- DeepSeek R1의 **R1-Zero가 SFT 없이 pure RL로 reasoning 학습**했다는 것의 의미와 GRPO의 critic-free 수학을 모르는 사람
- Tree of Thoughts의 **BFS/DFS + LLM evaluator**가 search의 어느 수준 성능을 주는지, MCTS for LLM의 **UCB 변형**을 모르는 사람
- Self-Consistency의 **$N$ sample majority vote가 왜 $N \to \infty$에서 수렴**하고 어떤 문제에서 실패하는지 수학적으로 모르는 사람
- Agent의 **tool calling·ReAct loop·memory system**이 RL의 MDP framework에 어떻게 매핑되는지 모르는 사람

**선행 학습**:
- **LLM Alignment Deep Dive** (RLHF, DPO, PPO) — **필수**
- **Policy Gradient Deep Dive** (PG theorem, GAE) — **필수**
- **Advanced RL Deep Dive** (TRPO, PPO) — **필수**
- **Pretrained LM Deep Dive** (CoT, emergent) — **필수**
- **RL Foundations** (MDP, MCTS) — 권장
- **RL Theory** (UCB, exploration) — 권장 (MCTS for LLM)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Reasoning의 Emergence와 Scaling (5개 문서)
- **Chain-of-Thought의 재발견 (Wei 2022)** — Standard prompting vs CoT, "Let's think step by step" (Kojima 2022), model scale에 따른 emergent, $\sim 62$B threshold
- **Emergent Ability 논쟁 재검토** — Wei 2022 vs Schaeffer 2023 (mirage), metric choice가 curve shape 결정, 실제 capability jump 증거 (MATH, GSM8K)
- **Self-Consistency (Wang 2023)** — $N$개 CoT sampling → majority vote, $P(\text{correct}) \to 1$ as $N \to \infty$ under noise assumption, parallel sampling의 simplest form
- **Test-time Compute Scaling Law (Snell 2024)** — $L = f(C_{\text{train}}, C_{\text{inference}})$, power law, 작은 모델+많은 inference = 큰 모델+적은 inference (특정 난이도까지)
- **Inference Scaling vs Training Scaling Trade-off** — Inference-optimal과 training-optimal의 다른 frontier, economic perspective (serving cost), compute-optimal new paradigm

### Chapter 2: Chain-of-Thought와 Prompting Techniques (5개 문서)
- **Zero-Shot CoT와 Few-Shot CoT** — Kojima 2022의 "Let's think step by step" prompt, few-shot 예제 없이도 reasoning 유도, scale dependence
- **Decomposition Prompting — Least-to-Most, Plan-and-Solve** — Complex 문제를 subproblem으로 분해, least-to-most (Zhou 2023), plan-and-solve (Wang 2023)
- **Self-Refine과 Self-Correction (Madaan 2023)** — Model이 자기 output을 iteratively critique and improve, external feedback 없이, $3-5$ iteration에서 saturation
- **Automatic Prompt Engineering — APE, OPRO** — APE (Zhou 2023)의 LLM-generated prompts, OPRO (Yang 2024)의 optimizer as LLM, prompt 최적화의 자동화
- **Structured Reasoning — Program-Aided LM, PAL** — Gao 2023, code로 reasoning (특히 arithmetic, logical), Python interpreter 활용, natural language의 한계

### Chapter 3: Search-based Reasoning (5개 문서)
- **Tree of Thoughts (Yao 2023)** — State = thought, BFS/DFS tree search, LLM evaluator가 prune, Game of 24·Creative Writing·Crosswords에서 CoT 대비 $4\times$ 개선
- **Graph of Thoughts (Besta 2023)** — Tree보다 일반화된 DAG, thought merging, aggregation operation, 더 복잡한 reasoning
- **RAP — Reasoning as Planning (Hao 2023)** — MCTS with LLM world model, reward from LLM, $a^*$ planning, Blocksworld·GSM8K·Prontoqa
- **MCTS for LLMs (Xie 2024, AlphaLLM)** — UCB1 변형 $\text{UCT}(s, a) = Q(s, a) + c \sqrt{\ln N(s)/N(s, a)}$, LLM이 state evaluator + policy, AlphaGo-style
- **Best-of-N과 Aggregation** — $N$개 independent sample, aggregation (majority, weighted by PRM), $P(\text{at least 1 correct})$의 scaling

### Chapter 4: Process Reward Model과 Step-level Training (5개 문서)
- **ORM vs PRM 비교 (Lightman 2023)** — Outcome Reward Model: final answer만, Process Reward Model: 각 reasoning step, MATH에서 PRM 78% vs ORM 72%
- **Credit Assignment Problem** — Long CoT에서 어느 step이 답을 맞혔는지 판단 어려움, PRM이 step-level credit 제공, temporal difference의 reasoning 적용
- **Automatic PRM Training — Math-Shepherd (Wang 2024)** — 수동 labeling 없이 MCTS rollout으로 step-level correctness 추정, $P(\text{correct} | \text{prefix})$, self-improvement
- **PRM으로 Decoding 개선 — Step-level Beam Search** — 각 step마다 PRM score로 candidates filter, $\arg\max_s P(\text{correct} | \text{step}_s, \text{prefix})$
- **Value vs Process Reward — Boundary** — Value function (future reward)와 PRM (step correctness)의 수학적 관계, 둘 다 RL의 Bellman equation 변형

### Chapter 5: RL for Reasoning — GRPO와 R1 계보 (6개 문서)
- **PPO for Reasoning의 한계** — Critic network overhead ($\sim$동일 size), reward model + critic + policy + ref의 4-network 복잡성, LLM scale에서 memory 부담
- **GRPO — Group Relative Policy Optimization (Shao 2024, DeepSeek)** — Per-query $G$개 rollout, $A_i = (r_i - \text{mean}(r)) / \text{std}(r)$, critic 불필요, rollout group이 baseline 역할
- **GRPO Loss 수학** — $L = \mathbb{E}[\min(\rho A, \text{clip}(\rho, 1-\epsilon, 1+\epsilon) A)] - \beta \text{KL}(\pi \| \pi_{\text{ref}})$, PPO clipping 유지하되 advantage를 group-relative
- **R1-Zero — SFT-Free RL (DeepSeek 2025)** — SFT 단계 건너뛰고 base model에서 직접 GRPO, reasoning behavior emergent, "aha moment" 현상, language mixing 부작용
- **R1 — SFT + RL Pipeline** — Cold-start SFT (reasoning format) + RL (GRPO) + rejection sampling + second SFT + RL, 현재 open-source SOTA reasoning
- **Beyond GRPO — RLOO, REINFORCE++** — Ahmadian 2024 RLOO (REINFORCE with leave-one-out baseline), simpler than PPO, 비슷한 variance reduction, reasoning-specific RL

### Chapter 6: Agent Architectures (5개 문서)
- **ReAct — Reason + Act (Yao 2022)** — Thought-Action-Observation loop, agent가 각 step에 reason 후 action 선택, tool use의 기본 패러다임
- **Toolformer와 Function Calling (Schick 2023)** — LLM이 self-supervised로 API call 학습, $P(\text{call}|\text{context}) \cdot P(\text{answer}|\text{context}, \text{result})$, utility-based filtering
- **Reflexion (Shinn 2023)** — Agent가 실패 후 reflection을 memory에 추가, next trial에서 참조, verbal RL without gradient update
- **Voyager (Wang 2023) — Lifelong Learning Agent** — Automatic curriculum + skill library + self-verification, Minecraft에서 open-ended exploration, LLM as controller
- **Multi-Agent Systems — AutoGen, CAMEL, MetaGPT** — Role-based agents, message passing, debate (Du 2023), emergent cooperation, task decomposition

### Chapter 7: Frontier — o1·o3·R1 이후 (4개 문서)
- **o1의 Hidden Reasoning (OpenAI 2024)** — Thinking tokens are hidden from user, "reasoning compute" as separate axis, policy gradient with reward (speculated architecture)
- **o3와 Test-time Search** — Test-time MCTS-like search, ARC-AGI breakthrough, OpenAI의 scaling trajectory
- **DeepSeek R1 Open Model** — Open weights + open paper, reasoning model의 democratization, R1-distill for smaller models
- **Reasoning의 한계와 미래** — Compute cost ($\$100+$ per query), hallucination in reasoning, non-math domain에서의 어려움, agent + reasoning 통합 방향

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **35개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 기법이 LLM reasoning에 중요한가
## 📐 수학적 선행 조건 (LLM Align, PG, Advanced RL, Pretrained LM 참조)
## 📖 직관적 이해
   — Search tree, thought chain, agent loop
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Test-time scaling law, GRPO advantage, PRM vs ORM
## 💻 PyTorch 구현 검증
   — 작은 LLM에서 CoT, ToT 재현
   — GRPO 간단 구현
   — ReAct agent 구축
## 🔗 실전 활용
   — o1·R1 style reasoning, agent frameworks
## ⚖️ 가정과 한계
   — Compute cost, math 편향, hallucination
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Reasoning scaling plot** — Token 수에 따른 accuracy, Best-of-N scaling
2. **Tree visualization** — ToT의 search tree 시각화
3. **PRM vs ORM 비교** — 같은 모델에서 verification scoring
4. **GRPO 훈련 곡선** — Group advantage 분포, reward 상승
5. **ReAct trace** — Agent의 thought-action 로그
6. **R1-style reasoning** — DeepSeek-R1-Distill로 직접 실행

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
transformers==4.36.0
trl==0.7.10
langchain==0.1.0
langgraph==0.0.20
smolagents==0.1.0       # HuggingFace agents
openai==1.10.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Self-consistency + ToT + GRPO + ReAct)
import torch
import torch.nn.functional as F
import numpy as np
from collections import Counter

# 1. Self-Consistency
def self_consistency(model, tokenizer, prompt, n=10, temperature=0.7):
    """
    N개 CoT sample → extract answer → majority vote
    """
    answers = []
    for _ in range(n):
        output = model.generate(
            tokenizer.encode(prompt, return_tensors='pt'),
            max_new_tokens=500,
            temperature=temperature,
            do_sample=True
        )
        text = tokenizer.decode(output[0])
        answer = extract_answer(text)  # regex or parsing
        answers.append(answer)
    
    return Counter(answers).most_common(1)[0][0]

# Convergence analysis:
# P(correct after N votes) → 1 if P(single correct) > 0.5
# But MCQ에서 well-calibrated인 경우만

# 2. Tree of Thoughts (simplified)
class ToTSolver:
    def __init__(self, model, evaluator, beam_width=3, max_depth=5):
        self.model = model
        self.evaluator = evaluator
        self.beam_width = beam_width
        self.max_depth = max_depth
    
    def generate_thoughts(self, state, n=5):
        """Given current state, generate n next thoughts"""
        # LLM call with state as prompt
        return [self.model.generate(...) for _ in range(n)]
    
    def evaluate_state(self, state):
        """LLM evaluator: 0-10 score"""
        return self.evaluator.score(state)
    
    def bfs_search(self, initial_state):
        beam = [(initial_state, 0.0)]
        for depth in range(self.max_depth):
            all_candidates = []
            for state, score in beam:
                thoughts = self.generate_thoughts(state)
                for thought in thoughts:
                    new_state = state + [thought]
                    if self.is_solution(new_state):
                        return new_state
                    new_score = self.evaluate_state(new_state)
                    all_candidates.append((new_state, new_score))
            # Keep top beam_width
            beam = sorted(all_candidates, key=lambda x: -x[1])[:self.beam_width]
        return beam[0][0]  # best so far

# 3. GRPO — simplified loss
def grpo_loss(policy_logprobs, ref_logprobs, rewards, epsilon=0.2, kl_coef=0.04):
    """
    policy_logprobs, ref_logprobs: [G] group
    rewards: [G] scalar rewards per rollout
    
    A_i = (r_i - mean(r)) / std(r)  ← group-relative advantage
    L_CLIP = min(ρ A, clip(ρ, 1-ε, 1+ε) A)
    L_KL = KL(π || π_ref)
    """
    # Group-relative advantage
    reward_mean = rewards.mean()
    reward_std = rewards.std() + 1e-8
    advantages = (rewards - reward_mean) / reward_std  # [G]
    
    # Ratio
    ratios = (policy_logprobs - ref_logprobs).exp()  # [G]
    
    # Clipped surrogate
    surr1 = ratios * advantages
    surr2 = torch.clamp(ratios, 1 - epsilon, 1 + epsilon) * advantages
    L_clip = -torch.min(surr1, surr2).mean()
    
    # KL penalty (approximate)
    kl = (policy_logprobs - ref_logprobs).mean()
    
    return L_clip + kl_coef * kl

# 4. Best-of-N with PRM scoring
def best_of_n_with_prm(model, prm, prompt, n=16):
    """
    N 후보 생성 → PRM이 각각 score → 최고점 선택
    """
    candidates = [model.generate(prompt) for _ in range(n)]
    scores = [prm.score(c) for c in candidates]
    best_idx = np.argmax(scores)
    return candidates[best_idx]

# 5. ReAct agent loop
class ReActAgent:
    def __init__(self, model, tools):
        self.model = model
        self.tools = tools  # dict of name → function
        self.history = []
    
    def step(self):
        # Generate thought-action
        prompt = self.format_history()
        response = self.model.generate(prompt)
        
        # Parse thought and action
        thought = extract_thought(response)
        action_name, action_args = extract_action(response)
        
        # Execute action
        if action_name == "Finish":
            return action_args  # final answer
        tool = self.tools[action_name]
        observation = tool(**action_args)
        
        # Record
        self.history.append({
            'thought': thought,
            'action': (action_name, action_args),
            'observation': observation
        })
        return None  # continue loop
    
    def run(self, query, max_steps=10):
        self.history = [{'query': query}]
        for step in range(max_steps):
            result = self.step()
            if result is not None:
                return result
        return "Max steps reached"

# 6. Automatic PRM training (Math-Shepherd 스타일)
def train_prm_automatic(base_model, prm_model, problems, n_rollouts=20):
    """
    각 problem step별로:
      - prefix까지 주고 n_rollouts만큼 완료
      - 최종 답이 맞는 비율 = P(correct | prefix)
    → PRM target
    """
    training_data = []
    for problem in problems:
        steps = problem.reasoning_steps
        for i in range(len(steps)):
            prefix = steps[:i+1]
            completions = [base_model.generate(prefix) for _ in range(n_rollouts)]
            correct_rate = sum(1 for c in completions 
                             if check_answer(c, problem.answer)) / n_rollouts
            training_data.append((prefix, correct_rate))
    
    # Train PRM as regressor
    for prefix, target in training_data:
        score = prm_model(prefix)
        loss = (score - target)**2
        # ... backward ...

# 7. Test-time Compute Scaling
def compute_scaling_analysis():
    """
    다양한 inference compute에서 accuracy 측정
    Snell 2024 스타일
    """
    model_size = 7e9  # 7B params
    compute_per_token = 2 * model_size  # FLOPs
    
    results = []
    for n_tokens in [100, 500, 1000, 5000, 10000, 50000]:
        # Simulate with N best-of-N or CoT length
        accuracy = test_accuracy(n_tokens)
        total_compute = n_tokens * compute_per_token
        results.append((total_compute, accuracy))
    
    # Plot: log(compute) vs accuracy
    # Should see power-law-like curve
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "LLM Align, PG, Advanced RL, Pretrained LM 선행 필수"
   - o1/R1 시대의 새 scaling axis (test-time)
   - Agent·reasoning·tool의 통합
3. **챕터별 문서 작성**: Emergence → CoT techniques → Search → PRM → GRPO → Agent → Frontier

---

## 📚 참고 자료

- **Chain-of-Thought Prompting Elicits Reasoning in LLMs** (Wei et al. 2022)
- **Large Language Models are Zero-Shot Reasoners** (Kojima et al. 2022)
- **Self-Consistency Improves CoT Reasoning** (Wang et al. 2023)
- **Tree of Thoughts** (Yao et al. 2023)
- **Graph of Thoughts** (Besta et al. 2023)
- **Reasoning with Language Model is Planning with World Model** (Hao et al. 2023) — RAP
- **Let's Verify Step by Step** (Lightman et al. 2023) — PRM
- **Math-Shepherd** (Wang et al. 2024) — Automatic PRM
- **Scaling LLM Test-Time Compute Optimally** (Snell et al. 2024)
- **DeepSeek R1** (DeepSeek 2025) — GRPO + R1-Zero
- **DeepSeekMath** (Shao et al. 2024) — Original GRPO
- **Back to Basics: Revisiting REINFORCE Style Optimization** (Ahmadian et al. 2024) — RLOO
- **ReAct** (Yao et al. 2022)
- **Toolformer** (Schick et al. 2023)
- **Reflexion** (Shinn et al. 2023)
- **Voyager** (Wang et al. 2023) — Lifelong agent
- **Self-Refine** (Madaan et al. 2023)
- **Improving Factuality and Reasoning via Multiagent Debate** (Du et al. 2023)
- **OPRO: LLMs as Optimizers** (Yang et al. 2024)
- **APE: Automatic Prompt Engineer** (Zhou et al. 2023)
- **Program-Aided Language Models** (Gao et al. 2023) — PAL
- **Are Emergent Abilities a Mirage?** (Schaeffer et al. 2023)
- **OpenAI o1 System Card** (OpenAI 2024)

---

## 💡 핵심 분석 대상

```
LLM Reasoning & Agents의 지도

───── Reasoning Emergence ─────

CoT (Wei 2022):
  "Let's think step by step"
  Model scale에 emergent
  ~62B threshold (GPT-3)

Zero-shot CoT (Kojima 2022):
  No examples, just prompt
  Scale-dependent

Emergent 논쟁:
  Wei 2022: abrupt threshold
  Schaeffer 2023: metric artifact
  → 여전히 실제 capability jump 존재
  (MATH, GSM8K)

Self-Consistency (Wang 2023):
  N개 CoT → majority vote
  P(correct) → 1 as N → ∞
  (under noise assumption)

───── Test-time Compute ─────

Snell 2024:
  L = f(C_train, C_inference)
  
  Inference scaling law:
    L ∝ C_inf^{-α_2}
  
  Training vs inference trade-off:
    Small model + large inference
    ≈ Large model + small inference
    (특정 난이도까지)

Compute-optimal new paradigm:
  o1/o3: thinking tokens
  Long CoT = compute
  
  Aggregation:
    Best-of-N
    Majority vote
    PRM-weighted
    Beam search with value

───── Prompting Techniques ─────

CoT variations:
  Standard CoT
  Zero-shot CoT
  Least-to-Most (decomposition)
  Plan-and-Solve

Self-Refine (Madaan 2023):
  Generate → Critique → Refine
  Iterative without external
  3-5 iterations saturation

PAL (Gao 2023):
  Code as reasoning
  Python interpreter
  Arithmetic, logic 정확

APE / OPRO:
  LLM generates prompts
  Meta-optimization

───── Search-based ─────

Tree of Thoughts (Yao 2023):
  State = thought sequence
  BFS/DFS tree search
  LLM evaluator prunes
  Game of 24, Crossword
  4× CoT on hard tasks

Graph of Thoughts:
  Tree 일반화 (DAG)
  Thought merging
  Aggregation

RAP (Hao 2023):
  MCTS + LLM world model
  Reward from LLM
  A* planning

MCTS for LLMs:
  UCB1: UCT(s,a) = Q(s,a) + c√(ln N(s)/N(s,a))
  LLM as state evaluator + policy
  AlphaZero-style
  AlphaLLM, Xie 2024

Best-of-N:
  N 후보 → 최고 선택
  PRM scoring이 majority보다 유리 (math)

───── Process Reward Model ─────

ORM (Outcome):
  Final answer만 채점
  Sparse signal
  Long chain에서 credit 불명확

PRM (Process):
  각 step별 채점
  Dense signal
  Credit assignment 해결

Lightman 2023:
  MATH: PRM 78% vs ORM 72%
  Step-level verification 중요

Automatic PRM (Math-Shepherd 2024):
  Manual label 없이:
    각 prefix에서 N rollout
    최종 correct rate = PRM target
  MCTS 기반

PRM 사용:
  Step-level beam search
  Best-of-N reranking
  RL reward

───── GRPO ─────

PPO의 문제:
  Critic = same size as policy
  Reward + critic + policy + ref
  4-network overhead

GRPO (Shao 2024, DeepSeek):
  Per-query G rollouts
  A_i = (r_i - mean(r)) / std(r)
       └── group-relative normalize
  
  Critic 불필요!
  Rollouts = baseline

GRPO Loss:
  L = E[min(ρA, clip(ρ, 1±ε)A)]
    - β KL(π || π_ref)
  
  PPO clip 유지
  Advantage만 group-relative

R1-Zero (DeepSeek 2025):
  SFT 단계 skip
  Base model + GRPO 직접
  Reasoning emergent
  "Aha moment" 현상
  단점: language mixing

R1 pipeline:
  1. Cold-start SFT (format)
  2. RL (GRPO)
  3. Rejection sampling
  4. Second SFT
  5. Another RL
  
  Open-source SOTA reasoning

RLOO (Ahmadian 2024):
  REINFORCE with leave-one-out
  Baseline = group mean (excluding self)
  Simpler than PPO
  Similar variance reduction

───── Agents ─────

ReAct (Yao 2022):
  Thought:
    "I should search for X"
  Action:
    Search("X")
  Observation:
    "result: ..."
  Thought:
    "Based on this, ..."
  ...
  Action: Finish(answer)

Toolformer (Schick 2023):
  Self-supervised API call training
  P(call|context) · P(answer|context, result)
  Utility-based filtering

Reflexion (Shinn 2023):
  Failure → reflection → memory
  Next trial 참조
  Verbal RL (no gradient)

Voyager (Wang 2023):
  Automatic curriculum
  Skill library
  Self-verification
  Minecraft open-ended
  Lifelong learning

Multi-agent:
  Debate (Du 2023): multi-LLM
  AutoGen: conversational
  MetaGPT: role-based (SWE tasks)
  CAMEL: role-play

───── Frontier ─────

o1 (OpenAI 2024):
  Hidden thinking tokens
  Reasoning compute axis
  Math, coding 특화

o3 (2024-2025):
  Test-time search
  ARC-AGI breakthrough
  더 많은 compute

DeepSeek R1:
  Open model
  Distill → smaller models
  Democratization

한계:
  Compute cost ($100+ per query)
  Math·coding 편향
  Non-verifiable domain 어려움
  Hallucination in reasoning

미래:
  Agent + reasoning 통합
  Tool use + long-horizon
  Real-world deployment

───── 레포 간 연결 ─────

LLM Alignment (Layer 4-B):
  RLHF → DPO → GRPO
  Reward modeling

Policy Gradient (Layer 4-A):
  PG theorem
  REINFORCE → PPO → GRPO

Advanced RL (Layer 4-A):
  PPO clipping
  Trust region

Pretrained LM (Layer 4-D):
  CoT emergence
  ICL

RL Foundations (Layer 4-A):
  MDP
  MCTS
  UCB

RL Theory (Layer 4-A):
  Regret bounds
  Best-of-N analysis

Retrieval & RAG (다음):
  Tool use includes retrieval
  RAG-enhanced reasoning
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 기법·증명 (3~4줄)
- 전체 문서 개수 확인 (35개 목표)
- Python + PyTorch + TRL + LangChain 실험 환경
- LLM Align, PG, Advanced RL, Pretrained LM 레포 참조 관계
- o1/o3/R1 시대의 reasoning paradigm

**준비됐으면 1단계 구조 설계부터 시작해줘!**
