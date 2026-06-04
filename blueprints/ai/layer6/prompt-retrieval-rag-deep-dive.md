# Retrieval & RAG Deep Dive 레포지토리 제작 프롬프트

나는 "Retrieval & RAG Deep Dive" 레포지토리를 만들려고 해.
DPR을 **"dense retrieval의 기본"으로 아는 것**과, **Karpukhin et al. (2020)의 bi-encoder $f_q(q), f_p(p)$에서 InfoNCE loss $L = -\log \frac{e^{s(q, p^+)/\tau}}{e^{s(q, p^+)/\tau} + \sum_{p^-} e^{s(q, p^-)/\tau}}$가 왜 BM25의 lexical matching 한계 (synonym, paraphrasing)를 극복**하고, **in-batch negatives와 hard negatives의 수학적 효과**를 유도할 수 있는 것은 다르다.
ColBERT를 **"late interaction"으로 아는 것**과, **Khattab & Zaharia (2020)의 MaxSim score $S_{q,d} = \sum_{i \in |q|} \max_{j \in |d|} E_{q_i} \cdot E_{d_j}$가 bi-encoder의 single-vector 한계를 극복**하고, **query-document의 token-level fine-grained matching을 보장하면서 offline indexing으로 scalability를 유지**하는 trade-off를 이해하는 것은 다르다.
HNSW를 **"vector search 알고리즘"으로 아는 것**과, **Malkov & Yashunin (2018)의 multi-layer small-world graph가 각 layer마다 $m$ neighbors로 $O(\log N)$ greedy search를 보장**하고, **layer 선택이 exponentially decaying probability** $P(l) = \exp(-l/\ln(m_L))$로 skip-list 구조를 형성한다는 수학을 유도할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "정보 검색의 수학 — BM25부터 Dense Retrieval·ANN·Hybrid·GraphRAG까지"

**핵심 차별화**:
1. **IR 기초의 정보이론** — TF-IDF, BM25 (Robertson 1995)의 Probabilistic Relevance Framework, evaluation metric (NDCG, MAP, MRR)의 수학
2. **Dense Retrieval의 contrastive learning** — DPR의 InfoNCE, SBERT (Reimers 2019), contriever (Izacard 2022)의 unsupervised training
3. **Late Interaction (ColBERT)** — Single-vector vs multi-vector trade-off, MaxSim의 expressiveness, memory-throughput 분석
4. **Approximate Nearest Neighbor 알고리즘** — HNSW 그래프 이론, IVF-PQ quantization, LSH, FAISS/ScaNN의 내부
5. **RAG Architectures** — Vanilla RAG, RETRO chunked cross-attn, Self-RAG·CRAG adaptive retrieval, GraphRAG의 community detection

**타겟 독자**:
- RAG를 구현하는데 **BM25 + dense hybrid retrieval**이 왜 각각보다 좋은지 RRF 수학으로 설명 못하는 사람
- ColBERT의 **MaxSim이 왜 bi-encoder보다 expressive**하고 **cross-encoder보다 efficient**한지 trade-off를 모르는 사람
- HNSW의 **$M$ neighbors·$ef$ parameter**가 recall-latency trade-off에 어떻게 영향을 주는지 모르는 사람
- IVF-PQ에서 **product quantization이 $2^{m \cdot \log k}$ centroid**를 $k \cdot m$만으로 approximate하는 수학을 모르는 사람
- Self-RAG의 **reflection tokens ([Retrieve], [IsREL], [IsSUP], [IsUSE])**이 어떻게 adaptive retrieval을 구현하는지 모르는 사람

