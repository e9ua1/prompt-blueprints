# Audio & Speech Deep Dive 레포지토리 제작 프롬프트

나는 "Audio & Speech Deep Dive" 레포지토리를 만들려고 해.
STFT를 **"시간-주파수 변환"으로 아는 것**과, **$X(m, \omega) = \sum_n x[n] w[n - mH] e^{-j\omega n}$에서 window 함수 $w$의 선택이 time-frequency resolution의 Heisenberg uncertainty $\Delta t \cdot \Delta f \geq \frac{1}{4\pi}$와 연결**되고, **Mel-scale $m = 2595 \log_{10}(1 + f/700)$이 왜 인간 청각의 perceptual 특성(cochlear basilar membrane)을 근사**하는지 심리음향학적으로 유도할 수 있는 것은 다르다.
CTC Loss를 **"정렬 없이 학습"으로 아는 것**과, **Graves et al. (2006)의 $P(y|x) = \sum_{\pi \in \mathcal{B}^{-1}(y)} P(\pi|x)$에서 모든 가능한 path $\pi$를 exponential 많은 alignment로 합치되, forward-backward algorithm으로 $O(T \cdot |y|)$에 dynamic programming** 가능함과 blank token $\phi$의 역할을 유도할 수 있는 것은 다르다.
Residual Vector Quantization (RVQ)을 **"Neural codec의 핵심"으로 아는 것**과, **Zeghidour et al. (2021) SoundStream의 residual 구조 $r_k = x - \sum_{i<k} q_i(r_{i})$, 각 stage가 이전 quantization residual을 학습하는 greedy 분해**가 왜 단순 VQ보다 $N^K$배의 표현력을 주고 bitrate 조절을 가능케 하는지 증명할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "오디오 신호의 수학 — Signal Processing부터 Neural Codec·Speech LLM까지"

**핵심 차별화**:
1. **Signal Processing의 엄밀한 복원** — Fourier·STFT·Mel의 물리학과 심리음향학, window function trade-off, Griffin-Lim phase reconstruction
2. **CTC Loss의 Dynamic Programming 완전 유도** — Forward-backward algorithm, blank token의 정보이론적 역할, RNN-Transducer로의 확장
3. **Self-Supervised Speech 계보** — wav2vec 2.0의 contrastive + masked, HuBERT의 iterative cluster assignment, WavLM, Whisper의 weakly-supervised
4. **Neural Audio Codec과 Speech LLM** — SoundStream·Encodec의 RVQ, AudioLM의 hierarchical generation, VALL-E, Moshi의 full-duplex

**타겟 독자**:
- MFCC를 쓰는데 **왜 log-mel 이후 DCT**가 추가되는지 decorrelation 관점에서 유도 못하는 사람
- CTC를 쓰지만 **$\mathcal{B}$ mapping (remove blank and consecutive duplicates)** 연산과 forward variable $\alpha_t(s)$의 recursion을 유도 못하는 사람
- Whisper를 쓰지만 **weakly-supervised 680K hours**와 기존 ASR의 LibriSpeech 960hrs 차이가 왜 robustness에 결정적인지 모르는 사람
- RVQ codec에서 **codebook 수 $K$와 codebook 크기 $V$의 $K \log V$ bits 증명**과 commitment loss의 수학을 모르는 사람
- AudioLM의 **semantic token vs acoustic token**의 hierarchical 분리와 각각 다른 codec에서 나오는 이유를 모르는 사람

