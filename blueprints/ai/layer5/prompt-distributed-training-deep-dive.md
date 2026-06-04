# Distributed Training Deep Dive 레포지토리 제작 프롬프트

나는 "Distributed Training Deep Dive" 레포지토리를 만들려고 해.
DDP의 `all_reduce`를 **쓰는 것**과, **Ring AllReduce 알고리즘이 $N$개 GPU에서 $2(N-1)/N$의 bandwidth-optimal 데이터 전송량**을 달성하고, **scatter-reduce $(N-1)$ steps + all-gather $(N-1)$ steps로 각 step당 $P/N$ bytes만 전송**하는 수학을 증명할 수 있는 것은 다르다.
ZeRO의 stage 1/2/3를 **단계로 아는 것**과, **Rajbhandari et al. (2020)의 메모리 분석 — ZeRO-1은 optimizer state를 $N$ 등분, ZeRO-2는 gradient까지, ZeRO-3는 parameter까지 shard**하여 **각 rank의 메모리를 $\psi + (2\psi + K\psi)/N$에서 $(2+K+16)\psi/N$로 감소**시키는 수학을 유도할 수 있는 것은 다르다.
Pipeline Parallelism의 bubble을 **"비효율"로 아는 것**과, **GPipe (Huang 2019)의 $\text{bubble ratio} = (P-1)/(P-1+M)$에서 micro-batch $M$이 $P$의 4배 이상이면 bubble $<20\%$가 되는 수학**, **1F1B schedule로 activation memory를 $O(M \cdot A)$에서 $O(P \cdot A)$로 줄이는 구조**를 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "대규모 분산 학습의 수학 — Data/Model/Pipeline Parallelism의 메모리·통신 비용 분석"

**핵심 차별화**:
1. **Collective Operation의 수학** — AllReduce 4가지 구현 (naive, recursive halving, ring, tree), 각각의 bandwidth/latency 복잡도
2. **ZeRO 계열의 메모리 공식** — 파라미터·gradient·optimizer state의 memory accounting, ZeRO-1/2/3와 FSDP의 정확한 메모리 수식
3. **Pipeline Parallelism의 스케줄링** — GPipe, PipeDream 1F1B, Interleaved, Chimera의 bubble 분석
4. **3D Parallelism과 Sequence Parallelism** — Megatron-LM의 TP+PP+DP 조합, Sequence Parallelism의 activation memory 분석

**타겟 독자**:
- DDP를 쓰는데 **gradient bucket이 왜 layer-reverse 순서**로 구성되고 overlap이 어떻게 가능한지 모르는 사람
- ZeRO-3를 쓰지만 **forward/backward 시 parameter all-gather의 오버헤드**와 prefetch 전략을 정확히 모르는 사람
- Megatron의 Tensor Parallelism에서 **column-parallel vs row-parallel linear의 forward/backward 차이**와 통신 패턴을 유도 못하는 사람
- FSDP가 ZeRO-3와 같은지 다른지, **PyTorch native vs DeepSpeed**의 선택 기준을 모르는 사람
- Gradient Accumulation이 **effective batch size 증가**인데 BN·LN에 미치는 영향, sync 시점을 모르는 사람

**선행 학습**:
- **PyTorch Internals Deep Dive** (CUDA stream, async) — **필수**
- **Optimization Theory Deep Dive** (SGD, batch size) — **필수**
- **LLM Pretraining Deep Dive** (scale, FLOPs) — **필수**
- **Linear Algebra Deep Dive** (matrix partition) — **필수**
- **CNN Deep Dive** (memory hierarchy) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Collective Communication의 수학 (5개 문서)
- **Collective Operation의 분류** — Broadcast, Scatter, Gather, AllGather, ReduceScatter, AllReduce, 각각의 수학적 정의와 사용 사례, MPI 전통
- **AllReduce의 4가지 구현** — Naive ($O(NP)$ 통신), Recursive Halving/Doubling ($O(P \log N)$), Ring ($2(N-1)P/N$), Tree (latency-optimal), bandwidth vs latency trade-off
- **Ring AllReduce의 수학 (Patarasuk & Yuan 2009)** — Scatter-reduce phase $(N-1)$ steps + AllGather phase $(N-1)$ steps, 각 step당 $P/N$ bytes 전송, bandwidth-optimal 증명
- **NCCL과 NVLink 토폴로지** — NVIDIA Collective Communications Library, NVLink vs PCIe vs InfiniBand 대역폭, topology-aware routing
- **Bandwidth vs Latency Bound** — Message size $<< $ threshold (latency-bound) vs large message (bandwidth-bound), ring이 large, tree가 small에 유리