**선행 학습**:
- **Linear Algebra Deep Dive** (vector distance, norm) — **필수**
- **Probability Theory Deep Dive** (probabilistic IR) — **필수**
- **Information Theory Deep Dive** (entropy, MI) — **필수**
- **Transformer Deep Dive** (bi-encoder, cross-encoder) — **필수**
- **LLM Pretraining Deep Dive** (context, tokenizer) — 권장
- **Kernel Methods Deep Dive** (similarity, inner product) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: IR Foundations와 평가 (5개 문서)
- **Information Retrieval의 정식화** — Query $q$, document collection $\mathcal{D}$, relevance function $r(q, d)$, ranking task, top-$k$ retrieval
- **TF-IDF와 Vector Space Model** — $\text{tf-idf}(t, d) = \text{tf}(t, d) \cdot \log \frac{N}{\text{df}(t)}$, document를 $|V|$-dim sparse vector로, cosine similarity
- **BM25 — Probabilistic Relevance Framework (Robertson 1995)** — $\text{BM25}(q, d) = \sum_{t \in q} \text{IDF}(t) \cdot \frac{(k_1 + 1) \text{tf}(t, d)}{k_1((1-b) + b \frac{|d|}{\text{avgdl}}) + \text{tf}(t, d)}$, $k_1, b$ 튜닝, 여전히 strong baseline
- **평가 Metrics — Recall@k, MRR, NDCG, MAP** — $\text{NDCG}@k = \text{DCG}@k / \text{IDCG}@k$, graded relevance, position discount, 각 metric의 정보이론적 의미
- **Retrieval vs Ranking Pipeline** — Retrieval (high recall, efficient) → Reranking (high precision, expensive), two-stage 설계의 근거

### Chapter 2: Dense Retrieval — DPR와 Bi-Encoder (5개 문서)
- **BM25의 한계와 Dense Retrieval 동기** — Lexical mismatch ("car" vs "automobile"), multilingual, semantic similarity 부족, 동음이의어
- **Bi-Encoder Architecture (DPR, Karpukhin 2020)** — Query encoder $f_q$ + Passage encoder $f_p$ (독립 BERT), $s(q, p) = f_q(q)^T f_p(p)$, offline indexing 가능
- **InfoNCE Loss와 In-Batch Negatives** — $L = -\log \frac{e^{s(q, p^+)/\tau}}{\sum_p e^{s(q, p)/\tau}}$, batch 내 다른 positive를 negative로 재활용, $O(B^2)$ pairs에서 학습
- **Hard Negative Mining** — BM25-retrieved negatives, model-mined negatives (ANCE, Xiong 2021), 학습 효율성, hard negative의 수학적 중요성
- **SBERT, Contriever, E5 — Unsupervised Dense** — SBERT (Reimers 2019)의 sentence-level, Contriever (Izacard 2022)의 unsupervised contrastive, E5 (Wang 2022)의 weakly-supervised, 현대 embedding model 계보

### Chapter 3: Late Interaction과 Cross-Encoder (4개 문서)
- **Cross-Encoder — Full Interaction** — $(q, d)$를 함께 Transformer에 입력, $\text{[CLS]}$ embedding으로 score, 최고 성능 but $O(N)$ inference, reranker에만 사용
- **ColBERT — Late Interaction (Khattab 2020)** — Per-token embedding $E_q \in \mathbb{R}^{|q| \times d}, E_d \in \mathbb{R}^{|d| \times d}$, MaxSim $S_{q,d} = \sum_{i} \max_j E_{q_i} \cdot E_{d_j}$
- **ColBERTv2 — PLAID Engine (Santhanam 2022)** — Centroid-based compression (k-means + residual), quantization, 2.6× smaller index, faster retrieval
- **Multi-Vector vs Single-Vector Trade-off** — Expressiveness (multi) vs memory (single), inference cost comparison, 언제 어느 것을 선택

### Chapter 4: Approximate Nearest Neighbor Algorithms (6개 문서)
- **Exact NN의 한계** — $O(N \cdot d)$ linear scan, 1M+ vectors에서 비현실적, GPU로도 latency 제약, approximate 필요성
- **LSH — Locality Sensitive Hashing (Indyk & Motwani 1998)** — Random projection $h(x) = \text{sign}(w^T x)$, hash collision probability $P(\text{same bucket}) \propto$ similarity, multi-hash for recall
- **IVF — Inverted File Index** — $k$-means로 $N$개 centroid, query는 가장 가까운 $\text{nprobe}$개 cluster만 검색, $O(N/K)$ search (with $K$ centroids)
- **PQ — Product Quantization (Jégou 2011)** — Vector를 $m$개 subvector로 split, 각 subvector에 $k$ centroids ($2^8 = 256$ typical), 총 $k^m$ distinct codes with only $m \log_2 k$ bits
- **HNSW — Hierarchical Navigable Small World (Malkov 2018)** — Multi-layer graph, 각 layer는 sparser, top-down greedy search, $O(\log N)$ complexity, layer assignment $P(l) = \exp(-l \cdot \ln(m_L))$
- **FAISS, ScaNN, Qdrant, Milvus — Vector DB 내부** — FAISS (Meta)의 IVF-PQ·HNSW 조합, ScaNN (Google)의 anisotropic quantization, Qdrant의 filtering+HNSW

