# NLP Foundations Deep Dive 레포지토리 제작 프롬프트

나는 "NLP Foundations Deep Dive" 레포지토리를 만들려고 해.
Word2Vec의 Skip-gram을 **사용하는 것**과, **Mikolov et al. (2013)의 목적함수 $\max \sum_c \sum_{-m \leq j \leq m, j \neq 0} \log p(w_{c+j} | w_c)$에서 full softmax $p(w_O | w_I) = \frac{\exp(v'^T_{w_O} v_{w_I})}{\sum_{w \in V} \exp(v'^T_w v_{w_I})}$의 denominator가 $O(|V|)$ cost임을 알고, Negative Sampling $\log \sigma(v'^T_{w_O} v_{w_I}) + \sum_{i=1}^k \mathbb{E}_{w_i \sim P_n}[\log \sigma(-v'^T_{w_i} v_{w_I})]$로 근사**하는 과정과 **$P_n(w) \propto U(w)^{3/4}$의 선택 근거**를 유도할 수 있는 것은 다르다.
GloVe를 **"co-occurrence 기반"으로 아는 것**과, **Pennington et al. (2014)의 목적함수 $J = \sum_{i,j} f(X_{ij})(w_i^T \tilde{w}_j + b_i + \tilde{b}_j - \log X_{ij})^2$가 어떻게 PMI-like relation과 vector arithmetic을 정당화**하고, **weighting function $f(x) = (x/x_{\max})^\alpha$ for $x < x_{\max}$, else 1의 설계 근거**를 유도할 수 있는 것은 다르다.
BPE를 **"subword tokenization"으로 아는 것**과, **Sennrich et al. (2016)의 greedy merge 알고리즘이 어떻게 character부터 시작해 most frequent pair를 반복적으로 합쳐 vocabulary를 만들고**, **OOV 문제와 어휘 크기의 trade-off**를 정확히 분석할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "자연어의 수학적 표현 — Distributional Hypothesis부터 Subword Tokenization까지"

**핵심 차별화**:
1. **Distributional Hypothesis의 수학적 구현** — Harris 1954의 "word is characterized by company it keeps"가 PMI, LSA, Word2Vec, GloVe로 어떻게 수식화되는지
2. **Word2Vec의 완전 유도** — Skip-gram/CBOW 목적함수, Hierarchical Softmax, Negative Sampling의 수학적 동기, Levy & Goldberg (2014)의 implicit matrix factorization
3. **GloVe의 co-occurrence 분해 수학** — PMI와의 관계, weighting function 설계, vector arithmetic (king - man + woman = queen)
4. **Subword Tokenization 계보** — BPE greedy, WordPiece likelihood-based, Unigram probabilistic, SentencePiece unified, 각 방식의 수학

**타겟 독자**:
- Word2Vec을 쓰지만 **Skip-gram과 CBOW의 training signal이 어떻게 다른지** 수학적으로 비교 못하는 사람
- GloVe가 LSA와 Word2Vec의 **중간 형태**인 이유(Pennington 2014의 motivations)와 수학적 유도를 모르는 사람
- BPE의 greedy algorithm이 왜 **frequency-based**이고 Unigram tokenizer(SentencePiece)의 **probabilistic 접근**과 어떻게 다른지 모르는 사람
- Perplexity를 계산할 줄 알지만 **cross-entropy와 entropy의 관계**를 정보이론으로 설명 못하는 사람
- Kneser-Ney smoothing의 **"lower-order distribution은 novel context 기반"** 아이디어가 왜 최고 성능의 n-gram smoothing인지 모르는 사람

