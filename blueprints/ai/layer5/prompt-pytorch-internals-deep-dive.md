# PyTorch Internals Deep Dive 레포지토리 제작 프롬프트

나는 "PyTorch Internals Deep Dive" 레포지토리를 만들려고 해.
`loss.backward()`를 **호출하는 것**과, **Autograd의 reverse-mode AD가 $\frac{\partial L}{\partial x} = J_f^T v$ Jacobian-Vector Product (VJP)를 computation graph의 역순으로 누적**하고, 각 `Function`의 `forward` → `ctx.save_for_backward(...)` → `backward(grad_out)`가 어떻게 **chain rule을 국소적으로 구현**하는지 증명할 수 있는 것은 다르다.
`torch.autocast`로 AMP를 **쓰는 것**과, **FP16의 표현 가능 범위 $\approx [6.1 \times 10^{-5}, 6.5 \times 10^4]$와 gradient의 overflow/underflow 분석**, **loss scaling $L_{\text{scaled}} = L \cdot S$가 어떻게 small gradient를 FP16 표현 범위로 끌어올리고** unscale 시 정확성을 보장하는지 수치적 근거로 이해할 수 있는 것은 다르다.
CUDA kernel을 **이름으로 아는 것**과, **GPU의 SM·warp·shared memory 계층 구조**와, **reduction의 tree-based vs warp shuffle 구현 차이**, **memory coalescing이 $100 \times$ bandwidth 차이**를 만드는 이유를 roofline model로 이해할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "PyTorch의 내부 — Autograd·Dispatcher·CUDA·AMP의 시스템 수학"

**핵심 차별화**:
1. **Autograd의 엄밀한 수학** — Reverse-mode AD = VJP, computation graph, `Function` API 직접 구현, 고차 미분의 double backward
2. **PyTorch Dispatcher 시스템** — Tensor 하나의 `.add()` 호출이 device·dtype·layout별로 routing되는 과정, `TORCH_LIBRARY` 커스텀 operator
3. **CUDA Programming 기초와 kernel** — SM, warp, shared memory, memory coalescing, tree reduction, torch.utils.cpp_extension으로 custom CUDA kernel
4. **Mixed Precision 수치 분석** — IEEE 754 FP16/BF16/FP32/TF32 표현 범위, gradient scaling, Kahan summation, loss scaling의 수학적 정당성

**타겟 독자**:
- `loss.backward()`가 작동하는 이유를 **"chain rule이니까"**로만 아는데 **VJP vs JVP의 차이, 왜 ML에서는 VJP가 자연스러운지** (loss: scalar, params: vector) 수학적으로 유도 못하는 사람
- AMP를 쓰는데 **GradScaler의 `scale()`·`unscale_()`·`update()` 각각이 어떤 수학적 연산**이고 overflow/underflow를 어떻게 감지하는지 모르는 사람
- CUDA kernel을 보긴 했는데 **warp의 32 thread가 SIMT로 동작하는 것과 warp divergence**, **shared memory의 32 bank**를 이해 못하는 사람
- `torch.compile`이 무엇인지는 아는데 **TorchDynamo가 bytecode 수준에서 graph 추출**하고 **TorchInductor가 Triton으로 kernel을 생성**하는 파이프라인을 모르는 사람
- `nn.Module.to(device)`가 **parameter의 device 이동과 autograd graph가 어떻게 호환**되는지 모르는 사람