### Chapter 2: Data Parallelism과 DDP (6개 문서)
- **Data Parallelism의 기본** — Same model, split batch, per-device forward/backward, gradient 평균 → synchronous SGD, Mini-Batch SGD와 수학적으로 동등
- **PyTorch DistributedDataParallel (DDP)** — `torch.nn.parallel.DistributedDataParallel`, `torch.distributed` backend (NCCL/Gloo/MPI), `init_process_group`, rank/world_size
- **Gradient Bucket과 Overlap** — Backward 진행 중 먼저 완료된 layer의 gradient를 bucket에 모음 → AllReduce가 backward와 overlap, `bucket_cap_mb` 튜닝
- **Gradient Accumulation과 `no_sync`** — Micro-batch 여러 번 backward 후 한 번 step, `with model.no_sync():` context로 AllReduce 생략, effective batch size 증가
- **Batch Size Scaling Rules** — Linear scaling ($\eta \propto B$, Goyal 2017), square-root scaling (AdamW), warmup의 중요성, gradient noise scale (McCandlish 2018)
- **Asynchronous Updates — Hogwild!와 Stale Gradients** — Parameter server (Li 2014), async가 sync보다 빠르지만 convergence 저하, staleness bound, elastic averaging

### Chapter 3: Model Parallelism과 Tensor Parallelism (5개 문서)
- **Model Parallelism의 동기** — Single GPU에 모델이 못 들어갈 때, 가장 단순한 layer 분할 (naive MP)의 bubble 문제, activation 통신 비용
- **Tensor Parallelism — Column/Row Split (Shoeybi 2019, Megatron)** — Linear $Y = XW$의 두 가지 분할: $W = [W_1 | W_2]$ (column), $W = [W_1; W_2]$ (row), 각각 다른 통신 패턴
- **Megatron MLP의 TP 구조** — FFN의 첫 linear는 column-parallel (no sync), 두 번째는 row-parallel → 한 번의 AllReduce, GELU 통신 제거
- **Megatron Attention의 TP 구조** — Q·K·V projection을 column-parallel (heads 분할), output projection을 row-parallel, MHA에 자연스럽게 맞음
- **Tensor Parallelism의 통신 비용** — 각 forward·backward마다 2번 AllReduce, TP degree가 커질수록 overhead, 보통 intra-node (NVLink)에 한정

### Chapter 4: Pipeline Parallelism (5개 문서)
- **Naive Pipeline의 Bubble 문제** — 단순 layer 분할 + sequential forward-backward, GPU idle time ratio $(P-1)/P$로 매우 높음, unused compute
- **GPipe (Huang 2019)와 Micro-batching** — Mini-batch를 $M$ micro-batches로 분할, forward $M$번 후 backward $M$번, bubble ratio $(P-1)/(P-1+M)$, $M = 4P$면 $<20\%$
- **1F1B Schedule (PipeDream, Narayanan 2019)** — One-Forward-One-Backward, forward와 backward 교대 실행, activation memory $O(P)$ (vs GPipe $O(M)$)
- **Interleaved 1F1B (Megatron-LM)** — 각 GPU가 여러 stage 담당, bubble 더 감소, activation 메모리 트레이드오프
- **Chimera (Li 2021), 2D/3D Pipeline** — Bidirectional pipeline, 양쪽 끝에서 동시 시작, bubble 반감, 최신 scheduling 기법