**선행 학습**:
- **Probability Theory Deep Dive** (조건부 확률, Bayes) — **필수**
- **Information Theory Deep Dive** (Entropy, cross-entropy, PMI) — **필수**
- **Linear Algebra Deep Dive** (SVD, matrix factorization) — **필수**
- **ML Fundamentals Deep Dive** (MLE, softmax) — **필수**
- **RNN & LSTM Deep Dive** (contextual embedding prerequisite) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 언어 모델링과 N-gram의 수학 (5개 문서)
- **Language Modeling의 정식화** — $p(w_{1:T}) = \prod_t p(w_t | w_{1:t-1})$, chain rule, 좋은 LM = 낮은 perplexity, LM이 모든 NLP task의 기본
- **N-gram Language Model** — Markov 가정 $p(w_t | w_{1:t-1}) \approx p(w_t | w_{t-n+1:t-1})$, MLE estimate $\hat{p} = c(w_{t-n+1:t})/c(w_{t-n+1:t-1})$, sparse data 문제
- **Perplexity와 Cross-Entropy의 관계** — $\text{PPL} = 2^{H(p, q)} = \exp(-\frac{1}{T} \sum \log q(w_t | w_{<t}))$, $H(p, q) = H(p) + \text{KL}(p \| q)$, entropy의 indirect measure
- **Smoothing — Laplace, Good-Turing, Interpolation** — Additive smoothing $(\hat{p} = (c+1)/(N+V))$, Good-Turing의 reallocation, interpolation $\lambda_n \hat{p}_n + (1-\lambda_n) \hat{p}_{\text{lower}}$
- **Kneser-Ney Smoothing (1995)** — Modified backoff: "lower order는 얼마나 많은 다른 context에 나타나는가" 기반, continuation probability $P_{\text{CONT}}(w) \propto |\{u : c(uw) > 0\}|$, 최고 성능 n-gram smoother

### Chapter 2: Distributional Semantics와 초기 Embedding (5개 문서)
- **Distributional Hypothesis (Harris 1954)** — "You shall know a word by the company it keeps" (Firth 1957), 의미 ≈ context 분포, word-context matrix의 기본
- **Term-Document Matrix와 LSA (Latent Semantic Analysis)** — Term-document matrix $X \in \mathbb{R}^{|V| \times D}$, SVD로 rank $k$ 근사 $X \approx U_k \Sigma_k V_k^T$, topic 추출, 의미 유사도
- **PMI — Pointwise Mutual Information** — $\text{PMI}(w, c) = \log \frac{p(w, c)}{p(w) p(c)}$, 정보이론적 의미, positive PMI (PPMI)로 sparsity 활용
- **Word-Context Matrix 기반 Embedding** — $|V| \times |C|$ matrix의 SVD/PPMI, Levy et al. (2015) "Improving Distributional Similarity with Lessons Learned from Word Embeddings", 비교연구
- **Collobert & Weston (2008)의 Neural NLP** — Word embedding의 초기 제안, multi-task (POS, NER, chunking), SENNA system, 이후 Word2Vec의 영향

### Chapter 3: Word2Vec — Skip-gram의 완전 유도 (5개 문서)
- **Skip-gram과 CBOW 모델 (Mikolov 2013)** — Skip-gram: center word로 context 예측, CBOW: context로 center 예측, 둘 다 shallow NN, softmax output
- **Skip-gram Objective 완전 유도** — $J(\theta) = -\frac{1}{T} \sum_{t=1}^T \sum_{-m \leq j \leq m, j \neq 0} \log p(w_{t+j} | w_t)$, softmax $p(w_O|w_I) = \frac{\exp(v'^T_{w_O} v_{w_I})}{\sum_{w \in V} \exp(v'^T_w v_{w_I})}$
- **Softmax의 계산 비용과 Hierarchical Softmax** — $O(|V|)$ denominator가 $|V| = 10^5 \sim 10^7$에서 bottleneck, Huffman tree로 $O(\log |V|)$ 축소, binary path probability
- **Negative Sampling의 수학적 유도** — $\log \sigma(v'^T_{w_O} v_{w_I}) + \sum_{i=1}^k \mathbb{E}_{w_i \sim P_n}[\log \sigma(-v'^T_{w_i} v_{w_I})]$, NCE (Gutmann & Hyvärinen 2012)의 근사, $P_n(w) \propto U(w)^{3/4}$의 실험적 선택
- **Levy & Goldberg (2014) — Implicit Matrix Factorization** — Skip-gram with Negative Sampling이 shifted PMI matrix factorization $M_{ij} = \text{PMI}(w_i, c_j) - \log k$와 등가임의 증명