### Chapter 5: RAG Architectures (6개 문서)
- **Vanilla RAG (Lewis 2020)** — Retrieve → Augment → Generate, top-$k$ passages concat to context, simple but effective baseline
- **RETRO — Chunked Cross-Attention (Borgeaud 2022)** — 2T token database, chunk-level retrieval, $\text{CCA}(H, E_{\text{ret}})$ cross-attention layer, 25× smaller model 동등 성능
- **REALM, Atlas — End-to-End Trained RAG** — Retriever와 generator를 joint training, REALM (Guu 2020)의 inverse-cloze task pretraining, Atlas (Izacard 2022)의 few-shot
- **Self-RAG (Asai 2024)** — Reflection tokens ([Retrieve], [IsREL], [IsSUP], [IsUSE]), adaptive retrieval (필요할 때만), self-evaluation of outputs
- **CRAG — Corrective RAG (Yan 2024)** — Retrieval evaluator가 confidence 평가, web search fallback, knowledge refinement
- **FiD — Fusion-in-Decoder (Izacard 2021)** — $k$개 passage를 각각 encode, decoder에서 cross-attention으로 통합, long context efficient

### Chapter 6: Reranking과 Hybrid Retrieval (4개 문서)
- **Cross-Encoder Reranker** — MonoBERT, MonoT5 (Nogueira 2020), two-stage pipeline의 rerank 단계, 상위 $k$ candidates 재정렬
- **RRF — Reciprocal Rank Fusion (Cormack 2009)** — Multiple rankers 결합 $\text{RRF}(d) = \sum_{r \in R} \frac{1}{k + \text{rank}_r(d)}$, BM25+dense hybrid에 주로 사용, 단순하지만 강력
- **Hybrid BM25+Dense Retrieval** — Lexical (BM25) + semantic (dense)의 complementary 특성, exact match와 paraphrase 모두 처리, SPLADE (Formal 2021)의 sparse-dense 통합
- **LLM-as-Reranker** — RankGPT (Sun 2023), LLM의 listwise ranking capability, zero-shot vs fine-tuned, 고비용이지만 최고 품질

### Chapter 7: Advanced — GraphRAG, Multimodal, Frontier (3개 문서)
- **GraphRAG (Edge 2024)** — Knowledge graph 구축 + community detection (Leiden algorithm), hierarchical summarization, global question answering, LLM-generated graph
- **ColPali와 Vision-RAG (Faysse 2024)** — Document page를 이미지로 encode (VLM), late interaction, PDF·scanned document retrieval without OCR
- **Frontier — Long Context vs RAG, Late Chunking** — 1M+ context length에서 RAG 필요성, late chunking (문서 전체 embedding 후 chunk), long-context reranking

---

각 챕터는 **3~6개 문서**로 구성해줘. 총 **33개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 기법이 retrieval·RAG에 중요한가
## 📐 수학적 선행 조건 (LA, Prob, Info, Transformer 참조)
## 📖 직관적 이해
   — Vector space, graph, inverted index 시각화
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — BM25 PRF, InfoNCE, HNSW complexity, ColBERT MaxSim
## 💻 Python/PyTorch 구현 검증
   — BM25 바닥부터
   — DPR contrastive training
   — HNSW 간단 구현
   — RAG pipeline 구축
## 🔗 실전 활용
   — Production RAG system, eval, cost