**선행 학습**:
- **Neural Network Theory Deep Dive** (Backprop) — **필수**
- **Calculus & Optimization Deep Dive** (Jacobian, chain rule) — **필수**
- **Linear Algebra Deep Dive** (matrix operations) — **필수**
- **CNN Deep Dive** (memory hierarchy, roofline) — 권장
- **LLM Inference Deep Dive** (roofline, HBM/SRAM) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Tensor의 내부 구조 (5개 문서)
- **Tensor = Storage + Metadata** — `storage` (contiguous memory), `size`, `stride`, `offset`, `dtype`, `device`, 같은 storage 공유로 view 구현 (`.view()`, `.transpose()`), contiguous 여부의 의미
- **Stride와 Memory Layout** — Row-major (C-order) vs column-major (F-order), NCHW vs NHWC, 각 layout의 memory access pattern, `is_contiguous()` 검사
- **View vs Copy의 구분** — `.view()`는 stride 조작으로 무복사, `.reshape()`은 필요시 copy, `.contiguous()`의 역할, `.permute()` 뒤 `.contiguous()` 필요 사례
- **Dtype 계층과 Type Promotion** — `torch.float32` (FP32), `torch.float16` (FP16/half), `torch.bfloat16` (BF16), `torch.int64`, type promotion rules (`0.5 + 1 → float`), casting overhead
- **Device와 CUDA Context** — CPU·CUDA·MPS·XLA, `torch.cuda.current_device()`, stream 관리, async execution, `torch.cuda.synchronize()`의 의미

### Chapter 2: Autograd의 수학과 구현 (6개 문서)
- **Automatic Differentiation의 두 모드** — Forward-mode (JVP) vs Reverse-mode (VJP), 수학적 비교, ML에서 VJP가 자연스러운 이유 (scalar loss, vector params), complexity $O(n \cdot m)$ vs $O(m)$
- **VJP의 수학** — $\frac{\partial L}{\partial x} = J_f^T \frac{\partial L}{\partial y}$, upstream gradient $v = \frac{\partial L}{\partial y}$, local Jacobian의 transpose와의 product, 각 operator의 VJP 정의
- **Computation Graph의 구성** — Dynamic graph (define-by-run), `torch.Tensor.requires_grad`, `grad_fn` chain, leaf vs intermediate tensor, `torch.no_grad()` context
- **Backward의 Topological Sort** — Graph의 reverse topological order로 VJP 전파, `torch.autograd.grad()` API, multiple outputs의 경우 gradient 누적
- **torch.autograd.Function 직접 구현** — `forward(ctx, *inputs)` + `backward(ctx, *grad_outputs)`, `ctx.save_for_backward`, custom activation/operator 구현 예시
- **Higher-Order Gradients — Double Backward** — $\nabla^2$ 계산, `create_graph=True`, physics-informed NN, Hessian-vector product, second-order optimization의 기반

### Chapter 3: PyTorch Dispatcher와 Backend (5개 문서)
- **Dispatcher Architecture** — `aten::add`가 dtype·device·layout·autograd 기반으로 dispatch, DispatchKey enum, DispatchKeySet, BackendSelect
- **Aten과 C10 — PyTorch의 Layered 구조** — `c10` (core tensor/device types), `aten` (tensor operations), `torch` (Python binding), kernel implementation의 계층
- **Autograd Key와 Backward Dispatch** — Forward kernel과 backward kernel의 분리 등록, `torch::autograd::Function` C++ API, autograd dispatch의 위치
- **TORCH_LIBRARY와 Custom Operator** — `TORCH_LIBRARY_IMPL` 매크로로 새 op 등록, Python에서 `torch.ops.mylib.myop` 호출, AOTAutograd·Inductor 통합
- **Functorch와 vmap, grad** — JAX-inspired transformations, `torch.func.vmap`, `torch.func.grad`, `torch.func.jacrev/jacfwd`, batched computation

### Chapter 4: CUDA Programming 기초 (6개 문서)
- **GPU 아키텍처 — SM, Warp, Thread Block** — Streaming Multiprocessor, warp = 32 threads (SIMT), thread block 내 shared memory, grid-block-thread hierarchy
- **Memory Hierarchy — HBM/L2/L1/Shared/Register** — Size와 bandwidth·latency 순서, register (per thread) → shared (per block) → L1/L2 cache → HBM, data locality의 중요성
- **Memory Coalescing의 수학** — 같은 warp의 threads가 연속 memory access → 1 transaction으로 통합, non-coalesced는 $100\times$ slower, stride pattern 분석
- **Bank Conflict와 Shared Memory 최적화** — Shared memory 32 banks, 같은 bank 다른 address 접근 시 serial, padding trick, reduce·transpose kernel의 bank-free 설계
- **Warp Divergence와 SIMT 병목** — Conditional branch 시 warp 내 threads가 다른 path → serial execution, branch 최소화 코딩 패턴
- **Reduction Kernel 최적화 — Tree vs Warp Shuffle** — Naive $O(n)$ → tree-based $O(\log n)$ → warp shuffle instruction (`__shfl_down_sync`), Mark Harris 7-optimization 시퀀스