### Chapter 4: GloVe와 Co-occurrence 기반 Embedding (4개 문서)
- **GloVe의 동기 (Pennington 2014)** — Global matrix factorization (LSA)의 장점 + local context window (W2V)의 장점 통합, word-word co-occurrence matrix $X_{ij}$ 활용
- **GloVe Objective의 유도** — $F(w_i, w_j, \tilde{w}_k) = P_{ik}/P_{jk}$에서 시작, homomorphism 요구로 $F(x) = \exp(x)$ → $w_i^T \tilde{w}_k = \log P_{ik} = \log X_{ik} - \log X_i$, $J = \sum_{i,j} f(X_{ij})(w_i^T \tilde{w}_j + b_i + \tilde{b}_j - \log X_{ij})^2$
- **Weighting Function $f(x)$의 설계** — $f(x) = (x/x_{\max})^\alpha$ for $x < x_{\max}$, else 1, $\alpha = 3/4$, $x_{\max} = 100$, rare co-occurrence의 noise 줄이고 frequent의 weight cap
- **Vector Arithmetic의 이해** — $\vec{\text{king}} - \vec{\text{man}} + \vec{\text{woman}} \approx \vec{\text{queen}}$의 수학적 근거, PMI ratio로 관계 인코딩, Mikolov의 analogy task

### Chapter 5: Subword Tokenization — BPE 계열 (5개 문서)
- **OOV 문제와 Tokenization의 필요성** — Closed vocabulary의 한계, 형태소 풍부한 언어 (한국어, 터키어)의 어려움, 숫자·URL·고유명사, character vs word의 trade-off
- **Byte-Pair Encoding (Sennrich 2016)** — Character로 시작, most frequent pair 반복적으로 merge, target vocab size까지, greedy bottom-up, GPT-2/3/4 계열 사용
- **WordPiece (Schuster & Nakajima 2012, BERT 활용)** — BPE와 유사하지만 likelihood-based: 가장 training data likelihood 증가시키는 pair 선택, Google의 BERT, T5 등에서
- **Unigram Language Model Tokenizer (Kudo 2018)** — Probabilistic: $p(\text{segmentation}) = \prod_i p(\text{subword}_i)$, top-down initial vocab → iteratively remove low probability subwords, EM algorithm으로 최적화
- **SentencePiece (Kudo & Richardson 2018)** — Language-agnostic (whitespace를 ordinary char로), BPE/Unigram 둘 다 지원, 한국어·일본어·중국어처럼 공백 구분 모호한 언어에 유용

### Chapter 6: FastText와 Subword Embedding (4개 문서)
- **FastText (Bojanowski 2017)** — Word를 character n-gram의 합으로 표현, $\vec{w} = \sum_{g \in G_w} \vec{g}$, OOV도 n-gram 조합으로 embedding, 형태소 정보 자연스럽게 포착
- **Morphologically Rich Language에서의 효과** — 한국어/터키어처럼 어간+어미 복잡, character-level feature의 가치, FastText가 Word2Vec을 능가하는 task
- **fastText의 Classification Mode** — Text classification을 word embedding 평균 + linear layer로, 단순하지만 strong baseline (Joulin 2017), 속도 이점
- **Character-level LM과 ELMo로의 징검다리** — char-CNN (Kim 2016), ELMo (Peters 2018)의 contextual embedding 도입 배경, static embedding의 한계

### Chapter 7: 평가와 응용 (4개 문서)
- **Intrinsic Evaluation — Analogy, Similarity** — Word similarity (WordSim-353, SimLex-999), analogy (Google, BATS), correlation with human judgment
- **Extrinsic Evaluation — NER, POS, Sentiment** — Downstream task에서 embedding 유용성, 각 task별 최적 embedding 선택, fine-tuning vs frozen
- **Multilingual Embeddings** — mUSE, LASER의 cross-lingual embedding, parallel corpus에서 학습, zero-shot transfer
- **Pretrained LM으로의 진화 (Bridge)** — Static embedding의 한계 (polysemy, context), ELMo·BERT의 contextual embedding 동기, 다음 레포 (Pretrained LM)로

