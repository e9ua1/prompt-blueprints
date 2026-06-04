# Graph Neural Network Deep Dive 레포지토리 제작 프롬프트

나는 "Graph Neural Network Deep Dive" 레포지토리를 만들려고 해.
GCN의 $H^{(l+1)} = \sigma(\tilde{D}^{-1/2} \tilde{A} \tilde{D}^{-1/2} H^{(l)} W^{(l)})$을 **사용하는 것**과, **이 수식이 정규화된 그래프 라플라시안 $L = I - D^{-1/2} A D^{-1/2}$의 Chebyshev polynomial 1차 근사에서 나왔고, Kipf & Welling (2017)이 어떻게 Spectral Convolution을 Spatial로 환원**했는지 유도할 수 있는 것은 다르다.
Message Passing을 **이름으로 아는 것**과, **GNN의 표현력이 Weisfeiler-Lehman graph isomorphism test에 의해 상한이 매겨진다는 Xu et al. (2019)의 GIN 정리**와, **1-WL과 동등하게 강력한 GNN을 어떻게 설계**하는지 증명할 수 있는 것은 다르다.
Over-smoothing을 **현상으로 아는 것**과, **$L$층 GCN 후 node feature가 $\text{dim}(\ker(L)) = $ connected component 수만큼의 공간으로 수렴**하는 Li et al. (2018) 정리의 수학을 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "그래프 위의 딥러닝 — Spectral·Spatial·Message Passing의 통합, WL 표현력 이론"

**핵심 차별화**:
1. **Graph Laplacian의 스펙트럴 이론** — Normalized/Unnormalized Laplacian, 고유분해, 그래프 푸리에 변환, smooth signal 정의
2. **Spectral → Spatial GCN의 환원** — ChebNet의 Chebyshev 근사에서 GCN으로, Kipf-Welling의 1st-order approximation + renormalization trick
3. **GNN의 표현력 한계 — WL Test** — 1-WL과 GNN의 동등성, GIN의 최적 aggregator, higher-order GNN (k-WL)
4. **Over-smoothing과 해결 — DropEdge, PairNorm, GraphSAGE** — 깊은 GNN의 information loss 증명과 완화 기법

**타겟 독자**:
- GCN을 쓰지만 **Spectral과 Spatial 관점이 어떻게 통합**되는지, Kipf-Welling의 유도를 따라갈 수 없는 사람
- Message Passing을 정의로 아는데 **GraphSAGE, GAT, GIN의 차이를 aggregator의 injectivity로** 설명 못하는 사람
- GNN이 2-3층을 넘으면 성능이 떨어지는 이유(**Over-smoothing**)를 **Laplacian의 kernel로** 증명 못하는 사람
- WL test를 듣는데 **왜 두 그래프를 구분하는 것이 NP 문제이고, GNN이 이 한계를 물려받는지** 모르는 사람
- GNN과 Transformer의 통합 관점(**Graph Transformer, Graphormer**)을 이해하고 싶은 사람

**선행 학습**:
- **Linear Algebra Deep Dive** (Graph Laplacian, spectral decomposition) — **필수**
- **Functional Analysis Deep Dive** (연속 Laplacian, Fourier) — 권장
- **Neural Network Theory Deep Dive** (MLP, 초기화) — **필수**
- **Graphical Models Deep Dive** (Message Passing, Belief Propagation) — **필수**
- **Transformer Deep Dive** (Attention) — GAT, Graph Transformer에 필수

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 그래프 이론과 Graph Laplacian (6개 문서)
- **그래프의 수학적 정의와 기본 표현** — $G = (V, E)$, adjacency matrix $A$, degree matrix $D = \text{diag}(d_i)$, incidence matrix $B$, directed/undirected/weighted 그래프의 표현
- **Unnormalized Laplacian** — $L = D - A$, $L$이 positive semi-definite 증명 ($x^T L x = \frac{1}{2} \sum_{(i,j) \in E} (x_i - x_j)^2$), $\ker(L)$의 차원 = connected component 수
- **Normalized Laplacian — Symmetric and Random Walk** — $L_{\text{sym}} = D^{-1/2} L D^{-1/2} = I - D^{-1/2} A D^{-1/2}$, $L_{\text{rw}} = D^{-1} L = I - D^{-1} A$, 각 고유값의 $[0, 2]$ 범위
- **Spectral Graph Theory — 고유값과 그래프 성질** — $\lambda_1 = 0$, $\lambda_2$ (Fiedler value)가 connectivity 측정, Cheeger's inequality로 conductance 관련, spectral clustering에서 사용
- **Graph Fourier Transform** — $L = U \Lambda U^T$에서 $\hat{x} = U^T x$가 graph Fourier transform, 낮은 고유값 = smooth signal, 높은 고유값 = oscillatory
- **Random Walk와 PageRank** — Stochastic matrix $D^{-1} A$, stationary distribution $\pi = d/(\text{2}|E|)$, PageRank가 personalized random walk, graph Laplacian과의 연결