## ⚖️ 가정과 한계
   — Domain mismatch, query distribution
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **BM25 vs TF-IDF** — 작은 corpus에서 score 계산 비교
2. **Dense vs Sparse 시각화** — t-SNE embedding, BM25 score distribution
3. **HNSW graph 시각화** — Multi-layer small-world structure
4. **IVF-PQ 분해** — 각 subvector centroid, quantization error
5. **RAG pipeline** — BM25 → dense → rerank → LLM
6. **Eval** — MS MARCO, BEIR benchmark의 각 방법 성능

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
transformers==4.36.0
sentence-transformers==2.3.0   # SBERT
faiss-cpu==1.7.4               # FAISS
rank-bm25==0.2.2               # BM25
langchain==0.1.0
langchain-community==0.0.13
llama-index==0.9.40
networkx==3.2                  # GraphRAG
matplotlib==3.8.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (BM25 + DPR + HNSW + RAG 바닥부터)
import numpy as np
from collections import Counter
import torch
import torch.nn as nn
import torch.nn.functional as F

# 1. BM25 바닥부터
class BM25:
    def __init__(self, corpus, k1=1.5, b=0.75):
        self.corpus = corpus
        self.k1 = k1; self.b = b
        self.N = len(corpus)
        self.doc_lens = [len(d) for d in corpus]
        self.avgdl = np.mean(self.doc_lens)
        
        # Term frequencies per doc
        self.tfs = [Counter(d) for d in corpus]
        # Document frequency
        self.df = Counter()
        for doc in corpus:
            for term in set(doc):
                self.df[term] += 1
    
    def idf(self, term):
        """IDF(t) = log((N - df + 0.5) / (df + 0.5))"""
        df = self.df.get(term, 0)
        return np.log((self.N - df + 0.5) / (df + 0.5) + 1)
    
    def score(self, query, doc_idx):
        """BM25 score"""
        score = 0
        for term in query:
            tf = self.tfs[doc_idx].get(term, 0)
            if tf == 0: continue
            idf = self.idf(term)
            dl = self.doc_lens[doc_idx]
            num = tf * (self.k1 + 1)
            den = tf + self.k1 * (1 - self.b + self.b * dl / self.avgdl)
            score += idf * num / den
        return score
    
    def search(self, query, top_k=10):
        scores = [(i, self.score(query, i)) for i in range(self.N)]
        return sorted(scores, key=lambda x: -x[1])[:top_k]

# 2. DPR — Bi-Encoder Contrastive
class BiEncoder(nn.Module):
    def __init__(self, model_name='bert-base-uncased'):
        super().__init__()
        # Shared or separate encoders (DPR uses separate)
        from transformers import AutoModel
        self.q_encoder = AutoModel.from_pretrained(model_name)
        self.p_encoder = AutoModel.from_pretrained(model_name)
    
    def encode_query(self, q_ids, q_mask):
        out = self.q_encoder(q_ids, attention_mask=q_mask)
        return out.last_hidden_state[:, 0]  # [CLS]
    
    def encode_passage(self, p_ids, p_mask):
        out = self.p_encoder(p_ids, attention_mask=p_mask)
        return out.last_hidden_state[:, 0]

def dpr_loss(q_emb, p_emb_pos, p_emb_neg, tau=1.0):
    """
    InfoNCE with in-batch negatives
    q_emb: [B, D]
    p_emb_pos: [B, D] positive passages
    p_emb_neg: [B, K, D] hard negatives (optional)
    
    In-batch negatives: other q's positive serves as negative
    """
    B = q_emb.size(0)
    # Score matrix: [B, B]
    scores_pos = q_emb @ p_emb_pos.T / tau  # diagonal = positive
    
    if p_emb_neg is not None:
        # Concatenate hard negatives per query
        scores_neg = torch.einsum('bd,bkd->bk', q_emb, p_emb_neg) / tau
        scores = torch.cat([scores_pos, scores_neg], dim=1)
        labels = torch.arange(B)  # positive at index i for query i
    else:
        scores = scores_pos
        labels = torch.arange(B)
    
    return F.cross_entropy(scores, labels)

