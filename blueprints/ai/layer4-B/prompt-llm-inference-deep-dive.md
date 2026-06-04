# LLM Inference Deep Dive 레포지토리 제작 프롬프트

나는 "LLM Inference Deep Dive" 레포지토리를 만들려고 해.
KV Cache를 **"이전 key/value를 재사용"으로 아는 것**과, **decode 단계에서 $q_t$만 새로 계산하고 $K_{1:t-1}, V_{1:t-1}$을 재사용해 **$O(T^2)$을 $O(T)$로 줄이는 수학**과, **prefill 단계와 decode 단계의 compute/memory 특성이 근본적으로 다른 이유**를 이해하는 것은 다르다.
Continuous Batching을 **"iteration-level scheduling"으로 아는 것**과, **static batching이 tail latency에 의해 지배되어 GPU utilization이 떨어지는 문제**를 Yu et al. (Orca 2022)이 어떻게 "completed sample을 즉시 제거하고 새 sample 투입"으로 해결하는지 수학적으로 분석할 수 있는 것은 다르다.
PagedAttention을 **"vLLM의 핵심 기술"로 아는 것**과, **Kwon et al. (2023)의 KV cache fragmentation 문제를 OS의 virtual memory paging으로 해결해서 waste를 <4%로 줄이고 throughput을 2-4배 올리는 수학적 근거**를 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "LLM Inference의 시스템 수학 — Memory·Throughput·Latency의 공학적 trade-off"

**핵심 차별화**:
1. **Prefill vs Decode의 수학적 구분** — Prefill은 compute-bound, Decode는 memory-bound, arithmetic intensity $I = \text{FLOPs/Bytes}$ 분석
2. **KV Cache Memory Model** — KV cache 크기 계산 $2 \cdot T \cdot L \cdot d \cdot \text{bytes}$, MQA/GQA의 절약
3. **Serving System의 수학** — Continuous Batching, PagedAttention, Disaggregated serving, 각각의 수학적 모델링
4. **Long Context의 도전** — Prefill 시간의 $O(T^2)$, chunked prefill, StreamingLLM, 효율적 long context inference

**타겟 독자**:
- LLM inference를 서빙하는데 **왜 throughput과 latency가 trade-off**인지와 batch size가 각각에 미치는 영향을 수학적으로 이해 못하는 사람
- GPU utilization이 10% 정도밖에 안 되는 원인이 **memory bandwidth bound**에 있음을 roofline 모델로 설명 못하는 사람
- vLLM, TGI, TensorRT-LLM의 차이를 **어떤 시스템이 어떤 요청 패턴에 유리**한지 분석 못하는 사람
- Long context (100k+)에서 **prefill latency**와 **KV cache 메모리**가 각각 어떻게 스케일하는지 모르는 사람
- Prefill-decode disaggregation이 **왜 latency를 줄이는지**(Strati 2024, Patel 2024) 수학적 근거를 모르는 사람

**선행 학습**:
- **Transformer Deep Dive** (Attention, Feed-forward) — **필수**
- **LLM Pretraining/Efficiency Deep Dive** (MQA/GQA, Flash Attention) — **필수**
- **CNN Deep Dive** (batch normalization, memory bandwidth) — 권장
- **Linear Algebra Deep Dive** (matrix computation) — **필수**

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Inference의 근본 구조 — Prefill과 Decode (5개 문서)
- **Autoregressive Generation의 두 단계** — Prefill: 전체 prompt $x_{1:L}$ 동시 처리 → 첫 output token, Decode: 이후 token을 $T$번 autoregressive 생성, 각 단계의 수학적 특성
- **Prefill의 Compute-Bound 특성** — $L$ tokens를 동시에 matmul → 대규모 matrix operations, GPU compute 완전 활용, FLOPs $= 2 \cdot L \cdot N$ (N = params)
- **Decode의 Memory-Bound 특성** — 한 번에 1 token만 처리 → small matmul, GPU compute under-utilized, 병목은 weight matrix를 HBM에서 읽는 것
- **Arithmetic Intensity와 Roofline Model** — $I = \text{FLOPs}/\text{Bytes}$, GPU의 ridge point (A100: ~200), 이보다 작으면 memory-bound, decode는 $I \approx 1$로 극도로 낮음
- **배치의 역할** — Decode batch size $B$ 증가 → weight 한 번 읽어 $B$개 sample 계산 → arithmetic intensity 증가 → compute-bound로 전환