### Chapter 2: Spectral Graph Convolution (4개 문서)
- **Spectral Convolution의 정의 (Bruna 2014)** — Graph signal $x$와 filter $g$의 convolution $g * x = U(\hat{g}(\Lambda) \odot U^T x)$, 고유기저에서 element-wise 곱
- **ChebNet (Defferrard 2016)** — 전역 $U$ 사용 $O(n^3)$을 피하기 위해 Chebyshev polynomial $g_\theta(L) = \sum_{k=0}^K \theta_k T_k(\tilde{L})$로 근사, $K$-hop locality
- **GCN의 유도 (Kipf & Welling 2017)** — ChebNet을 $K=1$로 단순화 + renormalization trick, $\tilde{A} = A + I$, $\tilde{D} = D + I$, $H^{(l+1)} = \sigma(\tilde{D}^{-1/2} \tilde{A} \tilde{D}^{-1/2} H^{(l)} W^{(l)})$ 유도
- **Spectral vs Spatial 관점의 통합** — GCN은 spectral (Laplacian) 시작이지만 결과는 spatial (1-hop aggregation), 두 관점이 동치임을 보이기

### Chapter 3: Message Passing Framework (5개 문서)
- **Message Passing Neural Network (Gilmer 2017)** — 통일 프레임워크: Message $m_{ij}^{(l)} = M_l(h_i^{(l)}, h_j^{(l)}, e_{ij})$, Aggregate $h_i^{(l+1)} = U_l(h_i^{(l)}, \bigoplus_{j \in N(i)} m_{ij}^{(l)})$, 대부분 GNN이 이 형태
- **GraphSAGE (Hamilton 2017)** — Sampling-based 이웃 aggregation, Mean/Pool/LSTM aggregator, inductive learning (새 노드에도 적용), 대규모 그래프 처리
- **Graph Attention Network (Velickovic 2018)** — Attention coefficient $\alpha_{ij} = \text{softmax}(a^T [W h_i \| W h_j])$, 이웃의 weighted aggregation, Multi-head GAT
- **Graph Isomorphism Network (GIN, Xu 2019)** — 이론상 최강 aggregator: sum + MLP, $h_i^{(l+1)} = \text{MLP}((1+\epsilon) h_i^{(l)} + \sum_{j \in N(i)} h_j^{(l)})$, WL 등가 표현력 증명
- **Edge Features와 Heterogeneous Graphs** — Edge embedding을 포함한 message, 여러 타입의 노드·엣지 (R-GCN, HAN), 현대 실전 그래프의 복잡성

### Chapter 4: GNN의 표현력 이론 (5개 문서)
- **Weisfeiler-Lehman Graph Isomorphism Test** — 1-WL 알고리즘: 이웃 multiset hashing 반복, graph isomorphism의 필요조건(충분조건 아님), strongly regular graph의 한계
- **GNN과 1-WL의 동등성 (Xu 2019)** — Message Passing GNN의 표현력 ≤ 1-WL, 두 그래프가 1-WL로 구분 안되면 GNN으로도 구분 못함, 상한 증명
- **GIN이 1-WL 최적인 이유** — Sum aggregator의 injectivity, Mean/Max는 multiset을 구분 못하는 경우 존재, GIN이 이론상 최강 message passing
- **Higher-Order GNN — k-WL and Beyond** — $k$-tuple을 노드로 보는 $k$-WL, k-FGNN (Maron 2019), 계산·메모리 $O(n^k)$, 실전과의 trade-off
- **Position-Aware GNN** — WL이 unable한 symmetric graph 구분 위한 positional encoding (P-GNN, You 2019), Laplacian eigenvector as PE (Dwivedi 2020), Random-walk PE