**선행 학습**:
- **Functional Analysis Deep Dive** (Fourier, Hilbert space) — **필수**
- **Probability Theory Deep Dive** (joint/conditional) — **필수**
- **Transformer Deep Dive** (attention, encoder-decoder) — **필수**
- **Information Theory Deep Dive** (rate-distortion, quantization) — **필수**
- **RNN & LSTM Deep Dive** (sequence model 기초) — 권장
- **LLM Pretraining Deep Dive** (scale의 효과) — 권장 (Whisper)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Signal Processing Foundations (6개 문서)
- **Continuous vs Discrete Signals** — Continuous $x(t)$의 Fourier transform, Nyquist-Shannon sampling theorem ($f_s > 2 f_{\max}$), aliasing, anti-aliasing filter, 16kHz vs 44.1kHz 선택 근거
- **DFT와 FFT의 수학** — $X[k] = \sum_{n=0}^{N-1} x[n] e^{-j 2\pi kn/N}$, Cooley-Tukey FFT $O(N \log N)$, power spectrum, phase
- **Short-Time Fourier Transform (STFT)** — $X(m, \omega) = \sum_n x[n] w[n-mH] e^{-j\omega n}$, window $w$ (Hann, Hamming), hop size $H$, overlap, spectrogram
- **Heisenberg Uncertainty in Signal Processing** — $\Delta t \cdot \Delta f \geq \frac{1}{4\pi}$, window length의 time-frequency trade-off, 짧은 window → 좋은 time resolution 하지만 나쁜 frequency
- **Mel-Scale과 심리음향학** — $m = 2595 \log_{10}(1 + f/700)$, cochlea의 basilar membrane tonotopic organization, mel filter bank (triangular weights)
- **MFCC — Mel-Frequency Cepstral Coefficients** — Log-mel 후 DCT로 decorrelation, 왜 DCT-II가 선택되는지, Homomorphic processing (log domain에서 source-filter 분리)

### Chapter 2: Classical Speech Processing과 HMM (4개 문서)
- **Source-Filter Model과 LPC** — Vocal cord (source) + vocal tract (filter), Linear Predictive Coding $x[n] = \sum_{k=1}^p a_k x[n-k]$, autocorrelation method, GMM-HMM ASR의 기반
- **HMM for Speech Recognition** — Phoneme hidden states, GMM observation, Viterbi decoding $O(T \cdot N^2)$, Baum-Welch training, pre-deep-learning ASR의 주류
- **WFST — Weighted Finite-State Transducer** — HCLG composition (HMM ∘ Context ∘ Lexicon ∘ Grammar), decoding graph, Kaldi의 철학
- **Griffin-Lim Phase Reconstruction (1984)** — Magnitude spectrogram에서 phase 복원, iterative projection, vocoder의 고전적 해결

### Chapter 3: Neural ASR — CTC와 Attention (6개 문서)
- **DeepSpeech — End-to-End ASR (Hannun 2014)** — Spectrogram → CNN → BiRNN → CTC, language model fusion (beam search with LM), 이전 GMM-HMM 대비 혁신
- **CTC Loss의 유도 (Graves 2006)** — Output $\pi \in (\Sigma \cup \{\phi\})^T$, mapping $\mathcal{B}: \pi \to y$ (remove blank + consecutive duplicates), $P(y|x) = \sum_{\pi \in \mathcal{B}^{-1}(y)} \prod_t P(\pi_t | x)$
- **CTC Forward-Backward Algorithm** — Extended output $y' = (\phi, y_1, \phi, y_2, \ldots, y_U, \phi)$, $\alpha_t(s) = \sum_{\pi: \pi_t = y'_s} \prod_{t'=1}^t P(\pi_{t'})$, recursion $\alpha_t(s) = y'_s \cdot (\alpha_{t-1}(s) + \alpha_{t-1}(s-1) + [\text{if valid}] \alpha_{t-1}(s-2))$
- **CTC Decoding — Greedy vs Beam Search** — Greedy: per-frame $\arg\max$, beam search with LM fusion, WFST-based decoding, blank skipping
- **Listen Attend Spell (LAS, Chan 2016)** — Encoder: BiLSTM pyramidal, Decoder: attention-based, end-to-end seq2seq, CTC 없이도 작동
- **RNN-Transducer (Graves 2012)** — CTC + autoregressive, streaming 가능, $P(y|x) = \sum_{\pi} \prod_t P(\pi_t | x_{1:t}, y_{1:\tau(\pi, t)-1})$, online ASR 표준