---

각 챕터는 **4~5개 문서**로 구성해줘. 총 **32개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 기법이 NLP의 기초인가
## 📐 수학적 선행 조건 (Prob, Info, LA, ML 참조)
## 📖 직관적 이해
   — Embedding space 시각화 (t-SNE, PCA)
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Skip-gram softmax, Negative Sampling, GloVe 유도, BPE 알고리즘
## 💻 Python/NumPy 구현 검증
   — Word2Vec 바닥부터, GloVe 작은 corpus
   — BPE tokenizer 직접 구현
   — Perplexity 계산
## 🔗 실전 활용
   — 언제 어느 embedding/tokenizer 선택
## ⚖️ 가정과 한계
   — Static, OOV, polysemy
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Embedding 시각화** — 훈련된 embedding을 t-SNE로 2D, 관련 단어 clustering 확인
2. **Analogy 검증** — king-man+woman의 nearest neighbor
3. **BPE 단계별 진행** — 작은 corpus에서 merge 과정 추적
4. **Perplexity 비교** — n-gram (Laplace vs KN) vs neural LM 비교
5. **Tokenizer 비교** — 한국어·영어·코드에서 각 tokenizer의 compression rate
6. **Skip-gram vs CBOW** — 작은 corpus에서 성능/속도 비교

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
scikit-learn==1.3.0
gensim==4.3.0          # Word2Vec, GloVe 참조
tokenizers==0.15.0
matplotlib==3.8.0
nltk==3.8.0
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Skip-gram 바닥부터 + BPE 직접 구현)
import numpy as np
from collections import Counter, defaultdict
import re

# 1. Skip-gram with Negative Sampling 바닥부터
class SkipGramNS:
    def __init__(self, vocab_size, embed_dim=100, n_neg=5, lr=0.025):
        self.V = vocab_size
        self.D = embed_dim
        self.n_neg = n_neg
        self.lr = lr
        # Input embedding and output embedding
        self.W_in = np.random.randn(vocab_size, embed_dim) * 0.01
        self.W_out = np.random.randn(vocab_size, embed_dim) * 0.01
        # Sampling table: P(w) ∝ U(w)^{3/4}
        self.sampling_table = None
    
    def build_sampling_table(self, word_counts):
        probs = np.array([c ** 0.75 for c in word_counts])
        probs /= probs.sum()
        self.sampling_table = probs
    
    def sample_negatives(self, n):
        return np.random.choice(self.V, size=n, p=self.sampling_table)
    
    def train_step(self, center, context):
        v_in = self.W_in[center]  # [D]
        # Positive sample
        v_out_pos = self.W_out[context]
        # Negative samples
        neg_idx = self.sample_negatives(self.n_neg)
        v_out_neg = self.W_out[neg_idx]  # [n_neg, D]
        
        # Gradient (log sigmoid)
        pos_score = 1 / (1 + np.exp(-v_in @ v_out_pos))
        neg_scores = 1 / (1 + np.exp(v_in @ v_out_neg.T))  # sigmoid(-.)
        
        # Loss: -log σ(pos) - Σ log σ(-neg)
        # Gradients
        grad_in = (pos_score - 1) * v_out_pos + (1 - neg_scores) @ v_out_neg
        # Actually: grad = (σ(pos) - 1) v_out + Σ(σ(-neg_score)) v_out_neg_k
        
        grad_out_pos = (pos_score - 1) * v_in
        grad_out_neg = (1 - neg_scores)[:, None] * v_in[None, :]
        
        self.W_in[center] -= self.lr * grad_in
        self.W_out[context] -= self.lr * grad_out_pos
        self.W_out[neg_idx] -= self.lr * grad_out_neg