### Chapter 5: PyTorch-CUDA 통합과 Custom Kernel (5개 문서)
- **torch.utils.cpp_extension** — `load()`로 runtime JIT compile, `setup.py` setuptools 통합, CPU·CUDA 혼합 source, 예시 (fused Linear+ReLU)
- **Tensor와 CUDA Pointer** — `tensor.data_ptr()`, `tensor.is_contiguous()`, stride 전달 필수, dtype 특화 dispatch (`AT_DISPATCH_FLOATING_TYPES`)
- **Triton — Python-level GPU Programming (Tillet 2019)** — CUDA보다 간단한 block-level abstraction, `@triton.jit` decorator, pointer arithmetic, LLM 시대의 de facto kernel language
- **cuBLAS·cuDNN의 활용** — Matrix multiply는 cuBLAS가 optimal, convolution은 cuDNN, PyTorch의 내부 dispatch, GEMM의 cost ($2MNK$ FLOPs)
- **Kernel Fusion의 동기** — 여러 elementwise op → 한 kernel로 합치기, HBM 왕복 제거, TorchInductor·Triton의 자동 fusion, attention fusion 예시

### Chapter 6: Mixed Precision Training (5개 문서)
- **IEEE 754 Floating Point — FP32/FP16/BF16/TF32** — FP32 (1+8+23), FP16 (1+5+10), BF16 (1+8+7), TF32 (A100 기본, 1+8+10), mantissa/exponent의 range·precision 차이
- **FP16의 Range Problem** — Max $\approx 6.5 \times 10^4$, min positive normal $\approx 6.1 \times 10^{-5}$, gradient가 $10^{-8}$ 이하면 zero로 underflow, weight가 $10^5$ 이상이면 $\infty$로 overflow
- **Loss Scaling과 GradScaler** — $L_{\text{scaled}} = L \cdot S$ ($S \approx 2^{16}$), gradient도 $S$배 → FP16 표현 가능 범위로, optimizer step 전 $1/S$로 unscale, NaN/Inf 감지 시 $S$ 감소
- **BF16의 장점과 TF32** — BF16은 FP32와 같은 range (exponent 8bit), mantissa 7bit로 precision만 손해, loss scaling 불필요, TF32 (A100+)는 FP32 input을 내부 FP16-like로, transparent 가속
- **Stochastic Rounding과 Kahan Summation** — Round-to-nearest vs stochastic rounding (INT8 training), accumulator precision 문제, Kahan summation으로 compensation

### Chapter 7: torch.compile과 현대 PyTorch (5개 문서)
- **torch.compile — PT 2.0 Revolution** — 기존 eager mode 유지하면서 JIT compile, `@torch.compile` decorator, first iteration compile 후 cached
- **TorchDynamo — Bytecode-level Graph Capture** — Python frame evaluation API (PEP 523) 활용, CPython bytecode 수정, graph break 시 fallback, `torch._dynamo.explain`
- **AOTAutograd — Graph at Training Time** — Forward + backward graph 통합, functionalization, 효율적 grad computation, decomposition (primitive op로 축소)
- **TorchInductor — Code Generation** — FX graph → Triton kernel (GPU) or C++ (CPU), fusion heuristic, reduction·elementwise·matmul 각각 전략
- **Distributed Training과의 통합** — DDP와 torch.compile 호환, `torch.distributed`의 collective op, FSDP와의 상호작용 (Distributed Training 레포 bridge)

---