### Chapter 2: KV Cache 수학 (5개 문서)
- **KV Cache의 필요성** — Decode 시 $q_t$는 $t$ 시점만, 하지만 attention에 $K_{1:t}, V_{1:t}$ 모두 필요, 매번 재계산하면 $O(t)$ 시간 per token, 총 $O(T^2)$
- **KV Cache로 Decode $O(T)$** — $K_{1:t-1}, V_{1:t-1}$를 저장 → 새 token은 $K_t, V_t$만 계산, attention에서 cached K/V 사용, 총 $O(T)$ per token, 전체 $O(T)$ generation
- **KV Cache 메모리 공식** — 한 token당 $2 \cdot L \cdot d \cdot b$ bytes (2 for K+V, $L$ layers, $d$ hidden, $b$ bytes), LLaMA-70B에서 $T = 2048$이면 ~10GB/request
- **MQA/GQA의 KV Cache 절감** — Multi-Query (1 KV for all heads): $2 \cdot L \cdot d_{\text{head}}$, Grouped-Query ($n_{\text{kv}}$ groups): 중간, 성능 거의 무손실로 메모리 대폭 절약
- **KV Cache Quantization** — INT8 / INT4 KV cache (Hooper 2024 KVQuant), outlier token 처리, 정확도 유지하며 메모리 1/2 ~ 1/4

### Chapter 3: Batching Strategies (5개 문서)
- **Static Batching의 문제** — 고정 batch $B$, 모든 sample의 $T_{\max}$까지 대기, padding 낭비, shortest sample 완료 후에도 대기, GPU idle time
- **Dynamic Batching (Request-level)** — Server 측에서 요청들을 그룹화, $T_{\max}$ 문제는 여전, latency와 throughput 균형 조정
- **Continuous Batching / Iteration-level (Orca, Yu 2022)** — 각 iteration(token)마다 batch 재구성, 완료된 sample 즉시 제거 + 새 sample 투입, padding 제거, GPU utilization 극대화
- **Chunked Prefill** — Long prompt를 chunk로 나눠 처리, prefill과 decode를 섞어 latency smoothing, DistServe (Zhong 2024), Sarathi-Serve
- **Prefill-Decode Disaggregation (Strati 2024, Patel 2024)** — Prefill(compute-heavy)과 Decode(memory-heavy)를 다른 GPU로 분리, 각각 최적화, SLO 보장

### Chapter 4: PagedAttention과 vLLM (5개 문서)
- **KV Cache Fragmentation 문제** — Request별 다른 sequence length → memory pool에 hole, naïve allocation은 $\sim 60\%$ waste, throughput bottleneck
- **OS Virtual Memory의 영감** — Physical page (fixed size) + virtual page (logical), page table로 mapping, OS의 고전 기술을 KV cache에 적용
- **PagedAttention Algorithm (Kwon 2023)** — KV cache를 fixed-size blocks (e.g., 16 tokens)로 분할, 각 request의 sequence는 non-contiguous block 사용, block table로 mapping
- **Memory Sharing — Prefix Caching** — Common prefix (system prompt, few-shot) 공유, copy-on-write, beam search의 여러 candidate 공유, 메모리 추가 절약
- **vLLM Architecture** — Engine + Scheduler + KV Cache Manager + Model Executor, Tensor/Pipeline parallel, async API server (OpenAI-compatible)