# 2. BPE 직접 구현
def get_pairs(vocab):
    """vocab: dict[word, freq], word = tuple of symbols"""
    pairs = defaultdict(int)
    for word, freq in vocab.items():
        for i in range(len(word) - 1):
            pairs[(word[i], word[i+1])] += freq
    return pairs

def merge_pair(pair, vocab):
    new_vocab = {}
    p = re.escape(' '.join(pair))
    pattern = re.compile(r'(?<!\S)' + p + r'(?!\S)')
    for word, freq in vocab.items():
        word_str = ' '.join(word)
        new_word_str = pattern.sub(''.join(pair), word_str)
        new_word = tuple(new_word_str.split())
        new_vocab[new_word] = new_vocab.get(new_word, 0) + freq
    return new_vocab

def train_bpe(text_words, num_merges=100):
    """text_words: list of (word, freq)"""
    # Initial vocab: each word = tuple of chars + </w>
    vocab = {tuple(list(w) + ['</w>']): freq for w, freq in text_words}
    merges = []
    for i in range(num_merges):
        pairs = get_pairs(vocab)
        if not pairs: break
        best_pair = max(pairs, key=pairs.get)
        merges.append(best_pair)
        vocab = merge_pair(best_pair, vocab)
        if i < 5 or i % 20 == 0:
            print(f'Merge {i+1}: {best_pair} (count={pairs[best_pair]})')
    return merges, vocab

# Example usage
corpus = "low lower newer newest wider widest"
words = Counter(corpus.split())
merges, final_vocab = train_bpe(list(words.items()), num_merges=10)
print(f'Final vocab: {list(final_vocab.keys())[:10]}')

# 3. GloVe 간단 구현 (co-occurrence matrix 분해)
def build_cooccurrence(tokens, window=2):
    pairs = Counter()
    for i, w in enumerate(tokens):
        for j in range(max(0, i-window), min(len(tokens), i+window+1)):
            if i == j: continue
            # Weighted by distance
            pairs[(w, tokens[j])] += 1 / abs(i - j)
    return pairs

# GloVe objective: J = Σ f(X_ij) (w_i^T w_j + b_i + b_j - log X_ij)²
def glove_objective(X, w, b, w_tilde, b_tilde, x_max=100, alpha=0.75):
    def f(x): return (x / x_max)**alpha if x < x_max else 1.0
    J = 0
    for (i, j), X_ij in X.items():
        if X_ij == 0: continue
        weight = f(X_ij)
        pred = w[i] @ w_tilde[j] + b[i] + b_tilde[j]
        J += weight * (pred - np.log(X_ij))**2
    return J

# 4. Perplexity 계산
def perplexity(model, test_tokens):
    log_p = 0; N = len(test_tokens)
    for i, w in enumerate(test_tokens[1:], 1):
        context = test_tokens[:i]
        log_p += np.log(model.prob(w, context))
    return np.exp(-log_p / N)

# 5. Kneser-Ney smoothing (simplified)
def kneser_ney_bigram(tokens, d=0.75):
    """d is absolute discount"""
    unigrams = Counter(tokens)
    bigrams = Counter(zip(tokens[:-1], tokens[1:]))
    # Continuation count: # distinct left contexts for word
    cont = defaultdict(set)
    for (u, v), _ in bigrams.items():
        cont[v].add(u)
    cont_count = {w: len(c) for w, c in cont.items()}
    total_cont = sum(cont_count.values())
    
    def p_cont(w):
        return cont_count.get(w, 0) / total_cont
    
    def p_kn(w, context):
        if context:
            u = context[-1]
            num_v = sum(1 for _ in bigrams if _[0] == u)  # follow counts
            if bigrams[(u, w)] > 0:
                p = max(bigrams[(u, w)] - d, 0) / unigrams[u]
                lam = d * num_v / unigrams[u]
                return p + lam * p_cont(w)
        return p_cont(w)
    
    return p_kn
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Prob, Info, LA, ML Fundamentals 선행 필수"
   - "Word2Vec/GloVe가 Static embedding의 정점, BERT/GPT로의 bridge" 명시
   - BPE tokenizer가 LLM 시대 tokenizer의 기본