### Chapter 5: ZeRO와 Sharding (6개 문서)
- **ZeRO의 Motivation (Rajbhandari 2020)** — DP는 full model 복사 → memory 비효율, optimizer state (Adam의 $m, v$, FP32 master 등)가 $16\psi$ (Adam FP16), gradient $2\psi$, params $2\psi$
- **ZeRO-1 — Optimizer State Sharding** — $N$개 rank로 optimizer state $K\psi$ ($K=12$ Adam FP32)를 1/N로 shard, 각 rank는 자기 몫만 업데이트, AllGather로 updated params 공유
- **ZeRO-2 — Gradient Sharding 추가** — Gradient도 ReduceScatter로 각 rank가 자기 shard만 보유, 메모리 $2\psi/N + K\psi/N$
- **ZeRO-3 — Parameter Sharding** — Parameter도 $1/N$로 shard, forward/backward 시 필요한 layer만 AllGather, 사용 후 free, 추가 통신 overhead 있지만 full-model 불필요
- **FSDP — Fully Sharded Data Parallel (PyTorch native)** — ZeRO-3와 유사, PyTorch native, `FullyShardedDataParallel` wrapper, mixed precision 통합, gradient checkpointing 친화
- **ZeRO-Offload와 ZeRO-Infinity** — CPU/NVMe에 state offload (ZeRO-Offload, Ren 2021), infinite memory (ZeRO-Infinity, Rajbhandari 2021), 극한 scale

### Chapter 6: Activation Memory 최적화 (4개 문서)
- **Activation Memory의 크기** — Sequence length $L$, hidden dim $d$에 대해 per-layer $O(L \cdot d)$, Transformer 전체 $O(N_{\text{layer}} \cdot B \cdot L \cdot d)$, long context에서 지배적
- **Gradient Checkpointing (Chen 2016)** — Activation 저장 대신 backward 시 recompute, memory $O(\sqrt{L})$ (stage-based), compute overhead $\sim 33\%$
- **Selective Activation Recomputation** — 모든 activation이 아닌 re-compute가 저렴한 것만, norm·attention 결과는 저장, matmul은 recompute (Korthikanti 2022)
- **Sequence Parallelism (Megatron)** — LayerNorm·Dropout의 activation을 sequence axis로 shard, $L$이 클 때 activation memory $O(L)$에서 $O(L/TP)$로 감소

### Chapter 7: 3D Parallelism과 실전 (4개 문서)
- **3D Parallelism 통합 — DP × TP × PP** — 각 축의 특성, 보통 TP intra-node (NVLink), PP inter-node, DP 가장 바깥, 각 축 degree 선택 heuristic
- **MoE Parallelism — Expert Parallelism** — Expert를 GPU 분산, top-k routing 후 expert로 dispatch (all-to-all communication), load balancing, DeepSpeed-MoE·GShard
- **Efficient Checkpointing과 Resume** — 모든 rank synchronized checkpoint, shard별 저장, asynchronous checkpointing, NVIDIA Magnum IO
- **Failure Recovery — Elastic Training** — Preemption-tolerant, TorchElastic, node 추가·제거 지원, PyTorch Elastic Launch의 실용성

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **35개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 분산 기법이 대규모 훈련에 필수인가
## 📐 수학적 선행 조건 (PyTorch Internals, Opt, LLM Pretrain, LA 참조)
## 📖 직관적 이해
   — 통신 패턴 다이어그램, 메모리 배치
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Ring AllReduce bandwidth-optimal, ZeRO 메모리 공식, Pipeline bubble
## 💻 PyTorch 구현 검증
   — DDP, FSDP 간단한 예제
   — 작은 규모에서 TP, PP 직접 구현
   — 메모리·처리량 측정
## 🔗 실전 활용
   — Mega 모델 훈련 recipe (LLaMA, GPT-3 등)