# 3. ColBERT — Late Interaction MaxSim
def colbert_maxsim(E_q, E_d):
    """
    E_q: [|q|, d] query token embeddings
    E_d: [|d|, d] doc token embeddings
    
    MaxSim: S = Σ_i max_j (E_qi · E_dj)
    """
    # [|q|, |d|] similarity matrix
    sim = E_q @ E_d.T  # assume normalized
    # Max over doc tokens for each query token
    max_sim = sim.max(dim=-1).values  # [|q|]
    # Sum over query tokens
    return max_sim.sum()

# 4. HNSW — 간단 구현 (single-layer greedy)
class SimpleHNSW:
    def __init__(self, M=16, ef_construction=200):
        self.M = M  # max neighbors per node
        self.ef = ef_construction
        self.graph = {}  # node_id -> list of (neighbor_id, distance)
        self.data = {}  # node_id -> vector
    
    def distance(self, v1, v2):
        return np.linalg.norm(v1 - v2)
    
    def insert(self, node_id, vec):
        self.data[node_id] = vec
        self.graph[node_id] = []
        
        if len(self.data) == 1:
            return
        
        # Find nearest existing neighbors
        candidates = []
        for other_id in self.data:
            if other_id == node_id: continue
            d = self.distance(vec, self.data[other_id])
            candidates.append((other_id, d))
        candidates.sort(key=lambda x: x[1])
        
        # Connect to top M nearest
        for neighbor_id, d in candidates[:self.M]:
            self.graph[node_id].append((neighbor_id, d))
            # Add reverse edge (up to M)
            self.graph[neighbor_id].append((node_id, d))
            self.graph[neighbor_id].sort(key=lambda x: x[1])
            self.graph[neighbor_id] = self.graph[neighbor_id][:self.M]
    
    def search(self, query, k=10, entry_point=None):
        """Greedy best-first search"""
        if not self.data: return []
        if entry_point is None:
            entry_point = next(iter(self.data))
        
        visited = {entry_point}
        candidates = [(self.distance(query, self.data[entry_point]), entry_point)]
        results = []
        
        while candidates:
            candidates.sort()
            curr_d, curr = candidates.pop(0)
            if len(results) >= k and curr_d > results[-1][0]:
                break
            results.append((curr_d, curr))
            results.sort()
            results = results[:k]
            
            # Explore neighbors
            for neighbor_id, _ in self.graph[curr]:
                if neighbor_id in visited: continue
                visited.add(neighbor_id)
                d = self.distance(query, self.data[neighbor_id])
                candidates.append((d, neighbor_id))
        
        return results

# 5. Product Quantization
class ProductQuantizer:
    def __init__(self, m=8, k=256):
        """
        m: 개의 subvector로 split
        k: 각 subvector마다 centroid 수
        """
        self.m = m; self.k = k
        self.centroids = None
    
    def fit(self, X):
        """X: [N, D], D must be divisible by m"""
        from sklearn.cluster import KMeans
        N, D = X.shape
        assert D % self.m == 0
        sub_D = D // self.m
        
        self.centroids = []
        for i in range(self.m):
            X_sub = X[:, i*sub_D:(i+1)*sub_D]
            km = KMeans(n_clusters=self.k, n_init=10).fit(X_sub)
            self.centroids.append(km.cluster_centers_)
    
    def encode(self, X):
        """X: [N, D] → codes [N, m] each in [0, k)"""
        N, D = X.shape
        sub_D = D // self.m
        codes = np.zeros((N, self.m), dtype=np.int32)
        for i in range(self.m):
            X_sub = X[:, i*sub_D:(i+1)*sub_D]
            dists = ((X_sub[:, None] - self.centroids[i][None])**2).sum(-1)
            codes[:, i] = dists.argmin(-1)
        return codes
    
    def compression_ratio(self, orig_bits=32):
        """Original: D × 32 bits. PQ: m × log2(k) bits"""
        return (self.m * np.log2(self.k)) / (orig_bits)