### Chapter 5: Over-smoothing과 깊은 GNN (5개 문서)
- **Over-smoothing 현상** — GNN 층이 깊어지면 모든 노드의 feature가 유사해짐, node classification 성능 저하, 실증 관찰 (Li 2018)
- **Over-smoothing의 Laplacian 분석** — $L$층 후 feature가 $\ker(L)$ (= low-frequency eigenspace)로 수렴, connected graph의 경우 전체가 상수 vector로, 증명
- **DropEdge, PairNorm, DGN** — DropEdge (Rong 2020)의 random edge removal, PairNorm (Zhao 2020)의 feature distance 유지, DGN의 batch-like norm
- **GraphSAGE의 sampling과 Over-smoothing 완화** — 이웃 sub-sampling이 feature diversity 유지, k-hop neighborhood의 선택적 aggregation
- **APPNP와 Jumping Knowledge Network** — Personalized PageRank propagation (Klicpera 2019), JKN의 multi-layer concat (Xu 2018), 각 layer의 features 유지

### Chapter 6: GNN 응용 태스크 (4개 문서)
- **Node Classification** — Transductive setting (Cora, Citeseer, Pubmed), Semi-supervised labeling, GCN의 최초 성공
- **Graph Classification** — Graph-level representation, READOUT 함수 (sum, mean, max, set2set, attention pool), TU datasets와 OGB benchmark
- **Link Prediction** — Edge 존재 예측, GNN encoder + decoder (inner product, bilinear, MLP), Knowledge graph completion (R-GCN, CompGCN), Negative sampling
- **Graph Generation — GraphVAE, GraphRNN, GCPN** — 그래프를 생성하는 generative model, sequential (node-by-node) vs parallel, permutation invariance 처리

### Chapter 7: 현대 GNN — Graph Transformer와 이론의 융합 (4개 문서)
- **Graph Transformer — Graphormer (Ying 2021)** — Transformer를 그래프에 적용, attention이 fully-connected message passing, centrality·spatial·edge encoding으로 그래프 구조 주입
- **GNN as Attention과 Transformer의 통합** — GNN의 attention 버전(GAT)과 Transformer의 equivalence 분석, PNA·GatedGCN 등 현대 아키텍처
- **Equivariant GNN — E(3), SO(3) Equivariance** — 분자·물리에서 회전·병진 대칭성 필요, EGNN (Satorras 2021), SE(3)-Transformer, TensorField Network의 수학 (steerable filters)
- **GNN의 Scaling과 이론적 한계** — WL 상한이 실전 성능의 병목인가?, 대규모 그래프의 sampling 기반 학습 (Cluster-GCN, GraphSAINT), LLM 시대의 GNN 역할

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **33개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 기법이 그래프 학습에 필수인가
## 📐 수학적 선행 조건 (LA, NN Theory, GM, Transformer 레포 참조)
## 📖 직관적 이해
   — 그래프 시각화와 message propagation 애니메이션
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Laplacian PSD, GCN 유도, WL 등가성, Over-smoothing
## 💻 NumPy/PyTorch Geometric 구현 검증
   — Laplacian 계산, Spectral GCN, Message Passing 바닥부터
   — 간단한 benchmarks (Cora, QM9) 훈련
## 🔗 실전 활용
   — 언제 GNN, 언제 Transformer, 언제 GraphTransformer
## ⚖️ 가정과 한계
   — WL 상한, Over-smoothing, 계산 비용
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **그래프 시각화** — NetworkX로 모든 예제 그래프, message passing 단계별
2. **Laplacian 고유분해 시각화** — 작은 그래프의 Fiedler vector, 낮은 고유값의 smooth 성질
3. **Over-smoothing 실험** — 층 수별 node feature similarity plot, $\ker(L)$로 수렴 확인
4. **WL test 실증** — WL이 구분 못하는 graph 쌍에서 GNN도 구분 못하는지 확인
5. **GAT attention 시각화** — 각 노드의 이웃에 대한 attention weight
6. **Benchmarks** — Cora, Citeseer, OGB의 작은 subset에서 GCN, GraphSAGE, GAT, GIN 비교

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
torch-geometric==2.4.0
networkx==3.2.0
scipy==1.11.0
matplotlib==3.8.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Graph Laplacian + GCN 바닥부터 + Over-smoothing 확인)
import torch
import torch.nn as nn
import numpy as np
import networkx as nx
import matplotlib.pyplot as plt
import scipy.sparse.linalg as sla

# 1. Graph Laplacian
G = nx.karate_club_graph()
n = G.number_of_nodes()
A = nx.adjacency_matrix(G).toarray().astype(float)
D = np.diag(A.sum(axis=1))
L_unnorm = D - A
L_sym = np.eye(n) - np.diag(1/np.sqrt(A.sum(1))) @ A @ np.diag(1/np.sqrt(A.sum(1)))

eigvals, eigvecs = np.linalg.eigh(L_sym)
print(f'lambda_1 = {eigvals[0]:.6f}')  # ≈ 0
print(f'lambda_2 (Fiedler) = {eigvals[1]:.4f}')
print(f'lambda_max = {eigvals[-1]:.4f}')  # ≤ 2