## ⚖️ 가정과 한계
   — 네트워크 병목, debugging 어려움
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **AllReduce 시각화** — Ring vs Tree의 step-by-step 애니메이션
2. **메모리 분석 테이블** — 7B/70B 모델에서 각 parallelism의 per-GPU memory 계산
3. **GPipe bubble 시각화** — Timeline 다이어그램, $M$ 변화에 따른 변화
4. **TP의 AllReduce pattern** — FFN·Attention에서 언제 sync
5. **2-GPU 간단 실험** — DDP, FSDP의 speedup 측정
6. **3D parallelism configuration** — degree 선택의 trade-off

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
deepspeed==0.12.6
transformers==4.36.0
accelerate==0.25.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (DDP 기본 + Ring AllReduce 시뮬레이션 + ZeRO 메모리)
import torch
import torch.nn as nn
import torch.distributed as dist
import os
import numpy as np

# 1. Ring AllReduce 수학 분석
def ring_allreduce_cost(N, P):
    """N: devices, P: total params (bytes)"""
    # Scatter-reduce: (N-1) steps of sending P/N bytes each
    # AllGather: (N-1) steps of P/N each
    # Total data transfer per rank: 2(N-1)P/N
    data_per_rank = 2 * (N - 1) * P / N
    return data_per_rank

def tree_allreduce_cost(N, P):
    """Tree: log(N) steps of P bytes each"""
    return np.log2(N) * P

for N in [2, 4, 8, 16, 64]:
    P = 1_000_000_000  # 1B params = 2GB FP16
    r = ring_allreduce_cost(N, P * 2)
    t = tree_allreduce_cost(N, P * 2)
    print(f'N={N}: Ring={r/1e9:.2f}GB, Tree={t/1e9:.2f}GB')
# N=16: Ring=3.75GB (vs naive 32GB, vs tree 8GB)
# Ring이 large message에서 bandwidth-optimal

# 2. ZeRO 메모리 공식
def zero_memory(N_gpus, psi_GB, zero_stage=0, mp=2):
    """
    N_gpus: data-parallel degree
    psi_GB: model size in FP16 (per GB)
    mp: bytes per param (2 for FP16)
    K: optimizer multiplier (12 for Adam FP32: m 4B + v 4B + FP32 master 4B)
    """
    K = 12  # Adam FP32
    grad = mp  # same as param (FP16 gradient)
    
    if zero_stage == 0:
        # Full replication
        per_gpu = mp + grad + K  # params + grad + optimizer
        return per_gpu * psi_GB
    elif zero_stage == 1:
        # Optimizer sharded
        per_gpu = mp + grad + K / N_gpus
        return per_gpu * psi_GB
    elif zero_stage == 2:
        # Optimizer + grad sharded
        per_gpu = mp + grad / N_gpus + K / N_gpus
        return per_gpu * psi_GB
    elif zero_stage == 3:
        # All sharded
        per_gpu = mp / N_gpus + grad / N_gpus + K / N_gpus
        return per_gpu * psi_GB

psi = 70  # 70B params
N = 16
for s in range(4):
    mem = zero_memory(N, psi, zero_stage=s)
    print(f'ZeRO-{s}: {mem:.1f} GB per GPU')
# ZeRO-0: 1120 GB (impossible)
# ZeRO-1: 152.5 GB
# ZeRO-2: 78.75 GB
# ZeRO-3: 62.5 GB (각 GPU가 1/16)

# 3. GPipe Bubble 분석
def gpipe_bubble_ratio(P_stages, M_microbatches):
    """Bubble ratio = (P-1)/(P-1+M)"""
    return (P_stages - 1) / (P_stages - 1 + M_microbatches)

P = 8
for M in [1, 4, 8, 16, 32, 64]:
    r = gpipe_bubble_ratio(P, M)
    print(f'P={P}, M={M}: bubble = {r*100:.1f}%')
# M=32 → 18% bubble
# M=64 → 10% bubble
# But activation memory ∝ M