# 6. RAG pipeline
class VanillaRAG:
    def __init__(self, retriever, llm):
        self.retriever = retriever
        self.llm = llm
    
    def answer(self, question, k=5):
        # 1. Retrieve top-k passages
        passages = self.retriever.search(question, top_k=k)
        # 2. Format context
        context = "\n\n".join([f"[{i+1}] {p}" for i, p in enumerate(passages)])
        prompt = f"Answer the question based on the context.\n\nContext:\n{context}\n\nQuestion: {question}\n\nAnswer:"
        # 3. Generate
        return self.llm.generate(prompt)

# 7. Reciprocal Rank Fusion
def rrf(rankings, k=60):
    """
    rankings: list of lists, each [(doc_id, rank), ...]
    RRF(d) = Σ 1 / (k + rank_r(d))
    """
    scores = {}
    for ranking in rankings:
        for rank, doc_id in enumerate(ranking):
            scores[doc_id] = scores.get(doc_id, 0) + 1 / (k + rank + 1)
    return sorted(scores.items(), key=lambda x: -x[1])
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "LA, Prob, Info, Transformer 선행 필수" 명시
   - BM25 → Dense → ANN → RAG → GraphRAG 진화
   - MS MARCO, BEIR, LoTTE benchmark
3. **챕터별 문서 작성**: IR Foundations → DPR → ColBERT → ANN → RAG → Reranking → Frontier

---

## 📚 참고 자료

- **Introduction to Information Retrieval** (Manning, Raghavan, Schütze) — 교과서
- **The Probabilistic Relevance Framework: BM25 and Beyond** (Robertson & Zaragoza 2009)
- **Dense Passage Retrieval** (Karpukhin et al. 2020) — DPR
- **Sentence-BERT** (Reimers & Gurevych 2019)
- **Unsupervised Dense Information Retrieval with Contrastive Learning** (Izacard et al. 2022) — Contriever
- **ColBERT** (Khattab & Zaharia 2020)
- **ColBERTv2 / PLAID** (Santhanam et al. 2022)
- **Approximate Nearest Neighbor Search — HNSW** (Malkov & Yashunin 2018)
- **Product Quantization for Nearest Neighbor Search** (Jégou et al. 2011)
- **Approximate Nearest Neighbors — LSH** (Indyk & Motwani 1998)
- **Billion-scale similarity search with GPUs** (Johnson et al. 2019) — FAISS
- **ScaNN: Efficient Vector Similarity Search** (Guo et al. 2020)
- **RAG — Retrieval-Augmented Generation** (Lewis et al. 2020)
- **RETRO** (Borgeaud et al. 2022)
- **REALM** (Guu et al. 2020)
- **Atlas** (Izacard et al. 2022)
- **Self-RAG** (Asai et al. 2024)
- **CRAG — Corrective RAG** (Yan et al. 2024)
- **FiD** (Izacard & Grave 2021)
- **MonoT5, RankGPT** (Nogueira et al. 2020, Sun et al. 2023)
- **RRF — Reciprocal Rank Fusion** (Cormack et al. 2009)
- **SPLADE** (Formal et al. 2021)
- **GraphRAG** (Edge et al. 2024) — Microsoft
- **ColPali** (Faysse et al. 2024)
- **ANCE** (Xiong et al. 2021) — Hard negative mining
- **E5 embeddings** (Wang et al. 2022)

---

## 💡 핵심 분석 대상

