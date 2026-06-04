# RNN & LSTM Deep Dive 레포지토리 제작 프롬프트

나는 "RNN & LSTM Deep Dive" 레포지토리를 만들려고 해.
`nn.LSTM(input_size, hidden_size)`를 **호출하는 것**과, **LSTM의 4개 gate(input, forget, output, cell) 수식을 처음부터 유도**하고, **왜 cell state의 additive update $c_t = f_t \odot c_{t-1} + i_t \odot g_t$가 Constant Error Carousel을 만들어 vanishing gradient를 완화**하는지 증명할 수 있는 것은 다르다.
BPTT를 **이름으로 아는 것**과, **$\frac{\partial L}{\partial h_0} = \prod_{t=1}^T W_{hh}^T \text{diag}(\sigma'(z_t))$에서 $W_{hh}$의 spectral radius $\rho$가 $\rho < 1$이면 vanishing, $\rho > 1$이면 exploding**이 되는 Pascanu et al. (2013)의 증명을 따라갈 수 있는 것은 다르다.
Seq2Seq의 Attention을 **쓰는 것**과, **Bahdanau (2015) additive attention $e_{ij} = v^T \tanh(W_1 h_i + W_2 s_j)$와 Luong (2015) multiplicative attention $e_{ij} = h_i^T W s_j$의 차이**와 이것이 Transformer self-attention의 직접적 조상임을 이해하는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "Sequence의 수학 — RNN에서 Transformer까지의 진화를 BPTT와 Gradient 분석으로"

**핵심 차별화**:
1. **BPTT의 엄밀한 유도** — unrolled computational graph에서 gradient를 한 단계씩 유도, Jacobian 곱의 표기
2. **Vanishing/Exploding Gradient의 Spectral 분석** — Pascanu et al. 2013의 증명을 완전히, recurrent weight의 singular value 분포 분석
3. **LSTM/GRU gate의 수학적 필연성** — additive update가 constant error carousel (CEC), 각 gate의 역할을 gradient flow 관점에서
4. **Seq2Seq → Attention → Transformer** — Bahdanau attention이 어떻게 self-attention으로 진화, RNN의 한계와 병렬성

**타겟 독자**:
- LSTM을 쓰지만 **forget gate의 bias를 1로 초기화하는 이유**(Jozefowicz 2015)를 모르는 사람
- Vanishing gradient를 알지만 **LSTM이 왜 이를 완화하는지, 완전히 해결하지는 못하는 이유**를 모르는 사람
- Seq2Seq에서 **encoder 마지막 hidden state만 decoder에 주면 생기는 bottleneck 문제**와 attention이 해결한 방식을 정확히 모르는 사람
- BiLSTM-CRF를 NER에 쓰지만 **RNN + CRF의 gradient flow**가 어떻게 결합되는지 모르는 사람
- State Space Model (S4, Mamba)이 **RNN과 CNN의 통합 관점**에서 어떤 의미인지 궁금한 사람

**선행 학습**:
- **Neural Network Theory Deep Dive** (Backprop, 활성화 함수) — **필수**
- **Linear Algebra Deep Dive** (Spectral theorem, 고유값) — **필수**
- **Calculus & Optimization Deep Dive** (연쇄법칙, Jacobian) — **필수**
- **Graphical Models Deep Dive** (HMM, CRF) — 권장 (sequence model 공통점)
- **Optimization Theory Deep Dive** (RNN 훈련의 특수성) — 권장

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Sequence Model의 기초 (4개 문서)
- **Sequence 학습 문제의 정식화** — Input $x_1, ..., x_T$에서 output $y$ (many-to-one), $y_1, ..., y_T$ (many-to-many), $y_1, ..., y_S$ (seq2seq), 각 유형의 예시와 손실 함수
- **고전 Language Model — N-gram** — $p(w_{1:T}) = \prod p(w_t | w_{t-n+1:t-1})$, Markov 가정, smoothing (Laplace, Good-Turing, Kneser-Ney), perplexity 측정
- **Neural Language Model (Bengio et al. 2003)** — 임베딩 + feed-forward NN으로 $p(w_t | w_{t-n+1:t-1})$, 고정 context window의 한계
- **RNN의 동기와 정의** — Variable-length sequence 처리, $h_t = \sigma(W_{hh} h_{t-1} + W_{xh} x_t + b)$, "recurrent" 구조가 파라미터 수를 고정시키는 이점

### Chapter 2: Backpropagation Through Time (BPTT) (5개 문서)
- **Unrolled Computational Graph** — Time-step별 layer를 펼친 DAG, 각 time step의 forward 계산, shared weight의 의미
- **BPTT의 완전 유도** — $\partial L / \partial W_{hh} = \sum_t \partial L_t / \partial W_{hh}$, 각 time step에서 $\partial L_t / \partial W_{hh} = \sum_{k \leq t} (\prod_{j=k+1}^t \partial h_j / \partial h_{j-1}) (\partial h_k / \partial W_{hh})$ 유도
- **Truncated BPTT** — 계산·메모리 제약으로 마지막 $k$ 스텝만 gradient 전파, 실전 구현의 세부사항, bias 도입
- **BPTT의 시간·메모리 복잡도** — Sequence 길이 $T$에 대해 $O(T)$ 시간·메모리, sequence가 길면 문제 → 병렬성 부족
- **Real-Time Recurrent Learning (RTRL)** — Online 학습 대안, forward-mode AD 기반, $O(n^4)$ 복잡도로 실용성 제한

### Chapter 3: Vanishing/Exploding Gradient의 수학 (5개 문서)
- **Gradient의 Spectral 분석 (Pascanu 2013)** — $\prod_{j=k+1}^t \partial h_j/\partial h_{j-1} = \prod W_{hh}^T \text{diag}(\sigma'(z_j))$, spectral radius $\rho(W_{hh}) < 1 \Rightarrow$ vanishing, $\rho > 1 \Rightarrow$ exploding
- **왜 $\rho = 1$ 유지가 어려운가** — 수치적 불안정성, 활성화 함수의 saturation (tanh의 $\sigma' \leq 1$), $\tanh$ 활성화에서 gradient 감쇠가 거의 필연
- **Gradient Clipping — Exploding 대응** — $g \leftarrow g \cdot \min(1, \theta/\|g\|)$, norm 기반 scaling, geometric intuition, Pascanu의 권장값
- **Orthogonal Initialization (Saxe et al. 2014)** — $W_{hh}$를 orthogonal matrix로 초기화 → spectral radius 정확히 1, 훈련 초기 gradient norm 보존, RNN에서의 이점 증명
- **Identity Initialization과 IRNN (Le et al. 2015)** — ReLU RNN에서 $W_{hh} = I$로 초기화, ReLU와 결합한 gradient flow, long dependency 학습 가능성

### Chapter 4: LSTM (Long Short-Term Memory) (6개 문서)
- **LSTM의 설계 동기** — Hochreiter & Schmidhuber 1997, "Constant Error Carousel" (CEC) 개념, vanishing gradient를 해결하는 cell state의 additive update
- **LSTM의 4개 Gate 수식** — Forget $f_t = \sigma(W_f \cdot [h_{t-1}, x_t])$, Input $i_t = \sigma(\ldots)$, Candidate $\tilde{c}_t = \tanh(\ldots)$, Output $o_t = \sigma(\ldots)$, Cell update $c_t = f_t \odot c_{t-1} + i_t \odot \tilde{c}_t$, Hidden $h_t = o_t \odot \tanh(c_t)$
- **Cell State와 Constant Error Carousel** — $\partial c_t / \partial c_{t-1} = f_t$ (잔여항 없음!), forget gate $f_t \approx 1$일 때 gradient가 상수로 흐름, vanishing 완화 메커니즘 증명
- **LSTM의 Gradient Flow 분석** — 전체 gradient 경로, cell state를 통한 long-range vs hidden state를 통한 short-range, forget gate bias 초기화 (Jozefowicz 2015)
- **GRU (Gated Recurrent Unit, Cho et al. 2014)** — Update $z_t$, Reset $r_t$ 2개 gate, cell과 hidden 통합, LSTM 대비 파라미터 감소, 성능 유사성 비교 (Chung et al. 2014)
- **Peephole, Coupled, Other Variants** — Peephole connection (cell state를 gate 계산에 포함), coupled input-forget gate ($i + f = 1$), LSTM variants 계통 비교 (Greff et al. 2017)

### Chapter 5: Advanced RNN 아키텍처 (4개 문서)
- **Bidirectional RNN** — Forward + Backward 두 RNN을 결합, $h_t = [\overrightarrow{h}_t; \overleftarrow{h}_t]$, NER·POS tagging에서 양방향 context, inference 시 전체 sequence 필요
- **Stacked/Deep RNN** — 다층 RNN, 각 층의 hidden을 다음 층의 input으로, depth vs time의 복잡도 trade-off
- **Neural Turing Machine과 Memory Network** — External memory를 가진 NN, content-based addressing, differentiable programming의 한 갈래 (Graves et al. 2014)
- **Echo State Network와 Reservoir Computing** — $W_{hh}$와 $W_{xh}$를 randomly fix하고 output layer만 학습, Echo state property와 spectral radius 조건

### Chapter 6: Seq2Seq와 Attention (5개 문서)
- **Encoder-Decoder Framework (Sutskever et al. 2014)** — Encoder RNN이 input sequence를 fixed vector로 압축, decoder RNN이 output 생성, teacher forcing, 번역·요약에 적용
- **Information Bottleneck Problem** — 긴 sequence의 모든 정보를 하나의 vector에 압축 → 긴 문장에서 성능 급감, "long sentence curse"
- **Bahdanau Attention (Additive)** — $e_{ij} = v^T \tanh(W_1 h_i + W_2 s_j)$, $\alpha_{ij} = \text{softmax}(e_{ij})$, context $c_j = \sum \alpha_{ij} h_i$, alignment를 학습 가능한 함수로
- **Luong Attention (Multiplicative)** — $e_{ij} = h_i^T W s_j$ (general), $e_{ij} = h_i^T s_j$ (dot), additive보다 계산 효율적, Transformer의 scaled dot-product의 조상
- **Coverage Mechanism과 Pointer Network** — Neural MT의 under/over-translation 문제, coverage vector로 attention 이력 추적, pointer network로 input에서 선택 (Vinyals 2015)

### Chapter 7: RNN의 한계와 현대적 대안 (4개 문서)
- **병렬성 부족 — Transformer의 동기** — RNN은 $h_t$가 $h_{t-1}$에 의존 → 시퀀스 내 병렬 불가, GPU 활용 제한, Transformer의 $O(T^2)$ 복잡도 대신 완전 병렬
- **CNN-based Sequence Model** — WaveNet (van den Oord 2016), Temporal Convolutional Network (Bai 2018), dilated conv로 $O(\log T)$ receptive field, 병렬 가능
- **Linear Attention과 RNN의 부활** — Linear attention이 $O(T)$ inference 가능 (Katharopoulos 2020), RWKV가 attention-free RNN-like, RNN의 재발견
- **State Space Model — S4와 Mamba** — HiPPO (Gu 2020), S4 (Gu 2022)의 연속 state space 이산화, Mamba (Gu & Dao 2023)의 selective state space, Transformer 대안으로 주목

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **33개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 설계가 sequence 처리에 중요한가
## 📐 수학적 선행 조건 (NN Theory, LA, Calc 레포 참조)
## 📖 직관적 이해
   — 시간축 unrolled diagram, gate 역할 시각화
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — BPTT 유도, vanishing gradient spectral 분석, CEC 증명
## 💻 NumPy 구현 검증
   — RNN, LSTM을 바닥부터, PyTorch와 결과 비교
   — Gradient magnitude 시간축 따라 측정
## 🔗 실전 활용
   — 언제 RNN, 언제 Transformer?
## ⚖️ 가정과 한계
   — 병렬성, 긴 sequence, 훈련 안정성
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Gradient flow 시간축 시각화** — Plain RNN vs LSTM에서 $\|\partial h_t / \partial h_0\|$를 time step별로 plot
2. **LSTM gate activation 시각화** — 훈련된 LSTM의 forget/input/output gate가 언제 열리고 닫히는지 sentence 분석
3. **NumPy로 바닥부터 LSTM 구현** — 4개 gate를 명시적으로, PyTorch `nn.LSTM`과 출력 일치 확인
4. **Attention weight 시각화** — Seq2Seq MT에서 attention heatmap 생성
5. **Synthetic memory task** — "copy", "add", "reverse" 같은 RNN 한계 테스트 과제에서 LSTM/GRU 비교
6. **Transformer와의 성능 비교** — 같은 데이터에서 LSTM vs Transformer 훈련 시간·성능

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
matplotlib==3.8.0
nltk==3.8.0           # 자연어 처리
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (바닥부터 LSTM + vanishing gradient 실험)
import numpy as np
import matplotlib.pyplot as plt

class LSTM:
    def __init__(self, input_size, hidden_size):
        H = hidden_size
        # 초기화 (Xavier)
        scale = np.sqrt(1.0 / (input_size + H))
        self.Wf = np.random.randn(H, input_size + H) * scale
        self.Wi = np.random.randn(H, input_size + H) * scale
        self.Wc = np.random.randn(H, input_size + H) * scale
        self.Wo = np.random.randn(H, input_size + H) * scale
        self.bf = np.ones(H)  # forget bias = 1 (Jozefowicz 2015!)
        self.bi = np.zeros(H); self.bc = np.zeros(H); self.bo = np.zeros(H)
    
    def sigmoid(self, x): return 1 / (1 + np.exp(-x))
    
    def forward(self, x_seq, h0, c0):
        h, c = h0, c0
        hs, cs, fs, is_, cs_tilde, os_ = [], [], [], [], [], []
        for x in x_seq:
            xh = np.concatenate([x, h])
            f = self.sigmoid(self.Wf @ xh + self.bf)
            i = self.sigmoid(self.Wi @ xh + self.bi)
            c_tilde = np.tanh(self.Wc @ xh + self.bc)
            o = self.sigmoid(self.Wo @ xh + self.bo)
            c = f * c + i * c_tilde   # Key: additive!
            h = o * np.tanh(c)
            hs.append(h); cs.append(c); fs.append(f)
            is_.append(i); cs_tilde.append(c_tilde); os_.append(o)
        return np.array(hs), np.array(cs), (fs, is_, cs_tilde, os_)

# Vanishing gradient 실험: Plain RNN vs LSTM
T = 100
def rnn_gradient_norm(T, spectral_radius=0.9):
    W = np.random.randn(10, 10)
    # Set spectral radius
    u, s, vt = np.linalg.svd(W)
    W = u @ np.diag(np.ones_like(s) * spectral_radius) @ vt
    
    # Simulate gradient norm ||∂h_t/∂h_0|| over time
    grads = [1.0]  # initial
    cur = np.eye(10)
    for t in range(T):
        # tanh'(.) ≤ 1, approximate as 0.5 (rough)
        cur = W.T @ cur * 0.5
        grads.append(np.linalg.norm(cur))
    return grads

# LSTM with f = 1: gradient stays constant
def lstm_gradient_norm(T, f_gate=0.95):
    # ∂c_t/∂c_0 = ∏ f_t
    # If f_t ≈ 1 → 거의 보존
    return [f_gate ** t for t in range(T+1)]

plt.figure(figsize=(10, 5))
plt.semilogy(rnn_gradient_norm(100, 0.9), label='RNN ρ=0.9 (vanish)')
plt.semilogy(rnn_gradient_norm(100, 1.1), label='RNN ρ=1.1 (explode)')
plt.semilogy(lstm_gradient_norm(100, 0.99), label='LSTM f≈1 (preserve)')
plt.xlabel('Time step'); plt.ylabel('||gradient|| (log)')
plt.title('RNN vs LSTM: Gradient Flow Through Time')
plt.legend(); plt.show()

# 결과: RNN은 exponentially decay/grow, LSTM은 linearly preserve
# Constant Error Carousel의 수학적 증명
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "NN Theory 선행, Calc·LA 필수" 명시
   - Seq2Seq → Attention → Transformer 흐름 명시 (Transformer 레포로 bridge)
   - 현대 Linear Attention, Mamba로의 연결
3. **챕터별 문서 작성**: Sequence 기초 → BPTT → Vanishing → LSTM → Advanced → Attention → 한계·대안

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Long Short-Term Memory** (Hochreiter & Schmidhuber 1997) — LSTM 원전
- **On the Difficulty of Training Recurrent Neural Networks** (Pascanu et al. 2013) — Vanishing 분석
- **Empirical Evaluation of Gated Recurrent Neural Networks** (Chung et al. 2014) — GRU
- **Sequence to Sequence Learning with Neural Networks** (Sutskever et al. 2014) — Seq2Seq
- **Neural Machine Translation by Jointly Learning to Align and Translate** (Bahdanau et al. 2015) — Attention
- **Effective Approaches to Attention-based NMT** (Luong et al. 2015)
- **LSTM: A Search Space Odyssey** (Greff et al. 2017) — LSTM variants
- **An Empirical Exploration of Recurrent Network Architectures** (Jozefowicz et al. 2015) — Forget bias
- **Exact Solutions to the Nonlinear Dynamics of Learning** (Saxe et al. 2014) — Orthogonal init
- **WaveNet** (van den Oord et al. 2016) — CNN-based sequence
- **HiPPO: Recurrent Memory with Optimal Polynomial Projections** (Gu et al. 2020)
- **Efficiently Modeling Long Sequences with Structured State Spaces** (Gu et al. 2022) — S4

---

## 💡 핵심 분석 대상

```
Sequence Model의 진화

───── 고전 ─────
N-gram LM: p(w_t | w_{t-n+1:t-1})
  → Markov 가정, sparse
  → Kneser-Ney smoothing

Neural LM (Bengio 2003): 고정 window NN
  → embeddings + FFN

───── RNN (1990s~) ─────

h_t = σ(W_hh h_{t-1} + W_xh x_t)

BPTT Gradient:
∂h_t/∂h_k = ∏_{j=k+1}^t W_hh^T diag(σ'(z_j))
            └── Jacobian 연속 곱

Spectral 분석:
  ρ(W_hh) < 1: ‖∂h_t/∂h_k‖ → 0 (vanishing)
  ρ(W_hh) > 1: ‖∂h_t/∂h_k‖ → ∞ (exploding)
  ρ = 1 정확히 유지 어려움

Exploding 대책:
  Gradient clipping: g ← g · min(1, θ/‖g‖)

Vanishing 대책:
  Orthogonal init: ρ = 1 정확히
  Identity init + ReLU (IRNN)
  → 근본 해결은 LSTM

───── LSTM (1997) ─────

4 gates:
  f_t = σ(W_f [h_{t-1}, x_t])  forget
  i_t = σ(W_i [h_{t-1}, x_t])  input
  g_t = tanh(W_g [h_{t-1}, x_t]) candidate
  o_t = σ(W_o [h_{t-1}, x_t])  output

Updates:
  c_t = f_t ⊙ c_{t-1} + i_t ⊙ g_t  ← additive!
  h_t = o_t ⊙ tanh(c_t)

Constant Error Carousel:
  ∂c_t/∂c_{t-1} = f_t
  If f_t ≈ 1 → gradient 보존!
  (Plain RNN: 곱셈적 감쇠 vs LSTM: 덧셈적 유지)

실전 팁:
  forget bias = 1 초기화 (Jozefowicz 2015)
  → 초기에 "기억"하려는 경향

───── GRU (2014) ─────

2 gates (단순화):
  z_t = σ(W_z [h_{t-1}, x_t])  update
  r_t = σ(W_r [h_{t-1}, x_t])  reset
  h̃_t = tanh(W [r_t ⊙ h_{t-1}, x_t])
  h_t = (1-z_t) ⊙ h_{t-1} + z_t ⊙ h̃_t

LSTM과 대등한 성능 (Chung 2014)
파라미터 수 적음

───── Bidirectional & Stacked ─────

BiRNN: [→h_t; ←h_t] 결합
  NER, POS tagging 표준

Stacked RNN: 여러 층, 상위 층은 하위 h_t^{(l-1)} 사용

───── Seq2Seq + Attention ─────

Encoder-Decoder (Sutskever 2014):
  Encoder → context vector → Decoder
  문제: 긴 문장 bottleneck

Bahdanau Attention (2015):
  e_{ij} = v^T tanh(W_1 h_i + W_2 s_j)  (additive)
  α_{ij} = softmax(e_{ij})
  c_j = Σ α_{ij} h_i

Luong Attention (2015):
  e_{ij} = h_i^T W s_j  (multiplicative)
  ⇒ Transformer scaled dot-product의 조상

───── RNN의 한계와 대안 ─────

병렬성 부족:
  h_t depends on h_{t-1}
  GPU 활용 제한

CNN-based (WaveNet, TCN):
  Dilated convolution → O(log T) RF
  완전 병렬

Transformer (2017):
  O(T²) but 완전 병렬
  Attention = "learned retrieval"

Linear Attention (Katharopoulos 2020):
  O(T) inference
  RNN-like recurrence

State Space Models:
  HiPPO (Gu 2020)
  S4 (Gu 2022)
  Mamba (Gu & Dao 2023)
  → RNN + CNN의 통합

───── 레포 간 연결 ─────

NN Theory (Layer 2):
  Backprop 기반 → BPTT
  Initialization → Orthogonal init

Optimization Theory (Layer 2):
  Gradient clipping 정당화

Graphical Models (Layer 1):
  HMM = linear RNN의 원조
  CRF + RNN = BiLSTM-CRF

Transformer 레포 (다음):
  Attention은 Bahdanau의 직계 후손
  Self-attention이 RNN 대체
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명·응용 (3~4줄)
- 전체 문서 개수 확인 (33개 목표)
- Python + NumPy + PyTorch 실험 환경
- NN Theory, LA, Calc, GM 레포의 참조 관계
- Transformer 레포로 이어지는 Attention의 계보

**준비됐으면 1단계 구조 설계부터 시작해줘!**