### Chapter 4: Modern ASR — Conformer와 Whisper (5개 문서)
- **Conformer (Gulati 2020)** — Conv + Transformer 결합, half-step residual, macaron-style FFN, LibriSpeech SOTA, 현대 ASR 표준 backbone
- **Squeezeformer, E-Branchformer** — Conformer 후속, downsampling strategy, efficient architecture, streaming 최적화
- **Wav2Vec 2.0 (Baevski 2020)** — CNN encoder + Transformer, **contrastive loss with masked latent representations**, $L = -\log \frac{\exp(\text{sim}(c_t, q_t)/\kappa)}{\sum_{\tilde{q}} \exp(\text{sim}(c_t, \tilde{q})/\kappa)}$, VQ-codebook
- **HuBERT (Hsu 2021)** — K-means on MFCC → pseudo-labels, iteratively refine with learned features, masked prediction like BERT, 다양한 downstream task transfer
- **Whisper (Radford 2023)** — 680K hours weakly-supervised (internet data), encoder-decoder Transformer, multitask format ([SOT, lang, task, timestamp, text, EOT]), robustness to noise·accent

### Chapter 5: Text-to-Speech (TTS) (5개 문서)
- **Tacotron (Wang 2017, Shen 2018)** — Text → mel-spectrogram → vocoder, attention-based seq2seq, stop token prediction, Location-sensitive attention
- **FastSpeech (Ren 2019)** — Non-autoregressive, parallel mel generation, length regulator (duration predictor), teacher forcing from autoregressive teacher
- **WaveNet (van den Oord 2016)** — Dilated causal convolution, $O(\log T)$ receptive field, autoregressive sample-level generation ($\mu$-law quantized), 첫 high-quality neural vocoder
- **HiFi-GAN (Kong 2020)** — GAN-based vocoder, multi-period + multi-scale discriminator, real-time synthesis, 현대 TTS의 표준 vocoder
- **VITS (Kim 2021) — End-to-End TTS** — VAE + Flow + GAN 통합, variational inference with monotonic alignment search, text-to-waveform 직접

### Chapter 6: Neural Audio Codec과 Vector Quantization (5개 문서)
- **Vector Quantization의 기본** — Codebook $\{e_1, ..., e_K\}$, VQ $q(x) = e_{k^*}$ where $k^* = \arg\min_k \|x - e_k\|$, commitment loss $L_{\text{VQ}} = \|sg[x] - e_{k^*}\|^2 + \beta \|x - sg[e_{k^*}]\|^2$
- **VQ-VAE (van den Oord 2017)** — Encoder → VQ → Decoder, straight-through estimator, codebook learning via EMA, discrete latent의 장점
- **Residual Vector Quantization (RVQ) in SoundStream (Zeghidour 2021)** — $K$-stage residual: $r_k = x - \sum_{i<k} q_i$, 각 stage는 이전 residual을 VQ, exponential 표현력 $V^K$, bitrate scaling
- **Encodec (Défossez 2022)** — RVQ codec with adversarial loss, 24kHz·48kHz, multi-scale discriminator, streaming 가능, Meta의 오디오 압축
- **Stream·Causal Codec과 Semantic Tokens** — Streaming codec (causal conv), semantic token (w2v-BERT 기반) vs acoustic token (RVQ 기반)의 분리, AudioLM 계보의 기반

### Chapter 7: Speech LLM과 Audio Generation (5개 문서)
- **AudioLM (Borsos 2023)** — Hierarchical: semantic tokens → coarse acoustic → fine acoustic, 각 stage별 Transformer autoregressive, long-term coherent audio generation
- **VALL-E (Wang 2023)** — Neural codec LM, 3-second prompt로 voice cloning, in-context learning for TTS, zero-shot speaker
- **MusicLM, MusicGen** — Music generation with hierarchical modeling, text conditioning, MusicLM (Google)의 MuLan joint embedding, MusicGen (Meta)의 single-stage
- **Moshi (Kyutai 2024)** — Full-duplex speech conversation, semantic + acoustic token 병렬, inner monologue stream, real-time interaction
- **Multimodal Audio — Qwen-Audio, GPT-4o audio, NotebookLM** — Audio-aware LLM, speech-to-speech, native multimodal audio, 2024+ frontier

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **36개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 기법이 오디오 처리의 핵심인가
## 📐 수학적 선행 조건 (FA, Prob, Transformer, Info 참조)
## 📖 직관적 이해
   — Spectrogram 시각화, audio 파형
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — STFT uncertainty, CTC forward-backward, RVQ rate
## 💻 PyTorch 구현 검증
   — STFT·Mel 바닥부터
   — CTC forward-backward 직접
   — 작은 RVQ codec 구현