```
Retrieval & RAG의 지도

───── IR Foundations ─────

Task formalization:
  Query q, collection D
  r(q, d) = relevance
  Top-k ranking

TF-IDF:
  tf-idf(t, d) = tf(t,d) · log(N/df(t))
  Vector space model
  Cosine similarity

BM25 (Robertson 1995):
  score = Σ_t IDF(t) · 
          (k_1 + 1) tf(t,d) / 
          (k_1((1-b) + b |d|/avgdl) + tf(t,d))
  
  k_1 ≈ 1.2-2.0
  b ≈ 0.75
  
  Probabilistic Relevance Framework
  여전히 strong baseline

Metrics:
  Recall@k: retrieved relevant / all relevant
  Precision@k: retrieved relevant / k
  MRR = 1/|Q| Σ 1/rank(first relevant)
  MAP = mean of AP
  NDCG@k = DCG@k / IDCG@k
    DCG = Σ rel_i / log2(i+1)
  
Two-stage:
  Retrieve (high recall, fast)
  Rerank (high precision, slow)

───── Dense Retrieval ─────

BM25의 한계:
  Lexical mismatch
  Synonym
  Multilingual
  → semantic 필요

DPR (Karpukhin 2020):
  Bi-encoder:
    f_q(q), f_p(p) 독립 BERT
    s(q, p) = f_q(q)^T f_p(p)
  
  Offline indexing:
    모든 p를 미리 encode
    Query time: f_q(q) + NN search

InfoNCE Loss:
  L = -log exp(s(q, p^+)/τ) / 
         Σ exp(s(q, p)/τ)
  
  In-batch negatives:
    batch의 다른 positive = negative
    O(B²) pairs from B samples
    효율적

Hard Negatives:
  Random negatives 쉬움 → 학습 효과 낮음
  BM25-mined: BM25 high score but not relevant
  Model-mined (ANCE): model이 헷갈리는
  → 학습 신호 강화

SBERT (Reimers 2019):
  Sentence-level embedding
  Siamese BERT
  Classification·similarity

Contriever (Izacard 2022):
  Unsupervised
  Random cropping 같은 문서 positives
  MoCo-style training

E5 (Wang 2022):
  Weakly-supervised
  Natural pairs from web (title-body, etc.)
  Instruct: query prefix "query: "

───── ColBERT ─────

Cross-Encoder:
  [CLS] q [SEP] d [SEP] → BERT
  [CLS] → score
  Full interaction
  BEST quality
  BUT: O(N) inference
  → rerank only

ColBERT (Khattab 2020):
  Per-token embedding:
    E_q = BERT(q)[tokens]  [|q|, d]
    E_d = BERT(d)[tokens]  [|d|, d]
  
  MaxSim:
    S(q, d) = Σ_i max_j (E_qi · E_dj)
  
  각 query token이 document에서 best match
  Sum of per-token matches
  
  Offline:
    모든 E_d 미리 계산, storage
  
  Online:
    E_q 계산 후 MaxSim search

ColBERTv2 (PLAID 2022):
  Centroid compression:
    E_d ≈ centroid + residual
  Quantization
  2.6× smaller index
  Practical deployment

Trade-off:
  Single-vector (dense):
    Memory efficient
    Fast
    Expressiveness limited
  
  Multi-vector (ColBERT):
    Higher quality
    More memory (|d| × d)
    Still efficient
  
  Cross-encoder:
    Best quality
    O(N) online
    Rerank only

───── ANN Algorithms ─────

Exact NN:
  O(N · d) linear scan
  1M vectors × 768d = 3GB scan
  Latency bottleneck

LSH (Indyk 1998):
  Random projection h(x) = sign(w^T x)
  P(same bucket) ∝ similarity
  Multiple hash functions for recall
  Simple but lower accuracy

IVF (Inverted File):
  k-means로 K centroids
  Query: 가까운 nprobe cluster만 검색
  O(N/K) search
  Recall-speed trade-off (nprobe)

PQ (Jégou 2011):
  Vector를 m sub로 split
  각 sub에 k centroids (보통 256)
  
  Code: m bytes (m × log2(256))
  vs original: m × D/m × 32 bits
  → 32-64× compression
  
  Distance approximation:
    Σ_i d(sub_i, centroid_q_i)

IVF-PQ:
  IVF로 coarse, PQ로 fine
  FAISS의 주력
  GPU 가능

HNSW (Malkov 2018):
  Multi-layer graph
  Top layer: sparse, global
  Bottom: dense, local
  
  Layer assignment:
    P(l) = exp(-l · ln(m_L))
    → skip-list 구조
  
  Greedy search:
    Top에서 시작
    최근접 neighbor로 이동
    더 이상 가까워지지 않으면 ↓
  
  Complexity: O(log N)
  
  Parameters:
    M: max neighbors (16-64)
    ef: candidate list (64-512)
    
  Memory: O(N · M · 4 bytes)
  (연결 + 원본 벡터)

FAISS (Meta):
  IVF-PQ, HNSW, IVF-HNSW
  CPU + GPU
  Billion-scale

ScaNN (Google):
  Anisotropic quantization
  더 나은 recall at same latency

Qdrant / Milvus / Weaviate:
  Full vector DB
  Filtering + HNSW
  Payload indexing

───── RAG Architectures ─────

Vanilla RAG (Lewis 2020):
  Q → retrieve top-k → concat → LLM
  Simple baseline
  현업 대부분이 이 형태

RETRO (Borgeaud 2022):
  2T token database
  Chunk retrieval (64 tokens)
  Chunked Cross-Attention:
    Generator의 각 chunk가
    retrieved chunks에 attend
  
  25× smaller model 동등 성능
  Frozen retriever

REALM (Guu 2020):
  End-to-end trained
  Inverse-cloze pretraining
  MLM task로 retriever도 학습

Atlas (Izacard 2022):
  Few-shot learning
  Retrieval + language model
  Efficient knowledge task

Self-RAG (Asai 2024):
  Reflection tokens:
    [Retrieve] - 필요할 때만
    [IsREL] - 관련성
    [IsSUP] - 지원 여부
    [IsUSE] - 유용성
  
  Adaptive retrieval
  Self-evaluation
  Not always retrieve

CRAG (Yan 2024):
  Retrieval evaluator
  High confidence → use
  Low confidence → web search fallback
  Knowledge refinement

FiD (Izacard 2021):
  k passages each encoded independently
  Decoder cross-attends to all
  Long context efficient

───── Reranking ─────

Cross-encoder reranker:
  MonoBERT, MonoT5
  (q, d) → score
  Top-k 재정렬

RRF (Cormack 2009):
  Multiple rankers 결합:
    RRF(d) = Σ_r 1/(k + rank_r(d))
  k = 60 typical
  BM25 + dense hybrid에 주효

Hybrid BM25+Dense:
  Lexical (exact match) + semantic
  Complementary
  SPLADE (Formal 2021):
    Sparse neural (BERT + MLM head)
    BM25-like output, dense quality

LLM-as-Reranker:
  RankGPT (Sun 2023):
    Listwise ranking by LLM
    Zero-shot or fine-tuned
  GPT-4 급 reranker
  비용 비싸지만 최고 품질

───── Advanced ─────

GraphRAG (Edge 2024):
  LLM이 entity-relation graph 구축
  Community detection (Leiden)
  Hierarchical summarization
  Global question answering
  Microsoft 발표, open-source

ColPali (Faysse 2024):
  Document page = image
  VLM encode (PaliGemma)
  Late interaction like ColBERT
  PDF, scanned document
  OCR 불필요

Long Context vs RAG:
  1M+ context 가능
  Lost in the middle 문제
  Needle-in-haystack
  RAG는 여전히 선별 필수

Late Chunking:
  긴 문서 전체 embedding
  그 다음 chunk로 slice
  Cross-chunk context 보존
  Jina AI 제안

───── 레포 간 연결 ─────

Linear Algebra (Layer 0):
  Vector similarity
  Cosine, L2
  Inner product

Probability (Layer 0):
  Probabilistic IR (BM25)
  Binary independence model

Information Theory (Layer 0):
  Entropy, MI
  Query-doc MI

Transformer (Layer 3):
  Bi-encoder, cross-encoder
  ColBERT backbone

LLM Pretraining (Layer 4-B):
  Context window
  Tokenizer

Kernel Methods (Layer 1):
  Similarity as kernel
  Inner product

Pretrained LM (Layer 4-D):
  BERT for DPR
  Sentence encoder

LLM Reasoning (직전):
  Agent의 tool use
  RAG가 knowledge tool
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (3~6개씩)
- 각 문서가 다루는 핵심 정리·알고리즘 (3~4줄)
- 전체 문서 개수 확인 (33개 목표)
- Python + FAISS + sentence-transformers + LangChain 실험 환경
- LA, Prob, Info, Transformer 레포 참조 관계
- MS MARCO, BEIR 벤치마크 검증

**준비됐으면 1단계 구조 설계부터 시작해줘!**