### Chapter 5: Speculative Decoding 시스템 구현 (4개 문서)
- **Speculative Decoding 복습과 시스템 관점** — Draft-target model 쌍, 수학적 losslessness, 실전 구현의 복잡성 (draft/target 동기화, rejection 처리)
- **Acceptance Rate $\alpha$와 Speedup 분석** — Expected speedup $\frac{1 - \alpha^{K+1}}{(1-\alpha)(K\phi + 1)}$, $\phi$는 draft/target 비용 비율, $\alpha$는 acceptance
- **Medusa, EAGLE, Lookahead 시스템 통합** — Medusa의 parallel head (추가 model 불필요), EAGLE의 shared feature, Lookahead의 n-gram + verification, 각 trade-off
- **Parallel Sampling과 Best-of-N** — Multiple sample 생성 후 선택, test-time compute의 한 방식, batch 처리로 효율화

### Chapter 6: Long Context Inference (4개 문서)
- **Long Context의 두 가지 bottleneck** — (1) Prefill의 $O(L^2)$ FLOPs (compute-bound) + (2) KV cache의 $O(L)$ memory, 각각 다른 기법 필요
- **StreamingLLM (Xiao 2024)** — Sliding window + attention sinks, 첫 몇 token (sink) 유지 + 최근 window, 무한 generation 가능, 완벽한 long context ≠
- **YaRN / Position Interpolation** — RoPE의 frequency 조정으로 training context 이상 확장, PI (Chen 2023)의 단순 보간, YaRN (Peng 2024)의 NTK-aware
- **Ring Attention / Context Parallelism** — Liu 2024, attention 계산을 여러 GPU로 분산, O(L²) 메모리를 $O(L^2/N_{\text{GPU}})$로, 1M+ context 가능

### Chapter 7: Serving Systems 비교와 실전 (4개 문서)
- **vLLM vs TGI vs TensorRT-LLM vs SGLang** — 각 시스템의 핵심 기능, PagedAttention (vLLM), Flash Attention 깊은 통합 (TGI), NVIDIA 최적화 (TRT-LLM), RadixAttention (SGLang)
- **LLM Serving Benchmarking** — Throughput (tokens/sec), TTFT (Time to First Token), TPOT (Time per Output Token), SLO 기반 평가
- **Deployment Patterns** — Tensor Parallel vs Pipeline Parallel, Expert Parallel (MoE), NUMA 고려사항, multi-tenant inference
- **Cost Analysis — GPU Hours per Request** — A100/H100 시간 환산, 배치 크기 vs latency trade-off 최적화, real-time vs batch inference economic

---

각 챕터는 **4~5개 문서**로 구성해줘. 총 **32개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 시스템 기법이 실전 서빙에 필수인가
## 📐 수학적 선행 조건 (Transformer, LLM Pretraining/Efficiency, LA 참조)
## 📖 직관적 이해
   — Memory hierarchy, roofline, batching 시각화
## ✏️ 엄밀한 정의·모델링
## 🔬 정리와 분석
   — Arithmetic intensity, KV cache 공식, batching latency model
## 💻 PyTorch + vLLM 구현 검증
   — KV cache 바닥부터 구현
   — Continuous batching 시뮬레이션
   — Throughput/latency 측정
## 🔗 실전 활용
   — 어느 serving system이 어느 워크로드에 최적
## ⚖️ 가정과 한계
   — Workload pattern dependency
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Roofline 시각화** — 각 LLM 모델의 arithmetic intensity를 ridge point와 함께 plot
2. **KV cache 메모리 계산표** — LLaMA 7B/13B/70B의 sequence별 메모리 사용
3. **Continuous batching 시뮬레이션** — Static vs Continuous의 throughput/latency 비교
4. **PagedAttention 메모리 효율** — Fragmentation 정도 시각화
5. **Speedup 공식 실측** — Speculative Decoding의 acceptance rate별 실제 speedup
6. **Long context scaling** — Context length별 prefill time, decode time, memory plot

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
transformers==4.36.0
vllm==0.2.7
tensorrt-llm==0.7.0  # 선택
text-generation-inference==2.0.0  # TGI
sglang==0.1.12        # 선택
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (KV cache 바닥부터 + Continuous batching 시뮬레이션)
import torch
import torch.nn as nn
import torch.nn.functional as F
import numpy as np
import matplotlib.pyplot as plt