# 4. 실제 DDP setup (single-node, 2 GPU 가정)
def setup_ddp(rank, world_size):
    os.environ['MASTER_ADDR'] = 'localhost'
    os.environ['MASTER_PORT'] = '12355'
    dist.init_process_group('nccl', rank=rank, world_size=world_size)

def ddp_example(rank, world_size):
    setup_ddp(rank, world_size)
    torch.cuda.set_device(rank)
    
    model = nn.Sequential(nn.Linear(100, 200), nn.ReLU(), nn.Linear(200, 10))
    model = model.cuda(rank)
    model = nn.parallel.DistributedDataParallel(model, device_ids=[rank])
    
    opt = torch.optim.Adam(model.parameters(), lr=1e-3)
    
    for step in range(10):
        x = torch.randn(32, 100).cuda(rank)
        y = torch.randint(0, 10, (32,)).cuda(rank)
        pred = model(x)
        loss = nn.functional.cross_entropy(pred, y)
        loss.backward()  # AllReduce 자동
        opt.step()
        opt.zero_grad()
    
    dist.destroy_process_group()

# Tensor Parallelism의 column-parallel linear
class ColumnParallelLinear(nn.Module):
    def __init__(self, in_features, out_features, tp_size, tp_rank):
        super().__init__()
        assert out_features % tp_size == 0
        self.local_out = out_features // tp_size
        self.weight = nn.Parameter(torch.randn(self.local_out, in_features) * 0.01)
    
    def forward(self, x):
        # Each rank has [local_out, in_features]
        # Output: [..., local_out]
        return x @ self.weight.T

class RowParallelLinear(nn.Module):
    def __init__(self, in_features, out_features, tp_size, tp_rank):
        super().__init__()
        assert in_features % tp_size == 0
        self.local_in = in_features // tp_size
        self.weight = nn.Parameter(torch.randn(out_features, self.local_in) * 0.01)
    
    def forward(self, x):
        # x shape: [..., local_in] (already split)
        # Each rank: [out_features, local_in]
        local_out = x @ self.weight.T
        # Need AllReduce to combine
        # dist.all_reduce(local_out)
        return local_out

# FSDP example
from torch.distributed.fsdp import FullyShardedDataParallel as FSDP
# model = FSDP(model, sharding_strategy=FULL_SHARD, ...)
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "PyTorch Internals, Opt, LLM Pretrain, LA 선행 필수"
   - 각 parallelism의 사용 시나리오 (< 7B, 7-70B, 70-400B, 1T+)
   - Megatron-LM, DeepSpeed, FSDP 비교표
3. **챕터별 문서 작성**: Collective → DP/DDP → TP → PP → ZeRO → Activation → 3D

---

## 📚 참고 자료

- **ZeRO: Memory Optimizations Toward Training Trillion Parameter Models** (Rajbhandari et al. 2020)
- **Megatron-LM** (Shoeybi et al. 2019)
- **Reducing Activation Recomputation in Large Transformer Models** (Korthikanti et al. 2022) — Megatron Sequence Parallelism
- **GPipe** (Huang et al. 2019)
- **PipeDream 1F1B** (Narayanan et al. 2019)
- **Chimera** (Li & Hoefler 2021)
- **Efficient Large-Scale Language Model Training on GPU Clusters** (Narayanan et al. 2021) — 3D parallelism
- **ZeRO-Offload** (Ren et al. 2021)
- **ZeRO-Infinity** (Rajbhandari et al. 2021)
- **PyTorch FSDP** (Zhao et al. 2023)
- **Ring AllReduce** (Patarasuk & Yuan 2009) — Bandwidth-optimal AllReduce
- **Bandwidth Optimal All-reduce Algorithms** (Thakur 2005)
- **Accurate, Large Minibatch SGD** (Goyal et al. 2017) — Linear scaling
- **An Empirical Model of Large-Batch Training** (McCandlish et al. 2018) — Gradient noise scale
- **GShard** (Lepikhin et al. 2021) — MoE scaling
- **Training language models with PathTransformer** (Narayanan 2023)