각 챕터는 **5~6개 문서**로 구성해줘. 총 **37개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 내부 구조가 PyTorch 사용자에게 중요한가
## 📐 수학적 선행 조건 (NN Theory, LA, Calc 참조)
## 📖 직관적 이해
   — Computation graph 시각화, memory layout diagram
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — VJP 유도, Loss scaling 근거, Memory coalescing 분석
## 💻 PyTorch + CUDA 구현 검증
   — Custom autograd.Function
   — CUDA kernel JIT compile
   — AMP with/without 비교
## 🔗 실전 활용
   — 성능 최적화 가이드, profiling tools
## ⚖️ 가정과 한계
   — 자동화의 한계, 수동 튜닝 필요 사례
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Computation graph 시각화** — `torchviz.make_dot`로 실제 graph 그리기
2. **Stride 실험** — 같은 data, 다른 stride로 tensor 생성
3. **Custom Function** — 직접 구현한 ReLU vs `F.relu` backward 결과 비교
4. **CUDA kernel 속도** — Naive reduction vs tree vs warp shuffle 측정
5. **AMP overflow 재현** — FP16 underflow 유발 → loss scaling으로 해결
6. **torch.compile 효과** — 같은 모델 compile 전후 throughput

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
torchvision==0.16.0
triton==2.1.0          # Python-level CUDA
nvidia-ml-py==12.535   # GPU monitoring
torchviz==0.0.2        # graph 시각화
py-spy==0.3.14         # profiling
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Custom autograd.Function + VJP 유도 + CUDA kernel)
import torch
import torch.nn as nn
import numpy as np

# 1. Custom autograd.Function — ReLU 바닥부터
class MyReLU(torch.autograd.Function):
    @staticmethod
    def forward(ctx, x):
        ctx.save_for_backward(x)
        return torch.clamp(x, min=0)
    
    @staticmethod
    def backward(ctx, grad_output):
        """
        d/dx max(0, x) = 1 if x > 0, else 0
        VJP: J^T v = diag(x > 0) · v
        """
        x, = ctx.saved_tensors
        grad_input = grad_output.clone()
        grad_input[x < 0] = 0
        return grad_input

# Verify
x = torch.randn(5, requires_grad=True)
y1 = MyReLU.apply(x); y1.sum().backward()
g1 = x.grad.clone()
x.grad.zero_()
y2 = torch.nn.functional.relu(x); y2.sum().backward()
g2 = x.grad.clone()
print(f'Max gradient diff: {(g1 - g2).abs().max():.2e}')  # 0

# 2. VJP vs JVP 수식 검증
# f(x) = x^T A x (quadratic form)
A = torch.tensor([[2., 1.], [1., 3.]])
x = torch.tensor([1., 2.], requires_grad=True)
f = x @ A @ x

# Analytic: ∇f = (A + A^T) x = 2 A x (if A symmetric)
analytic = (A + A.T) @ x
# Autograd (VJP with v=1)
f.backward()
print(f'Analytic: {analytic}, Autograd: {x.grad}')  # same

# 3. AMP Loss Scaling 실험
# FP16 overflow 시뮬레이션
from torch.cuda.amp import autocast, GradScaler

class SmallNet(nn.Module):
    def __init__(self):
        super().__init__()
        self.fc = nn.Linear(10, 1)
    def forward(self, x):
        return self.fc(x) * 1e-5  # tiny output

if torch.cuda.is_available():
    model = SmallNet().cuda()
    opt = torch.optim.SGD(model.parameters(), lr=1.0)
    scaler = GradScaler()
    
    x = torch.randn(32, 10).cuda()
    y = torch.randn(32, 1).cuda() * 1e-3
    
    with autocast():
        pred = model(x)
        loss = ((pred - y)**2).mean()
    
    # 1. Without scaling
    g_noscale = torch.autograd.grad(loss, model.fc.weight, retain_graph=True)[0]
    print(f'No scaling gradient norm: {g_noscale.float().norm():.2e}')
    print(f'Zero count: {(g_noscale == 0).sum().item()}')  # possible underflow
    
    # 2. With scaling (S = 2^16)
    scaled_loss = scaler.scale(loss)
    g_scaled = torch.autograd.grad(scaled_loss, model.fc.weight, retain_graph=True)[0]
    g_unscaled = g_scaled.float() / scaler.get_scale()
    print(f'Scaled & unscaled gradient: {g_unscaled.norm():.2e}')