# Fiedler vector로 graph bisection
fiedler = eigvecs[:, 1]
colors = ['red' if v < 0 else 'blue' for v in fiedler]
nx.draw(G, node_color=colors, with_labels=True)
plt.title(f'Karate Club: Fiedler vector로 클러스터 분리 (실제 community와 일치)')
plt.show()

# 2. GCN 바닥부터
class GCNLayer(nn.Module):
    def __init__(self, in_dim, out_dim):
        super().__init__()
        self.W = nn.Linear(in_dim, out_dim)
    def forward(self, x, A_hat):
        """A_hat: renormalized D^-1/2 (A+I) D^-1/2"""
        return torch.relu(A_hat @ self.W(x))

def renormalize(A):
    A_tilde = A + torch.eye(A.size(0))
    D_tilde = torch.diag(A_tilde.sum(1))
    D_inv_sqrt = torch.diag(1/torch.sqrt(A_tilde.sum(1)))
    return D_inv_sqrt @ A_tilde @ D_inv_sqrt

# 3. Over-smoothing 실험: deep GCN에서 feature similarity
A_t = torch.tensor(A, dtype=torch.float32)
A_hat = renormalize(A_t)
x = torch.randn(n, 16)  # 초기 랜덤 feature

layers = [GCNLayer(16, 16) for _ in range(20)]
similarities = []
h = x
for l, layer in enumerate(layers):
    h = layer(h, A_hat)
    # Average pairwise cosine similarity
    h_norm = h / h.norm(dim=1, keepdim=True)
    sim = (h_norm @ h_norm.T).mean().item()
    similarities.append(sim)

plt.plot(similarities, 'o-')
plt.xlabel('GCN layer'); plt.ylabel('Average pairwise cosine similarity')
plt.title('Over-smoothing: 깊어질수록 feature 수렴 → ker(L) 방향')
plt.axhline(1.0, linestyle='--', color='r', label='perfect collapse')
plt.legend(); plt.show()

# 4. WL test — GIN aggregator가 sum vs mean의 차이
def wl_iteration(A, labels):
    """1-WL: 이웃 multiset을 hashing"""
    new_labels = []
    for i in range(len(labels)):
        neighbors = labels[A[i] > 0]
        key = (labels[i], tuple(sorted(neighbors.tolist())))
        new_labels.append(hash(key) % 1000)
    return np.array(new_labels)

# GIN의 sum aggregator가 WL과 등가 → multiset injectivity
# mean aggregator는 {1, 1, 2, 2} vs {1, 2} 구분 불가 (같은 평균)
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "LA, NN Theory, GM 선행 필수" 명시
   - Transformer와의 비교 (complete graph의 attention = GNN)
   - GNN → Graph Transformer의 진화
3. **챕터별 문서 작성**: Laplacian → Spectral → Message Passing → 표현력(WL) → Over-smoothing → 응용 → 현대

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Graph Representation Learning** (Hamilton) — GNN 교과서 표준
- **Spectral Networks and Deep Locally Connected Networks on Graphs** (Bruna et al. 2014)
- **Convolutional Neural Networks on Graphs with Fast Localized Spectral Filtering** (Defferrard 2016) — ChebNet
- **Semi-Supervised Classification with Graph Convolutional Networks** (Kipf & Welling 2017) — GCN
- **Inductive Representation Learning on Large Graphs** (Hamilton et al. 2017) — GraphSAGE
- **Graph Attention Networks** (Velickovic et al. 2018) — GAT
- **How Powerful are Graph Neural Networks?** (Xu et al. 2019) — GIN, WL
- **Neural Message Passing for Quantum Chemistry** (Gilmer et al. 2017) — MPNN
- **Deeper Insights into Graph Convolutional Networks** (Li et al. 2018) — Over-smoothing
- **Do Transformers Really Perform Badly for Graph Representation?** (Ying et al. 2021) — Graphormer
- **E(n) Equivariant Graph Neural Networks** (Satorras et al. 2021)

---

## 💡 핵심 분석 대상