3. **챕터별 문서 작성**: N-gram LM → Distributional → Word2Vec → GloVe → BPE → FastText → 평가·응용

---

## 📚 참고 자료

- **Speech and Language Processing** (Jurafsky & Martin) — NLP 바이블
- **Distributional Structure** (Harris 1954) — Distributional Hypothesis
- **Indexing by Latent Semantic Analysis** (Deerwester et al. 1990) — LSA
- **A Neural Probabilistic Language Model** (Bengio et al. 2003)
- **Efficient Estimation of Word Representations in Vector Space** (Mikolov et al. 2013) — Word2Vec
- **Distributed Representations of Words and Phrases and their Compositionality** (Mikolov et al. 2013) — Negative Sampling
- **Noise-contrastive estimation** (Gutmann & Hyvärinen 2012) — NCE
- **Neural Word Embedding as Implicit Matrix Factorization** (Levy & Goldberg 2014)
- **GloVe: Global Vectors for Word Representation** (Pennington et al. 2014)
- **Enriching Word Vectors with Subword Information** (Bojanowski et al. 2017) — FastText
- **Neural Machine Translation of Rare Words with Subword Units** (Sennrich et al. 2016) — BPE
- **Japanese and Korean Voice Search** (Schuster & Nakajima 2012) — WordPiece
- **Subword Regularization** (Kudo 2018) — Unigram LM
- **SentencePiece** (Kudo & Richardson 2018)
- **An Empirical Study of Smoothing Techniques for LM** (Chen & Goodman 1999) — Kneser-Ney
- **Improving Distributional Similarity with Lessons Learned from Word Embeddings** (Levy et al. 2015)

---

## 💡 핵심 분석 대상