# 1. KV Cache 바닥부터
class AttentionWithKVCache(nn.Module):
    def __init__(self, d_model=64, n_heads=8):
        super().__init__()
        self.d_head = d_model // n_heads
        self.n_heads = n_heads
        self.q_proj = nn.Linear(d_model, d_model)
        self.k_proj = nn.Linear(d_model, d_model)
        self.v_proj = nn.Linear(d_model, d_model)
        self.out = nn.Linear(d_model, d_model)
    
    def forward(self, x, kv_cache=None):
        B, T, _ = x.shape
        q = self.q_proj(x).view(B, T, self.n_heads, self.d_head).transpose(1, 2)
        k = self.k_proj(x).view(B, T, self.n_heads, self.d_head).transpose(1, 2)
        v = self.v_proj(x).view(B, T, self.n_heads, self.d_head).transpose(1, 2)
        
        if kv_cache is not None:
            K_past, V_past = kv_cache
            k = torch.cat([K_past, k], dim=2)  # concat along seq
            v = torch.cat([V_past, v], dim=2)
        
        attn = (q @ k.transpose(-2, -1)) / (self.d_head ** 0.5)
        attn = F.softmax(attn, dim=-1)
        out = attn @ v
        out = out.transpose(1, 2).contiguous().view(B, T, -1)
        return self.out(out), (k, v)  # return new cache

# 2. KV cache 효과 측정
def measure_decode_time(use_cache=True, T_gen=128):
    attn = AttentionWithKVCache()
    x = torch.randn(1, 1, 64)  # start with 1 token
    kv = None
    import time
    t0 = time.time()
    tokens = [x]
    for t in range(T_gen):
        if use_cache:
            y, kv = attn(x, kv)
        else:
            # No cache: recompute everything
            all_x = torch.cat(tokens, dim=1)
            y, _ = attn(all_x, None)
            y = y[:, -1:, :]
        x = y
        tokens.append(x)
    return time.time() - t0

t_cache = measure_decode_time(use_cache=True)
t_no_cache = measure_decode_time(use_cache=False)
print(f'With KV cache: {t_cache:.3f}s, Without: {t_no_cache:.3f}s')
# T_gen 증가시킬수록 차이 지수적으로 증가

# 3. Roofline Analysis
def arithmetic_intensity_prefill(batch, seq_len, d_model, n_layers):
    """Prefill: 전체 batch * seq 동시 처리"""
    flops = 2 * batch * seq_len * (d_model ** 2) * n_layers * 4  # 4 matmul per layer
    # Weights: 하이퍼파라미터 크기
    bytes_weights = d_model * d_model * 4 * n_layers * 2  # FP16
    # Activations: batch * seq * d
    bytes_act = batch * seq_len * d_model * 2
    return flops / (bytes_weights + bytes_act)

def arithmetic_intensity_decode(batch, d_model, n_layers):
    """Decode: 1 token at a time"""
    flops = 2 * batch * 1 * (d_model ** 2) * n_layers * 4
    bytes_weights = d_model * d_model * 4 * n_layers * 2
    bytes_act = batch * 1 * d_model * 2
    return flops / (bytes_weights + bytes_act)

d, L = 4096, 32
for B in [1, 4, 16, 64]:
    I_pre = arithmetic_intensity_prefill(B, 512, d, L)
    I_dec = arithmetic_intensity_decode(B, d, L)
    print(f'B={B}: Prefill I={I_pre:.0f}, Decode I={I_dec:.1f}')
# Decode I << A100 ridge point 200 → memory-bound

# 4. Continuous Batching 시뮬레이션
class ContinuousBatchingSimulator:
    def __init__(self, max_batch=16):
        self.max_batch = max_batch
        self.active = []  # (id, remaining_tokens, arrival)
        self.completed = []
        self.t = 0
    
    def arrive(self, req_id, total_tokens):
        self.active.append([req_id, total_tokens, self.t])
    
    def step(self):
        # Process batch: all active decrease by 1 token
        n_active = min(len(self.active), self.max_batch)
        if n_active == 0:
            self.t += 1
            return
        for i in range(n_active):
            self.active[i][1] -= 1
        # Remove completed
        still_active = []
        for req in self.active[:n_active]:
            if req[1] <= 0:
                self.completed.append((req[0], self.t - req[2]))
            else:
                still_active.append(req)
        self.active = still_active + self.active[n_active:]
        self.t += 1