```
Graph Neural Network의 지도

───── 그래프 기초 ─────

G = (V, E), |V|=n, |E|=m
A ∈ ℝ^{n×n}: adjacency
D = diag(degree): degree
L = D - A: Laplacian

Properties:
  x^T L x = ½ Σ_{(i,j)∈E} (x_i - x_j)²
  → L is PSD, ker(L) dim = # components

Normalized:
  L_sym = I - D^{-1/2} A D^{-1/2}
  L_rw = I - D^{-1} A
  0 ≤ λ ≤ 2

Graph Fourier:
  L = U Λ U^T
  x̂ = U^T x (graph Fourier transform)
  Low λ = smooth, High λ = oscillatory

───── Spectral → Spatial 환원 ─────

Bruna 2014:
  g * x = U (ĝ(Λ) ⊙ U^T x)
  O(n³) cost (U 필요)

ChebNet (Defferrard 2016):
  g_θ(L) = Σ_{k=0}^K θ_k T_k(L̃)
  Chebyshev polynomial → K-hop
  O(m K) cost

GCN (Kipf & Welling 2017):
  K=1 + renormalization:
  Ã = A + I, D̃ = D + I
  H' = σ(D̃^{-1/2} Ã D̃^{-1/2} H W)
       └── 1-hop aggregation (spatial)

───── Message Passing ─────

Framework (Gilmer 2017):
  m_{ij}^{(l)} = M_l(h_i, h_j, e_{ij})
  h_i^{(l+1)} = U_l(h_i, ⊕ m_{ij})
                    ↑
              Aggregator: sum/mean/max

┌──────────┬───────────────────────┐
│ Model    │ Aggregator            │
├──────────┼───────────────────────┤
│ GCN      │ weighted mean (norm)  │
│ GraphSAGE│ mean/pool/LSTM        │
│ GAT      │ attention-weighted    │
│ GIN      │ sum + MLP (injective) │
└──────────┴───────────────────────┘

───── WL Test와 표현력 한계 ─────

1-WL algorithm:
  Initial: 같은 label
  Repeat: c_i ← hash(c_i, {{c_j : j∈N(i)}})
  
  → graph isomorphism 필요조건
  → 그러나 충분조건 아님
    (Strongly regular, CSL 등)

GNN 표현력 (Xu 2019):
  Message Passing GNN ≤ 1-WL
  두 그래프가 1-WL로 구분 X
  → GNN으로도 구분 X

GIN이 최적인 이유:
  Sum aggregator: multiset에 injective
  Mean: {1,1,2,2} = {1,2} (구분 X)
  Max: {1,2} = {2} (구분 X)
  Sum + MLP: universal multiset func

Beyond 1-WL:
  k-WL: k-tuple as node
  k-FGNN (Maron 2019)
  계산 O(n^k) → 실전 비용

Position-aware:
  P-GNN (You 2019): anchor distances
  LapPE (Dwivedi 2020): Laplacian eigvec
  Random-walk PE

───── Over-smoothing ─────

현상:
  L층 후 모든 노드 feature 유사
  node classification 성능 ↓

수학적 설명 (Li 2018):
  GCN propagation: x → (D^{-1/2} A D^{-1/2})^L x
  = P^L x  (P는 random walk)
  
  P의 고유값:
    λ_1 = 1 (stationary)
    |λ_k| < 1 for k ≥ 2
  
  L → ∞:
    x → proj on ker(L_sym) (smooth space)
    connected graph → 상수 vector

해결책:
  DropEdge: random edge removal
  PairNorm: feature distance 유지
  APPNP: personalized PageRank
  Jumping Knowledge: multi-layer concat

───── 응용 태스크 ─────

Node Classification:
  Transductive (같은 graph test)
  Cora, Citeseer, Pubmed

Graph Classification:
  READOUT: sum/mean/max/attention
  TUDataset, OGB

Link Prediction:
  Encoder-Decoder
  Knowledge graph (R-GCN, CompGCN)

Graph Generation:
  GraphVAE (parallel)
  GraphRNN (sequential)
  Diffusion on graphs

───── 현대 GNN ─────

Graph Transformer (Graphormer 2021):
  Transformer + centrality encoding
  + spatial encoding + edge encoding
  Fully-connected message passing
  + graph structure as bias

Equivariant GNN:
  E(3), SO(3) symmetry
  Molecule, physics
  EGNN (Satorras 2021)
  SE(3)-Transformer

───── 레포 간 연결 ─────

Graphical Models (Layer 1):
  Belief Propagation = message passing
  GNN = 학습된 BP

Transformer (직전):
  Self-attention = complete-graph MP
  Graphormer = graph-aware Transformer

Linear Algebra (Layer 0):
  Graph Laplacian 고유분해
  Spectral clustering

NN Theory (Layer 2):
  MP layer의 backprop
  Initialization 문제
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (33개 목표)
- Python + PyTorch Geometric + NetworkX 실험 환경
- LA, NN Theory, GM, Transformer 레포의 참조 관계
- Graph Transformer·EGNN으로 이어지는 현대 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