## 🔗 실전 활용
   — Whisper로 transcription, TTS, codec 압축
## ⚖️ 가정과 한계
   — Stationarity, noise, low-resource language
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Spectrogram 시각화** — 음성·음악의 spectrogram, mel-spectrogram, log 차이
2. **CTC alignment visualization** — 훈련된 CTC 모델의 α matrix heatmap
3. **Window function 효과** — Hann vs Rectangular의 spectral leakage 비교
4. **RVQ stage별 reconstruction** — 각 stage에서 audio quality 향상 청취
5. **Whisper transcription** — 다양한 noise 조건에서 성능
6. **Attention visualization** — Tacotron의 text-to-audio alignment

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
torch==2.1.0
torchaudio==2.1.0
librosa==0.10.1         # 오디오 처리
soundfile==0.12.1
transformers==4.36.0    # Whisper
openai-whisper==20231117
matplotlib==3.8.0
IPython==8.20.0         # 오디오 재생
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (STFT + CTC + RVQ 구현)
import numpy as np
import torch
import torch.nn as nn
import torch.nn.functional as F
import matplotlib.pyplot as plt

# 1. STFT 바닥부터
def stft(x, n_fft=512, hop_length=256, window='hann'):
    """
    X(m, ω) = Σ x[n] w[n - mH] exp(-j ω n)
    """
    N = len(x)
    w = np.hanning(n_fft) if window == 'hann' else np.ones(n_fft)
    n_frames = 1 + (N - n_fft) // hop_length
    X = np.zeros((n_fft // 2 + 1, n_frames), dtype=complex)
    for m in range(n_frames):
        start = m * hop_length
        frame = x[start:start + n_fft] * w
        X[:, m] = np.fft.rfft(frame)
    return X

# Verify with librosa
# x, sr = librosa.load('audio.wav', sr=16000)
# X_ours = stft(x)
# X_lib = librosa.stft(x, n_fft=512, hop_length=256)

# 2. Mel-scale filter bank
def mel_filter_bank(n_mels=80, n_fft=512, sr=16000, f_min=0, f_max=8000):
    """
    Mel-scale: m = 2595 · log10(1 + f/700)
    Triangular filters equally spaced in mel-scale
    """
    def hz_to_mel(f): return 2595 * np.log10(1 + f / 700)
    def mel_to_hz(m): return 700 * (10**(m / 2595) - 1)
    
    mel_min = hz_to_mel(f_min)
    mel_max = hz_to_mel(f_max)
    mel_points = np.linspace(mel_min, mel_max, n_mels + 2)
    hz_points = mel_to_hz(mel_points)
    
    bin_points = np.floor((n_fft + 1) * hz_points / sr).astype(int)
    
    filters = np.zeros((n_mels, n_fft // 2 + 1))
    for m in range(1, n_mels + 1):
        left, center, right = bin_points[m-1], bin_points[m], bin_points[m+1]
        for k in range(left, center):
            filters[m-1, k] = (k - left) / (center - left + 1e-10)
        for k in range(center, right):
            filters[m-1, k] = (right - k) / (right - center + 1e-10)
    return filters

# 3. CTC Loss forward algorithm (NumPy)
def ctc_forward(log_probs, targets, blank=0):
    """
    log_probs: [T, V] log probability
    targets: [U] target sequence (no blank)
    
    Extended target: y' = [blank, y_1, blank, y_2, ..., y_U, blank]
    α_t(s) recursion
    """
    T, V = log_probs.shape
    U = len(targets)
    L = 2 * U + 1  # extended length
    
    # Extended target
    y_ext = [blank] * L
    for i, y in enumerate(targets):
        y_ext[2*i + 1] = y
    
    # Initialize log α
    log_alpha = np.full((T, L), -np.inf)
    log_alpha[0, 0] = log_probs[0, blank]
    if L > 1:
        log_alpha[0, 1] = log_probs[0, y_ext[1]]
    
    # Forward recursion
    def logsumexp(a, b):
        if a == -np.inf: return b
        if b == -np.inf: return a
        return max(a, b) + np.log(1 + np.exp(-abs(a - b)))
    
    for t in range(1, T):
        for s in range(L):
            # From same s or s-1
            prev = log_alpha[t-1, s]
            if s > 0:
                prev = logsumexp(prev, log_alpha[t-1, s-1])
            # Skip blank: if y_ext[s] != blank and y_ext[s] != y_ext[s-2]
            if s > 1 and y_ext[s] != blank and y_ext[s] != y_ext[s-2]:
                prev = logsumexp(prev, log_alpha[t-1, s-2])
            log_alpha[t, s] = prev + log_probs[t, y_ext[s]]
    
    # Sum over last two positions
    log_p = logsumexp(log_alpha[T-1, L-1], log_alpha[T-1, L-2])
    return -log_p, log_alpha

# Verify against PyTorch CTCLoss
T, V, U = 50, 10, 5
log_probs_t = torch.randn(T, 1, V).log_softmax(-1)
targets_t = torch.tensor([1, 2, 3, 4, 5])
input_lens = torch.tensor([T])
target_lens = torch.tensor([U])
loss_pt = F.ctc_loss(log_probs_t, targets_t, input_lens, target_lens, blank=0)

loss_ours, _ = ctc_forward(log_probs_t.squeeze(1).numpy(), targets_t.numpy())
print(f'PyTorch CTC: {loss_pt.item():.4f}, Ours: {loss_ours:.4f}')

# 4. RVQ — Residual Vector Quantization
class ResidualVQ(nn.Module):
    def __init__(self, dim, codebook_size, n_stages):
        super().__init__()
        self.codebooks = nn.ModuleList([
            nn.Embedding(codebook_size, dim) for _ in range(n_stages)
        ])
        self.n_stages = n_stages
    
    def forward(self, x):
        """
        x: [B, D]
        r_k = x - Σ_{i<k} q_i(r_i)
        """
        residual = x
        quantized_total = torch.zeros_like(x)
        indices_all = []
        losses = 0
        
        for k in range(self.n_stages):
            codebook = self.codebooks[k].weight  # [K, D]
            # Find nearest codebook entry
            distances = (residual.unsqueeze(1) - codebook.unsqueeze(0)).pow(2).sum(-1)
            indices = distances.argmin(dim=-1)  # [B]
            quantized = self.codebooks[k](indices)  # [B, D]
            
            # Commitment loss
            losses = losses + ((quantized.detach() - residual)**2).mean()
            losses = losses + ((quantized - residual.detach())**2).mean()
            
            # Straight-through
            quantized = residual + (quantized - residual).detach()
            
            quantized_total = quantized_total + quantized
            residual = residual - quantized
            indices_all.append(indices)
        
        return quantized_total, indices_all, losses

# 5. Positional mel-spectrogram 시각화
def mel_spectrogram(x, sr=16000, n_fft=1024, hop=256, n_mels=80):
    X = stft(x, n_fft=n_fft, hop_length=hop)
    power = np.abs(X) ** 2
    filters = mel_filter_bank(n_mels, n_fft, sr)
    mel = filters @ power  # [n_mels, n_frames]
    log_mel = np.log(mel + 1e-10)
    return log_mel

# 6. Whisper inference (if installed)
# import whisper
# model = whisper.load_model("small")
# result = model.transcribe("audio.wav")
# print(result["text"])
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "FA, Prob, Transformer, Info 선행 필수" 명시
   - Signal Processing → Neural → Speech LLM 진화
   - Whisper, MusicGen, Moshi 등 현대 응용
3. **챕터별 문서 작성**: Signal Processing → Classical → CTC → Modern ASR → TTS → Codec → Speech LLM

---

## 📚 참고 자료

- **Speech and Language Processing** (Jurafsky & Martin) — ASR·speech 기본
- **Discrete-Time Signal Processing** (Oppenheim & Schafer)
- **Connectionist Temporal Classification** (Graves et al. 2006) — CTC
- **Sequence Transduction with RNN** (Graves 2012) — RNN-T
- **Deep Speech** (Hannun et al. 2014)
- **Listen Attend Spell** (Chan et al. 2016)
- **Tacotron 2** (Shen et al. 2018)
- **FastSpeech 2** (Ren et al. 2020)
- **WaveNet** (van den Oord et al. 2016)
- **HiFi-GAN** (Kong et al. 2020)
- **VITS** (Kim et al. 2021)
- **Conformer** (Gulati et al. 2020)
- **Wav2Vec 2.0** (Baevski et al. 2020)
- **HuBERT** (Hsu et al. 2021)
- **WavLM** (Chen et al. 2022)
- **Whisper: Robust Speech Recognition via Large-Scale Weak Supervision** (Radford et al. 2023)
- **SoundStream** (Zeghidour et al. 2021) — Neural codec with RVQ
- **Encodec** (Défossez et al. 2022)
- **AudioLM** (Borsos et al. 2023)
- **VALL-E** (Wang et al. 2023)
- **MusicLM** (Agostinelli et al. 2023)
- **MusicGen** (Copet et al. 2023)
- **Moshi** (Kyutai 2024)

---

## 💡 핵심 분석 대상

```
Audio & Speech의 지도

───── Signal Processing ─────

Continuous vs Discrete:
  Nyquist-Shannon: f_s > 2 f_max
  16 kHz speech: ~8 kHz band
  44.1/48 kHz music
  
  Aliasing 방지:
    Anti-aliasing LPF 먼저

DFT/FFT:
  X[k] = Σ x[n] exp(-j 2π kn/N)
  Cooley-Tukey O(N log N)
  Real-valued audio → rfft

STFT:
  X(m, ω) = Σ x[n] w[n-mH] exp(-jωn)
  Window w: Hann, Hamming
  Hop H: overlap amount
  
  Heisenberg:
    ΔtΔf ≥ 1/(4π)
    긴 window → 좋은 freq, 나쁜 time
    짧은 window → 반대

Mel-Scale:
  m = 2595 log10(1 + f/700)
  Cochlear basilar membrane
  Tonotopic organization
  Low freq에 dense, high에 sparse

Mel filter bank:
  Triangular filters
  Mel-scale에서 uniform spacing
  
MFCC:
  log-mel + DCT
  DCT는 decorrelation
  (covariance diagonalize)
  Source-filter 분리 (homomorphic)

Griffin-Lim:
  Magnitude → Phase 복원
  Iterative projection
  Vocoder의 고전적 방법

───── Classical ASR — HMM ─────

Source-Filter:
  Vocal cord (source) + tract (filter)
  LPC: x[n] = Σ a_k x[n-k]
  Autocorrelation method

HMM:
  Phoneme hidden states
  GMM observation
  Viterbi decoding O(T N²)
  Baum-Welch training

WFST (Kaldi):
  HCLG = HMM ∘ Context ∘ Lexicon ∘ Grammar
  Composition으로 decoding graph
  
Pre-deep learning 주류
→ Deep Speech가 대체

───── CTC ─────

Graves 2006:
  Input: T frames
  Output: π ∈ (Σ ∪ {φ})^T
  
  Mapping B:
    Remove consecutive duplicates
    Remove blanks
    Example:
      π = φ a a φ φ b b b φ
      B(π) = a b

Loss:
  P(y|x) = Σ_{π ∈ B^{-1}(y)} Π P(π_t | x)

Forward-Backward:
  y' = φ y_1 φ y_2 φ ... y_U φ
  α_t(s) = Σ_{valid paths} Π P
  
  Recursion:
    α_t(s) = y'_s · (α_{t-1}(s) 
                    + α_{t-1}(s-1)
                    + [skip blank] α_{t-1}(s-2))
  
  O(T · U) DP

Decoding:
  Greedy: per-frame argmax
  Beam search with LM

RNN-T (Graves 2012):
  CTC + autoregressive
  P(π_t | x_{1:t}, y_{<})
  Streaming 가능
  Online ASR 표준

───── Modern ASR ─────

LAS (Chan 2016):
  Pyramidal BiLSTM encoder
  Attention-based decoder
  No CTC 필요

Conformer (Gulati 2020):
  Conv + Transformer
  Macaron-style FFN
  LibriSpeech SOTA

Wav2Vec 2.0 (Baevski 2020):
  CNN encoder + Transformer
  Masked prediction
  Contrastive loss:
    L = -log exp(sim(c, q)/κ) 
        / Σ exp(sim(c, q̃)/κ)
  VQ codebook

HuBERT (Hsu 2021):
  K-means on MFCC → pseudo-labels
  Masked LM on pseudo-labels
  Iterative refinement

Whisper (Radford 2023):
  680K hours weakly-supervised
  (internet data, not clean)
  Encoder-decoder Transformer
  Multitask format:
    [SOT, lang, task, timestamp, text, EOT]
  Robustness: noise, accent, music

───── TTS ─────

Tacotron (2017/2018):
  Text → mel → vocoder
  Attention (location-sensitive)
  Autoregressive

FastSpeech (2019):
  Non-autoregressive
  Duration predictor
  Parallel gen

WaveNet (2016):
  Dilated causal conv
  Autoregressive sample-level
  μ-law quantization

HiFi-GAN (2020):
  GAN vocoder
  Multi-period + multi-scale disc
  Real-time

VITS (2021):
  End-to-end TTS
  VAE + Flow + GAN
  Monotonic Alignment Search

───── Neural Audio Codec ─────

Vector Quantization:
  q(x) = argmin ‖x - e_k‖
  Commitment loss:
    L = ‖sg[x] - e‖² + β ‖x - sg[e]‖²
  Straight-through estimator

VQ-VAE (van den Oord 2017):
  Encoder → VQ → Decoder
  EMA codebook update
  Discrete latent

RVQ (SoundStream, Zeghidour 2021):
  r_k = x - Σ_{i<k} q_i(r_i)
  Each stage quantizes residual
  
  Total: V^K effective codebook
  Bits: K log V
  Bitrate scalable (drop later stages)

Encodec (Défossez 2022):
  RVQ + adversarial loss
  Multi-scale discriminator
  Streaming 가능

Semantic vs Acoustic tokens:
  Semantic (w2v-BERT): content
  Acoustic (RVQ): pitch, timbre
  AudioLM 기반

───── Speech LLM ─────

AudioLM (Borsos 2023):
  Hierarchical:
    Semantic tokens → coarse acoustic
    → fine acoustic
  Each: Transformer AR
  Long coherent audio

VALL-E (Wang 2023):
  Neural codec LM
  3-sec prompt → voice cloning
  In-context learning for TTS
  Zero-shot speaker

MusicGen (Copet 2023):
  Single-stage Transformer
  Delay pattern for RVQ codebooks
  Text-conditioned music

Moshi (Kyutai 2024):
  Full-duplex conversation
  Semantic + acoustic 병렬
  Inner monologue stream
  Real-time

GPT-4o / Gemini audio:
  Native multimodal
  End-to-end audio-text
  Low latency

───── 레포 간 연결 ─────

Functional Analysis (Layer 0):
  Fourier transform
  Hilbert space (L²)
  
Probability (Layer 0):
  HMM forward-backward
  CTC DP = marginalization

Information Theory (Layer 0):
  Rate-distortion
  Codec bitrate
  RVQ = vector quantization

Transformer (Layer 3):
  Conformer, Whisper
  Speech LLM

RNN & LSTM (Layer 3):
  Tacotron, Wav2Vec precursor

LLM Pretraining (Layer 4-B):
  Whisper = LLM-scale speech

Generative Model (Layer 3):
  Neural vocoder (WaveNet)
  VITS (VAE + Flow + GAN)
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·유도 (3~4줄)
- 전체 문서 개수 확인 (36개 목표)
- Python + PyTorch + librosa + torchaudio 실험 환경
- FA, Prob, Transformer, Info 레포 참조 관계
- Whisper, Moshi, GPT-4o audio 등 현대 audio-AI의 수학적 기반

**준비됐으면 1단계 구조 설계부터 시작해줘!**