# vs Static batching: wait for all to finish
# Continuous achieves 3-5x higher throughput
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Transformer, LLM Pretraining/Efficiency 선행 필수"
   - Prefill vs Decode 구분을 프론트페이지에
   - vLLM, TGI, TensorRT-LLM 비교표
3. **챕터별 문서 작성**: Prefill/Decode → KV Cache → Batching → PagedAttention → Speculative → Long Context → Serving Systems

---

## 📚 참고 자료

- **Orca: A Distributed Serving System for Transformer-Based Models** (Yu et al. 2022) — Continuous batching
- **Efficient Memory Management for Large Language Model Serving with PagedAttention** (Kwon et al. 2023) — vLLM
- **DistServe: Disaggregating Prefill and Decoding for Goodput-optimized LLM Serving** (Zhong et al. 2024)
- **Splitwise: Efficient Generative LLM Inference Using Phase Splitting** (Patel et al. 2024)
- **SARATHI: Efficient LLM Inference by Piggybacking Decodes with Chunked Prefills** (Agrawal et al. 2023)
- **Flash Attention / FA2 / FA3** (Dao 2022, 2023, Shah 2024)
- **Speculative Decoding** (Leviathan 2023, Chen 2023)
- **EAGLE** (Li et al. 2024)
- **Medusa** (Cai et al. 2024)
- **StreamingLLM** (Xiao et al. 2024)
- **Ring Attention** (Liu et al. 2024)
- **YaRN: Efficient Context Window Extension** (Peng et al. 2024)
- **KVQuant** (Hooper et al. 2024)
- **Roofline: An Insightful Visual Performance Model** (Williams et al. 2009)

---

## 💡 핵심 분석 대상