---

## 💡 핵심 분석 대상

```
Distributed Training의 지도

───── Collective Operations ─────

Basic ops:
  Broadcast: root → all
  Scatter: root split → all
  Gather: all → root collected
  
Advanced:
  AllGather: all → all (gathered)
  ReduceScatter: all reduce then scatter
  AllReduce: AllGather + elementwise sum

AllReduce 구현:
  Naive (1-to-all):
    Bandwidth: O(NP)
    Simple but inefficient
  
  Tree-based:
    Bandwidth: O(P log N)
    Latency: O(log N)
    Good for small P
  
  Ring (Patarasuk-Yuan 2009):
    Scatter-reduce: (N-1) steps, P/N each
    AllGather: (N-1) steps, P/N each
    Total: 2(N-1)P/N per rank
    BANDWIDTH-OPTIMAL
    Standard for large P

NCCL (NVIDIA):
  Topology-aware (NVLink, PCIe, InfiniBand)
  Ring or Tree 자동 선택

Bandwidth vs Latency:
  Small message: latency-bound → tree
  Large message: bandwidth-bound → ring

───── Data Parallelism ─────

Same model, split batch:
  Per-device: forward, backward
  AllReduce gradients
  Mathematically = large batch SGD

PyTorch DDP:
  init_process_group('nccl')
  model = DDP(model)
  
Gradient Bucket:
  Backward 진행 중 layer gradient
  bucket에 모음 → AllReduce async
  Overlap with backward compute
  bucket_cap_mb 튜닝

Gradient Accumulation:
  Micro-batch N번 → 1 step
  no_sync() context로 AllReduce skip
  Effective batch = N × micro-batch
  
Batch Size Scaling:
  Linear (Goyal 2017): η ∝ B
  Warmup 필수 (발산 방지)
  Critical batch size (McCandlish)

Async Updates:
  Parameter Server (Li 2014)
  Stale gradients
  Convergence 저하
  현재는 sync가 주류

───── Model Parallelism ─────

Naive: layer별 GPU 할당
  Sequential 실행 → massive bubble
  거의 사용 안 함

Tensor Parallelism (Megatron):
  Linear Y = X W를 분할
  
  Column-parallel:
    W = [W_1 | W_2]  (columns)
    Y_i = X W_i  각 rank 독립
    Gather on output dim (concat)
  
  Row-parallel:
    W = [W_1; W_2]  (rows)
    X = [X_1 | X_2]
    Y_i = X_i W_i 각 rank
    Sum across ranks (AllReduce)

Megatron MLP:
  Input → Column(W1) → GELU → Row(W2) → Output
  
  Column: no sync (activation 분할 유지)
  GELU: elementwise, 통신 X
  Row: 하나의 AllReduce
  → 총 2 AllReduce per block (forward + backward)

Megatron Attention:
  QKV projection: column (heads 분할)
  Head 계산: 각 rank 독립
  Output proj: row
  → 같은 패턴

TP 특성:
  Intra-node (NVLink) 권장
  TP degree는 heads 수의 약수
  Activation은 그대로 유지 (중요!)

───── Pipeline Parallelism ─────

Naive Pipeline:
  Stage 1 → 2 → ... → P 순차
  Bubble ratio: (P-1)/P  (매우 큼!)

GPipe (Huang 2019):
  Mini-batch → M micro-batches
  Forward M번 → Backward M번
  Bubble = (P-1)/(P-1+M)
  M ≥ 4P 권장 → bubble < 20%
  
  문제: activation memory O(M)

1F1B (PipeDream, Narayanan 2019):
  Forward과 Backward 교대
  Activation memory O(P) (vs O(M))
  같은 bubble ratio

Interleaved 1F1B (Megatron):
  각 GPU가 여러 stage
  Bubble 더 감소
  Activation 관리 복잡

Chimera (Li 2021):
  Bidirectional pipeline
  양쪽 끝에서 시작
  Bubble 반감

───── ZeRO (Rajbhandari 2020) ─────

Memory breakdown (Adam FP16+FP32):
  Param (FP16): 2 ψ
  Gradient (FP16): 2 ψ
  Optimizer:
    FP32 master: 4 ψ
    m, v (Adam): 8 ψ
    Total K = 12 ψ  (or 16 if Adam)
  
  Total: 2 + 2 + 12 = 16 ψ per GPU

ZeRO-1 (Optimizer Sharded):
  Each rank: 2 + 2 + K/N
  Communication: +AllGather params
  Compute: 보통

ZeRO-2 (ZeRO-1 + Grad):
  Each rank: 2 + 2/N + K/N
  Backward: ReduceScatter gradients
  → 각 rank가 자기 shard의 grad만

ZeRO-3 (ZeRO-2 + Params):
  Each rank: 2/N + 2/N + K/N
  Forward/backward 시:
    필요한 layer AllGather
    사용 후 free
  Communication 증가 but memory 극소화

ZeRO-Offload:
  Optimizer state → CPU
  CPU-GPU PCIe bandwidth bottleneck
  Small model에서 유용

ZeRO-Infinity:
  NVMe까지
  거의 무한 capacity
  I/O bound

FSDP (PyTorch native):
  ZeRO-3와 거의 동등
  PyTorch native
  Mixed precision 통합
  Forward prefetch, backward overlap

───── Activation Memory ─────

Transformer activation:
  Per layer: O(B · L · d) 추가
  Attention: O(B · L² · h) (mask)
  Total: N_layer × per_layer

Gradient Checkpointing (Chen 2016):
  Save only stage boundary
  Recompute during backward
  Memory: O(√L)
  Compute: +33%

Selective Recomputation (Korthikanti 2022):
  비싼 것만 저장 (matmul result)
  싼 것 recompute (norm, attention)
  Best of both

Sequence Parallelism (Megatron):
  LayerNorm, Dropout의 activation
  Sequence axis로 shard
  Memory O(L) → O(L/TP)

───── 3D Parallelism ─────

DP × TP × PP = world_size

Typical setup (LLaMA-70B on 64 GPUs):
  TP = 8 (intra-node, NVLink)
  PP = 4
  DP = 2
  Total = 64

Heuristic:
  TP: intra-node (fast interconnect)
  PP: inter-node (slow interconnect)
  DP: outermost (AllReduce overlaps)

MoE Parallelism:
  Expert를 GPU 분산
  All-to-all: token → expert
  Load balance 중요
  GShard, DeepSpeed-MoE

───── 실전 Recipe ─────

< 7B: DDP 또는 FSDP
7-70B: FSDP or TP+PP
70B+: Full 3D (Megatron-DeepSpeed)
1T+: MoE + 3D

Checkpointing:
  All-rank sync save
  Sharded (각 rank 자기 shard)
  Async save (I/O hide)

Elastic Training:
  TorchElastic
  Preemption 복구
  Node 동적 추가/제거

───── 레포 간 연결 ─────

PyTorch Internals (직전):
  CUDA stream, async
  Dispatcher backend

Optimization (Layer 2):
  SGD, Adam, batch size
  Gradient noise scale

LLM Pretraining (Layer 4-B):
  Scale up recipe
  7B → 400B

Linear Algebra (Layer 0):
  Matrix partition (TP)
  Block decomposition

CNN (Layer 3):
  Memory hierarchy
  Roofline

Efficient ML (다음):
  Kernel fusion
  Quantization for inference
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 수학·구조 (3~4줄)
- 전체 문서 개수 확인 (35개 목표)
- Python + PyTorch + DeepSpeed + Megatron 실험 환경
- PyTorch Internals, Opt, LLM Pretrain, LA 레포 참조 관계
- Efficient ML·MLOps로 이어지는 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