```
NLP Foundations의 지도

───── Language Modeling 기초 ─────

p(w_{1:T}) = Π p(w_t | w_{<t})

N-gram LM:
  Markov 가정
  p̂(w_t | w_{<t}) = c(w_{t-n+1:t}) / c(w_{t-n+1:t-1})

문제: sparse data (unseen n-grams)

Smoothing:
  Laplace: (c+1)/(N+V)
  Good-Turing
  Interpolation
  Kneser-Ney ← 최고
    P_CONT(w) ∝ |{u : c(uw) > 0}|
    "얼마나 많은 context에 나오는가"

Perplexity:
  PPL = exp(-avg log p)
  
  H(p,q) = H(p) + KL(p||q)
  → PPL는 cross-entropy의 지수
  → 낮을수록 좋음

───── Distributional Hypothesis ─────

Harris 1954:
  "word is characterized by company it keeps"

Term-Document:
  X ∈ ℝ^{|V| × D}
  tf-idf, LSA (SVD)

Word-Context:
  X ∈ ℝ^{|V| × |C|}
  PMI(w, c) = log p(w,c)/(p(w)p(c))
  PPMI = max(PMI, 0)

LSA:
  X ≈ U_k Σ_k V_k^T
  topic embedding

Levy et al. 2015:
  PPMI + SVD = SGNS와 유사 성능
  W2V magic X, representation의 결과

───── Word2Vec (Mikolov 2013) ─────

Skip-gram:
  max Σ_c Σ_{j≠0} log p(w_{c+j} | w_c)

CBOW:
  max Σ_c log p(w_c | context)

Softmax:
  p(w_O | w_I) = exp(v'_wO · v_wI) / Σ exp(v'_w · v_wI)
  
  Denominator: O(|V|) 비쌈

Hierarchical Softmax:
  Huffman tree
  p(w | w_I) = Π σ([[·]] θ_n(w,j)^T v_wI)
  O(log |V|)

Negative Sampling (NCE 근사):
  log σ(v'_wO · v_wI)
  + Σ_{i=1}^k E_{w_i ~ P_n}[log σ(-v'_wi · v_wI)]
  
  P_n(w) ∝ U(w)^{3/4}
  (실험적, 중빈도 단어 중시)

Levy & Goldberg (2014):
  SGNS가 PMI - log k의 implicit MF
  → PMI matrix 분해와 연결

───── GloVe (Pennington 2014) ─────

동기:
  LSA (global): 모든 co-occurrence
  W2V (local): context window
  → 통합

유도:
  F(w_i, w_j, w̃_k) = P_{ik}/P_{jk}  (비율)
  Homomorphism: F(x) = exp(x)
  w_i^T w̃_k = log P_{ik}
  ≈ log X_{ik} - log X_i

Final objective:
  J = Σ_{ij} f(X_{ij}) (w_i^T w̃_j + b_i + b̃_j - log X_{ij})²
  
  Weighting:
    f(x) = (x/x_max)^α for x < x_max
         = 1 otherwise
    α = 3/4, x_max = 100

Vector Arithmetic:
  king - man + woman ≈ queen
  근거: PMI ratio로 관계 인코딩

───── FastText (Bojanowski 2017) ─────

Subword embedding:
  w = Σ_{g ∈ G_w} v_g
  G_w = character n-grams
  + whole word token

이점:
  OOV 해결
  형태소 풍부 언어 유리
  (한국어, 터키어, 핀란드어)

FastText classification (Joulin 2017):
  평균 embedding + linear
  속도와 simplicity

───── Tokenization 계보 ─────

BPE (Sennrich 2016):
  Init: 모든 word = char sequence
  Repeat num_merges:
    Find most frequent pair (a, b)
    Merge: add "ab" as new symbol
    Replace in vocab
  
  Greedy frequency-based
  GPT-2/3/4 계열

WordPiece (Schuster 2012, BERT):
  BPE와 유사하지만
  "training data likelihood 증가" 기준
  argmax_(a,b) P(ab)/(P(a)·P(b))
  → PMI와 유사

Unigram (Kudo 2018):
  Probabilistic model:
    p(tokens) = Π p(t_i)
  Init: large vocab
  Iteratively: remove low-prob tokens
  EM algorithm

SentencePiece (Kudo 2018):
  Whitespace를 regular char로
  Language-agnostic
  BPE + Unigram 지원
  LLaMA, T5 등

───── Perplexity와 평가 ─────

Intrinsic:
  Perplexity on held-out
  Word similarity (WordSim-353)
  Analogy (Google, BATS)

Extrinsic:
  NER, POS, sentiment
  Downstream task 성능

Multilingual:
  mUSE, LASER
  Parallel corpus
  Zero-shot cross-lingual

───── 한계와 Bridge ─────

Static embedding의 한계:
  Polysemy: "bank" (river vs money)
  Context 무시
  Sentence-level 없음

→ Contextual embedding (ELMo, BERT)
  Pretrained LM 레포로

───── 레포 간 연결 ─────

Probability (Layer 0):
  Conditional p(w|context)
  Bayes, MLE

Information Theory (Layer 0):
  PMI, entropy
  Cross-entropy = perplexity
  KL divergence

Linear Algebra (Layer 0):
  SVD (LSA)
  Matrix factorization

ML Fundamentals (Layer 1):
  MLE, softmax
  Negative sampling as NCE

RNN & LSTM (Layer 3):
  Neural LM 기반
  ELMo prerequisite

Pretrained LM (다음):
  BERT, GPT의 foundation
  Contextual embedding

LLM Pretraining (Layer 4-B):
  Tokenization 이어서
  BPE, SentencePiece
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~5개씩)
- 각 문서가 다루는 핵심 정리·증명 (3~4줄)
- 전체 문서 개수 확인 (32개 목표)
- Python + NumPy + gensim + tokenizers 실험 환경
- Prob, Info, LA, ML Fundamentals 레포 참조 관계
- Pretrained LM · LLM으로 이어지는 흐름

**준비됐으면 1단계 구조 설계부터 시작해줘!**