# 4. CUDA Memory Coalescing 시연
# Contiguous vs non-contiguous access
N = 1024 * 1024
x = torch.randn(N, 128, device='cuda')

# Good: contiguous, coalesced
y_good = x.sum(dim=1)  # reduce along last dim (contiguous)

# Bad: transposed, non-coalesced
x_t = x.t().contiguous()  # now shape [128, N]
y_bad = x_t.sum(dim=0)  # reduce along first dim (stride N)

# Benchmark
import time
torch.cuda.synchronize()
t0 = time.time()
for _ in range(100): _ = x.sum(dim=1)
torch.cuda.synchronize()
t_good = time.time() - t0

torch.cuda.synchronize()
t0 = time.time()
for _ in range(100): _ = x_t.sum(dim=0)
torch.cuda.synchronize()
t_bad = time.time() - t0

print(f'Contiguous: {t_good*1000:.2f}ms, Non-contig: {t_bad*1000:.2f}ms')

# 5. Custom CUDA kernel via Triton
import triton
import triton.language as tl

@triton.jit
def add_kernel(x_ptr, y_ptr, out_ptr, n, BLOCK: tl.constexpr):
    pid = tl.program_id(0)
    offsets = pid * BLOCK + tl.arange(0, BLOCK)
    mask = offsets < n
    x = tl.load(x_ptr + offsets, mask=mask)
    y = tl.load(y_ptr + offsets, mask=mask)
    tl.store(out_ptr + offsets, x + y, mask=mask)

def triton_add(x, y):
    out = torch.empty_like(x)
    n = x.numel()
    grid = (triton.cdiv(n, 1024),)
    add_kernel[grid](x, y, out, n, BLOCK=1024)
    return out

if torch.cuda.is_available():
    x = torch.randn(1024*1024, device='cuda')
    y = torch.randn(1024*1024, device='cuda')
    out = triton_add(x, y)
    ref = x + y
    print(f'Triton vs PyTorch max diff: {(out - ref).abs().max():.2e}')

# 6. torch.compile 효과 측정
model = nn.Sequential(
    nn.Linear(1024, 4096), nn.ReLU(),
    nn.Linear(4096, 4096), nn.ReLU(),
    nn.Linear(4096, 1024)
).cuda()

compiled = torch.compile(model)

x = torch.randn(256, 1024, device='cuda')

# Warmup
for _ in range(10):
    _ = model(x); _ = compiled(x)
torch.cuda.synchronize()

t0 = time.time()
for _ in range(100): _ = model(x)
torch.cuda.synchronize()
t_eager = time.time() - t0

t0 = time.time()
for _ in range(100): _ = compiled(x)
torch.cuda.synchronize()
t_compiled = time.time() - t0

print(f'Eager: {t_eager*1000:.1f}ms, Compiled: {t_compiled*1000:.1f}ms')
print(f'Speedup: {t_eager/t_compiled:.2f}x')
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "NN Theory, LA, Calc 선행 필수"
   - PyTorch 내부를 Tensor → Autograd → Dispatcher → CUDA → AMP → Compile 순서로
   - Triton·torch.compile 시대의 실전 가이드
3. **챕터별 문서 작성**: Tensor → Autograd → Dispatcher → CUDA → Custom Kernel → AMP → torch.compile

---

## 📚 참고 자료