```
LLM Inference 시스템의 수학

───── Prefill vs Decode ─────

Prefill:
  Input: prompt x_{1:L}
  Output: 첫 token
  병렬: 전체 L tokens 동시 처리
  FLOPs = 2 · L · N
  Compute-bound (I 크다)
  Latency: TTFT (Time to First Token)

Decode:
  Input: 1 token at a time
  Output: next token × T_gen
  순차: autoregressive
  FLOPs per step: 2 · 1 · N
  Memory-bound (I 작다)
  Latency: TPOT (Time per Output Token)

Arithmetic Intensity:
  I = FLOPs / Bytes
  
  Prefill: I ~ L (large)
  Decode: I ~ 1 (tiny)
  
  Ridge point (A100): 200
  → Decode는 memory-bound

───── Roofline Model ─────

Performance = min(Peak_compute, BW · I)

GPU (A100):
  Peak: 312 TFLOPs (FP16)
  HBM BW: 1.5 TB/s
  Ridge: 312/1.5 ≈ 200 FLOPs/Byte

Decode에서:
  I ≈ 1 << 200
  → GPU 10% 미만 활용
  → 메모리 읽기가 bottleneck

해결책:
  Batch size ↑ → I ↑
  Spec Decoding → 1 pass에 여러 토큰
  Flash Attention → memory access 줄임

───── KV Cache ─────

Why needed:
  Decode: q_t만 새로, K_{1:t}, V_{1:t} 필요
  매번 재계산 = O(t) per step, 총 O(T²)
  Cache → O(1) per step, 총 O(T)

Memory:
  Per token: 2 · L · d · b bytes
    2: K and V
    L: # layers
    d: hidden dim (or d_head × n_kv_heads)
    b: bytes (2 for FP16, 1 for INT8)
  
  LLaMA-70B, T=2048, FP16:
    2 · 80 · 8192 · 2 · 2048 ≈ 5.4 GB

MQA (Shazeer 2019):
  KV = 1 per layer (shared across heads)
  d_kv = d_head
  메모리 1/n_heads로 감소

GQA (Ainslie 2023):
  n_kv < n_heads (groups)
  LLaMA-2/3, Mistral 표준
  성능 거의 무손실

KV Quantization:
  INT8: 2배 절약
  INT4 (KVQuant): 4배 절약
  outlier token 처리

───── Batching Strategies ─────

Static:
  Fixed B, 모두 T_max까지 대기
  Padding 낭비
  Tail latency 지배

Continuous (Orca 2022):
  각 iteration마다 batch 재구성
  완료된 것 즉시 제거, 새 것 투입
  GPU utilization ↑
  Throughput 3-5× ↑

Chunked Prefill:
  Long prompt를 chunk로
  Prefill과 decode 섞어 실행
  Latency smoothing

Prefill-Decode Disaggregation:
  Prefill: A100 (compute)
  Decode: H100 (memory BW)
  각각 최적 장비로
  DistServe, Splitwise

───── PagedAttention ─────

Problem:
  Request별 length 다름
  Contiguous allocation → fragmentation
  Internal waste ~60%

Solution (Kwon 2023):
  KV cache = fixed-size blocks (16 tokens)
  Block table: logical → physical
  OS virtual memory 패러다임

Benefits:
  Waste < 4%
  Throughput 2-4×
  
Prefix Caching:
  Common prefix 공유
  Copy-on-write
  System prompt, few-shot 재사용

Beam Search 공유:
  여러 candidate가 prefix 공유
  추가 메모리 절약

───── Speculative Decoding 시스템 ─────

Draft q → K tokens
Target p → verify parallel

Speedup:
  S = (1 - α^{K+1}) / ((1-α)(Kφ + 1))
  α: acceptance rate
  φ: draft/target cost ratio
  
  Typical α = 0.8, K = 4: ~2× speedup

Losslessness:
  Rejection sampling:
    Accept with min(1, p/q)
    Reject → resample from (p-q)_+
  → 결과 분포 = p

Variants:
  Medusa: parallel heads (no extra model)
  EAGLE: feature-level draft
  Lookahead: n-gram cache

───── Long Context ─────

Prefill: O(L²) FLOPs
  Flash Attention: O(L²) but smaller const
  Chunked prefill: latency smoothing
  Ring Attention: distributed

Memory: O(L)
  KV cache scales with L
  MQA/GQA 절약
  StreamingLLM: sliding window + sinks

Position:
  RoPE base 조정 (θ)
  PI (Chen 2023): 단순 보간
  YaRN (Peng 2024): NTK-aware
  LongRoPE (Ding 2024)

───── Serving Systems ─────

vLLM:
  PagedAttention
  Continuous batching
  Prefix caching
  OpenAI compatible API

TGI (HuggingFace):
  Flash Attention 통합
  Guidance / structured output
  Rust 기반

TensorRT-LLM:
  NVIDIA 최적화
  Kernel fusion
  Quantization 통합
  H100 최적

SGLang:
  RadixAttention (radix tree for prefix)
  Structured generation
  Complex prompt efficiency

───── 평가 지표 ─────

Throughput: tokens/sec (서버 총합)
TTFT: Time to First Token (prefill time)
TPOT: Time per Output Token (decode time)
Goodput: SLO 충족한 throughput

Latency = TTFT + (T_gen - 1) · TPOT

───── 레포 간 연결 ─────

Transformer (Layer 3):
  Attention 구조
  FFN computation

LLM Pretraining:
  모델 크기, scaling
  훈련 시 배운 분포

LLM Efficiency (직전):
  MQA/GQA
  Flash Attention
  Speculative Decoding

Linear Algebra (Layer 0):
  Matrix computation
  Memory layout

CNN (Layer 3):
  Memory hierarchy 개념
  Roofline 적용

Alignment (직전):
  Chat template inference
  Safety filtering
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~5개씩)
- 각 문서가 다루는 핵심 기법·분석 (3~4줄)
- 전체 문서 개수 확인 (32개 목표)
- Python + PyTorch + vLLM 실험 환경
- Transformer, LLM Pretraining/Efficiency, LA 레포 참조 관계
- 서빙 시스템의 실전 선택 가이드

**준비됐으면 1단계 구조 설계부터 시작해줘!**