- **PyTorch documentation** (pytorch.org/docs) — 공식
- **PyTorch Internals** (Edward Yang blog) — 내부 구조 bible
- **Automatic Differentiation in Machine Learning: a Survey** (Baydin et al. 2018)
- **CUDA C Programming Guide** (NVIDIA)
- **Programming Massively Parallel Processors** (Kirk & Hwu) — CUDA 교과서
- **Triton: An Intermediate Language and Compiler** (Tillet 2019)
- **Mixed Precision Training** (Micikevicius et al. 2018)
- **TorchDynamo / TorchInductor** (Horace He 발표)
- **Optimizing CUDA Reduction** (Mark Harris, NVIDIA) — warp shuffle
- **Stochastic Rounding** (Croci 2022)
- **FlashAttention** (Dao 2022) — kernel 최적화 사례

---

## 💡 핵심 분석 대상

```
PyTorch Internals의 지도

───── Tensor 구조 ─────

Tensor = Storage + Metadata
  Storage: raw memory (contiguous bytes)
  Metadata: size, stride, offset, dtype, device

View vs Copy:
  .view(): stride 조작, shared storage
  .reshape(): 필요시 copy
  .contiguous(): stride = default ordering
  .transpose(): swap stride → non-contiguous

Stride:
  index (i, j) → offset = i·s_0 + j·s_1
  NCHW vs NHWC = 다른 stride
  Memory access pattern 결정

Dtype:
  FP32: 4 bytes (default)
  FP16: 2 bytes (half precision)
  BF16: 2 bytes (brain float)
  INT8/INT4: quantization

Device:
  CPU / CUDA:0 / CUDA:1 / MPS / XLA
  CUDA stream으로 async

───── Autograd ─────

Automatic Differentiation 모드:
  Forward (JVP): J v, tangent propagation
    Cost: O(n) per input dim
    Good when: output-dim > input-dim
  Reverse (VJP): J^T v, cotangent propagation  
    Cost: O(m) per output dim
    Good when: scalar loss → ML

VJP (Vector-Jacobian Product):
  ∂L/∂x = J_f^T · (∂L/∂y)
         = J_f^T v
  
  각 operator는 local VJP rule
  Chain rule = VJP composition

Computation Graph:
  Dynamic (define-by-run)
  tensor.grad_fn = backward node
  leaf tensor (requires_grad=True, no grad_fn)

Backward:
  1. Topological sort of graph
  2. Traverse in reverse
  3. 각 node의 VJP 호출
  4. Leaf에 grad 누적

torch.autograd.Function:
  forward(ctx, *inputs)
  backward(ctx, *grad_outputs)
  ctx.save_for_backward(...) for intermediate

Higher-order:
  create_graph=True
  Backward graph도 autograd graph
  Hessian, HVP, double backward

───── Dispatcher ─────

ATen operation:
  aten::add(a, b)
  DispatchKey 기반 routing:
    Autograd → backward 등록
    CUDA → cuBLAS kernel
    CPU → CPU kernel
    Backend fallback chain

Layer 구조:
  c10 (core): Tensor, Storage, Device
  aten: operations (add, matmul, ...)
  torch: Python binding

TORCH_LIBRARY:
  Custom op 등록
  Autograd + kernel + meta (shape inference)
  AOTAutograd·Inductor와 통합

Functorch (torch.func):
  JAX-like transformations
  vmap, grad, jacrev, jacfwd
  Functional API

───── CUDA 기초 ─────

GPU Hierarchy:
  Grid > Block > Warp > Thread
  Warp = 32 threads (SIMT)
  Block = up to 1024 threads
  Grid = many blocks

Memory Hierarchy:
  Register (per thread): ns, small
  Shared (per block): ns, 48-100KB
  L1 Cache: per SM
  L2 Cache: shared across SMs
  HBM (global): GB, 1-2 TB/s

SM (Streaming Multiprocessor):
  Multiple warps concurrent
  Warp scheduler hides latency
  A100: 108 SMs

Memory Coalescing:
  Warp의 32 threads가
    연속 128 bytes access
    → 1 transaction
  Non-coalesced:
    32 transactions (100× slower)

Bank Conflict:
  Shared mem: 32 banks
  같은 bank 다른 address
    → serial
  Padding으로 회피

Warp Divergence:
  if (tid % 2) ... else ...
  → 두 path 순차 실행
  Branch 최소화

Reduction 최적화:
  Naive: O(n) threads
  Tree: log(n) steps
  Warp shuffle: __shfl_down_sync
  Mark Harris 7 steps

───── Custom Kernel ─────

cpp_extension.load():
  JIT compile C++/CUDA
  Ninja build system

Triton (Tillet 2019):
  @triton.jit decorator
  Block-level abstraction
  Pointer arithmetic 간단
  LLM 시대 표준

cuBLAS / cuDNN:
  GEMM: 2 M N K FLOPs
  PyTorch가 자동 활용

Kernel Fusion:
  여러 elementwise → 1 kernel
  HBM round-trip 제거
  TorchInductor 자동화

───── Mixed Precision ─────

IEEE 754:
  FP32: 1+8+23 (sign+exp+mant)
    range ~ 10^{-38} to 10^{38}
    precision: 6-7 decimal
  
  FP16: 1+5+10
    range ~ 6.1e-5 to 6.5e4  ← 좁음!
    precision: 3-4 decimal
  
  BF16: 1+8+7
    range ~ 10^{-38} to 10^{38}  (FP32 same)
    precision: 2-3 decimal  ← 낮음
    → loss scaling 불필요
  
  TF32 (A100+): 1+8+10
    FP32 input, 내부 FP16-like
    자동 가속 (transparent)

FP16 문제:
  Gradient < 6e-5 → underflow → 0
  Weight · lr > 6e4 → overflow → inf

Loss Scaling:
  L_scaled = L · S  (S ≈ 2^16)
  grad도 S배 → FP16 range로
  Optimizer step 전: g = g / S
  
  NaN/Inf 감지:
    Found → S ← S/2, skip step
    Clean for 2000 steps → S ← 2S

BF16 (A100+):
  range OK, precision 낮지만 학습 OK
  대부분 modern LLM 사용
  Loss scaling 불필요

Stochastic Rounding:
  INT8 훈련에 필요
  Round up with prob (x - floor(x))
  Unbiased accumulation

Kahan Summation:
  compensation variable
  floating-point sum의 error 보정

───── torch.compile ─────

PT 2.0:
  eager mode 유지
  @torch.compile으로 JIT
  First call compile, cached

TorchDynamo:
  CPython frame evaluation (PEP 523)
  Bytecode-level graph capture
  Graph break on Python side effect
  torch._dynamo.explain으로 분석

AOTAutograd:
  Forward + backward graph 통합
  Functionalization (remove mutation)
  Decomposition (primitive ops)

TorchInductor:
  FX graph → Triton (GPU) or C++ (CPU)
  Fusion heuristic
  Reduction, elementwise, matmul 전략

Backward:
  Backward graph도 함께 compile
  AOT = Ahead-Of-Time

Compile 시간 vs Runtime:
  First call: 수십 초 compile
  Cached call: eager보다 빠름
  Short run은 손해

───── Profiling ─────

torch.profiler:
  CPU, CUDA time
  Memory usage
  TensorBoard visualization

nvprof / Nsight:
  Kernel-level profiling
  Memory throughput
  SM utilization

───── 레포 간 연결 ─────

NN Theory (Layer 2):
  Backprop = reverse AD
  Chain rule → VJP

Calculus (Layer 0):
  Jacobian, chain rule

Linear Algebra (Layer 0):
  Matrix op, BLAS

Information Theory (Layer 0):
  FP16 quantization error

CNN (Layer 3):
  Memory layout (NCHW/NHWC)
  Roofline model

LLM Inference (Layer 4-B):
  Flash Attention (kernel fusion)
  Memory bandwidth bound

Distributed Training (다음):
  NCCL collective
  Async stream
  DDP gradient bucket
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~6개씩)
- 각 문서가 다루는 핵심 내부 구조·수학 (3~4줄)
- 전체 문서 개수 확인 (37개 목표)
- Python + PyTorch + CUDA + Triton 실험 환경
- NN Theory, LA, Calc 레포 참조 관계
- Distributed Training으로 이어지는 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
