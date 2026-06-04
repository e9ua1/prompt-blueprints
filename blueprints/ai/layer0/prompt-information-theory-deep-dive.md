# Information Theory Deep Dive 레포지토리 제작 프롬프트

나는 "Information Theory Deep Dive" 레포지토리를 만들려고 해.
Cross-entropy 손실을 쓰는 것과, "**엔트로피는 평균 최적 부호 길이**"라는 Shannon의 근본 메시지를 아는 것은 다르다.
KL-divergence를 수식으로 외우는 것과, **KL이 비대칭이고 $\geq 0$이라는 성질을 왜 만족하는지** Jensen으로 증명할 수 있는 것은 다르다.
VAE에서 ELBO를 보는 것과, 그것이 **Evidence − KL** 구조의 정보이론적 해석인 것을 아는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "정보는 '놀라움의 로그'다. 엔트로피·KL·상호정보량은 모두 이 하나의 아이디어에서 유도된다"

**핵심 차별화**:
1. **Shannon의 공리적 유도** — 왜 하필 $-\log p$인가를 정보 측도의 공리에서 유도
2. **KL-divergence를 ML 전체의 중심축으로** — MLE, VAE, GAN, Diffusion이 모두 KL 최소화로 통합됨
3. **Source Coding과 Channel Coding 정리** — Shannon의 두 정리 완전 증명 (적어도 AEP 기반 증명)
4. **정보 기하 맛보기** — KL이 비대칭 "거리"인 이유, Fisher 정보계량과의 관계 (Information Geometry 레포로 연결)

**타겟 독자**:
- Cross-Entropy 손실을 쓰지만 왜 $-\sum y_i \log \hat{y}_i$인지 정보이론적으로 설명 못하는 개발자
- KL-divergence가 왜 비대칭인지, 비대칭성이 어떤 상황에서 문제가 되는지 모르는 사람
- Mutual Information을 직관으로만 이해하고 ML의 InfoNCE, Representation Learning과 연결 못하는 사람
- GAN의 $\text{JS}$-divergence와 Wasserstein distance의 차이를 모르는 사람
- VAE의 ELBO를 "최적 하한"이라고 외우지만 왜 "하한"인지 유도하지 못하는 사람

**선행 학습**:
- **Probability Theory Deep Dive** (확률변수, 기댓값, Jensen 부등식) — **필수**
- **Linear Algebra Deep Dive** (다변수 정규분포의 KL 계산)

---

## 🎯 1단계: 전체 구조 설계

다음 6개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Shannon 엔트로피 — 공리적 정의 (6개 문서)
- **정보의 공리적 유도** — 연속성·가법성·단조성 3개 공리에서 $-\log p$가 유일함을 증명 (Shannon 1948의 유도)
- **엔트로피 $H(X)$의 정의와 성질** — $H(X) = -\sum p(x) \log p(x) \geq 0$, 최대 엔트로피는 균등분포 (Jensen)
- **결합·조건부·상호정보량** — $H(X,Y), H(X|Y), I(X;Y)$의 정의, $I(X;Y) = H(X) - H(X|Y)$ 증명
- **Chain Rule과 정보의 계층 구조** — $H(X_1,...,X_n) = \sum H(X_i | X_{<i})$, 조건부 엔트로피의 감소성
- **미분 엔트로피(Differential Entropy)** — 연속 확률변수의 엔트로피, 왜 "측도에 의존"하는 값이 되는가, 정규분포의 엔트로피
- **최대 엔트로피 분포** — 평균 고정 → 지수분포, 분산 고정 → 정규분포가 최대 엔트로피임을 라그랑주로 유도

### Chapter 2: KL-Divergence와 관련 측도 (6개 문서)
- **KL-divergence의 정의와 비음수성** — $D(p\|q) = \sum p \log(p/q) \geq 0$ Jensen으로 완전 증명, 등호 조건 $p = q$
- **KL의 비대칭성 — forward vs reverse KL** — $D(p\|q)$ vs $D(q\|p)$의 기하 (mean-seeking vs mode-seeking), VI의 reverse KL 선택 이유
- **JS-divergence와 대칭화** — JSD의 정의, 유한성, $\sqrt{JSD}$가 metric인 이유, GAN 원 논문의 JSD
- **f-divergence 일반론** — KL, JSD, Hellinger, Total Variation을 통합하는 $f$-divergence 프레임워크, 변분 표현
- **Wasserstein 거리(지구 이동 거리)** — Optimal Transport의 기초, $W_1$의 Kantorovich-Rubinstein 쌍대, WGAN의 수학적 기반
- **분포 간 거리의 선택 — 언제 어느 것을 쓸 것인가** — KL이 실패하는 상황(겹치지 않는 support), Wasserstein이 해결하는 이유

### Chapter 3: 상호정보량(Mutual Information) (5개 문서)
- **MI의 다각적 정의** — $I(X;Y) = H(X) - H(X|Y) = D(p_{XY} \| p_X p_Y)$, 세 정의 동치성 증명
- **Data Processing Inequality(DPI)** — $X \to Y \to Z$ 마르코프 사슬에서 $I(X;Z) \leq I(X;Y)$ 증명, 정보는 처리로 증가하지 않는다
- **Fano 부등식** — 오류 확률의 하한 $H(P_e) + P_e \log(|X|-1) \geq H(X|Y)$, 분류 불가능성의 정보이론적 경계
- **Continuous MI와 추정 문제** — 연속 변수의 MI, MINE(Mutual Information Neural Estimator), 샘플에서 MI를 추정하는 어려움
- **MI와 Representation Learning** — InfoMax 원리, InfoNCE 손실이 MI의 하한임을 유도, Contrastive Learning의 정보이론

### Chapter 4: Source Coding — 데이터 압축의 한계 (5개 문서)
- **Prefix Code와 Kraft 부등식** — 일의 해독 가능한 부호의 길이 제약, Kraft $\sum 2^{-l_i} \leq 1$ 증명
- **Huffman 부호와 최적성** — Huffman 알고리즘, 평균 길이의 최소성 증명
- **Shannon Source Coding Theorem** — $L^* \geq H(X)$ 및 $L^* < H(X) + 1$ 증명, 엔트로피가 압축의 이론적 한계
- **Asymptotic Equipartition Property(AEP)** — $-\frac{1}{n}\log p(X_1,...,X_n) \xrightarrow{p} H(X)$, 전형적 집합(typical set)의 크기가 $2^{nH}$
- **Arithmetic Coding과 실전 압축** — 엔트로피에 임의로 가까운 평균 길이, JPEG·PNG의 원리와 연결

### Chapter 5: Channel Coding — 신뢰성의 한계 (4개 문서)
- **채널 용량(Channel Capacity)** — $C = \max_{p(x)} I(X;Y)$, 이진 대칭 채널의 용량 계산
- **Shannon Channel Coding Theorem** — 오류 확률을 임의로 작게 줄이는 부호의 존재 조건 $R < C$, 증명 스케치 (random coding + AEP)
- **Converse의 증명** — $R > C$면 오류 확률이 0으로 가지 못함, Fano 부등식을 이용한 역정리
- **실전 오류 정정 부호 — Turbo, LDPC, Polar Code 맛보기** — 샤논 한계에 접근하는 현대 부호들, 딥러닝 기반 채널 부호

### Chapter 6: 정보이론의 AI/ML 응용 (6개 문서)
- **Cross-Entropy와 MLE의 정보이론적 해석** — $H(p, q) = H(p) + D(p \| q)$, cross-entropy 최소화 = KL 최소화 = MLE
- **ELBO의 정보이론적 분해** — VAE의 $\log p(x) = \text{ELBO} + D(q(z|x) \| p(z|x))$, 왜 "Evidence Lower Bound"인가
- **MDL 원리(Minimum Description Length)** — 모델 선택의 MDL 관점, Occam의 면도날의 정보이론적 정식화, 베이지안 선택과의 관계
- **Information Bottleneck** — Tishby의 IB 원리, 표현학습과의 연결, "Deep Learning and the Information Bottleneck Principle" 논문
- **Diffusion Model의 변분 한계** — DDPM의 ELBO가 왜 KL 합으로 분해되는가, Score Matching과 정보이론
- **Fisher Information과 정보 기하 입문** — Fisher 정보계량이 KL의 2차 근사, 통계다양체의 Natural Gradient (Information Geometry 레포로 연결)

---

각 챕터는 **4~6개 문서**로 구성해줘. 총 **32개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 AI에서 중요한가
## 📐 수학적 선행 조건 (Prob, LA 레포 참조)
## 📖 직관적 이해
   — "놀라움", "코드 길이"로 정보를 체감
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — Shannon의 두 정리, Kraft, Fano 등
## 💻 NumPy 구현/시뮬레이션
   — 엔트로피 계산, Huffman 부호, KL 수치 계산
   — 전형적 집합을 실제로 관찰
## 🔗 AI/ML 연결
   — Cross-Entropy, VAE ELBO, GAN, InfoNCE 등
## ⚖️ 가정과 한계
   — 이산·연속의 차이 (미분 엔트로피의 음수 가능성)
   — KL이 무한대가 되는 경우
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **Shannon의 정보 측도는 "코드 길이"로 반복 체화** — 엔트로피 계산할 때마다 "평균 최적 부호 길이" 해석 병행
2. **비대칭성 명시** — KL 등장할 때마다 "어느 쪽이 데이터, 어느 쪽이 모델인지" 확실히
3. **시뮬레이션** — AEP, 전형적 집합은 반드시 실제로 샘플링하여 확인
4. **변분 표현 강조** — 현대 ML에서 KL의 변분 하한(예: Donsker-Varadhan)이 핵심
5. **기호 통일** — $H$ 엔트로피, $D$ KL, $I$ MI, 로그 밑은 기본 자연로그 (nats 단위)

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
matplotlib==3.8.0
torch==2.1.0       # Neural MI estimation, InfoNCE
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (AEP — Typical Set 시각화)
import numpy as np
import matplotlib.pyplot as plt

# Bernoulli(p=0.3) 시퀀스 샘플링
p = 0.3
H = -p*np.log(p) - (1-p)*np.log(1-p)  # 엔트로피 (nats)

for n in [10, 100, 1000]:
    n_trials = 10000
    neg_log_probs = []
    for _ in range(n_trials):
        seq = np.random.binomial(1, p, size=n)
        k = seq.sum()
        logp = k*np.log(p) + (n-k)*np.log(1-p)
        neg_log_probs.append(-logp / n)  # -1/n * log p
    
    plt.hist(neg_log_probs, bins=50, alpha=0.5, label=f'n={n}', density=True)

plt.axvline(H, color='r', linestyle='--', label='H(X)')
plt.xlabel('-1/n log p(X_1,...,X_n)')
plt.ylabel('density')
plt.legend(); plt.title('AEP: -1/n log p concentrates at H(X)')
plt.show()
# n이 클수록 H(X)에 집중 — AEP의 시각화
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Probability Theory 필수 선행" 명시
   - "Information Geometry로 이어짐" 언급
   - ML 전체의 손실 함수가 정보이론으로 통합된다는 메시지 강조
3. **챕터별 문서 작성**: 엔트로피 공리 → KL → MI → Source → Channel → ML 응용

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Elements of Information Theory** (Cover & Thomas) — 표준 교과서
- **Information Theory, Inference, and Learning Algorithms** (MacKay) — ML 관점 입문
- **Shannon 1948, "A Mathematical Theory of Communication"** — 원전
- **Pattern Recognition and Machine Learning** (Bishop) Chapter 1.6 — ML에서의 정보 이론
- **Deep Learning and the Information Bottleneck Principle** (Tishby & Zaslavsky 2015)
- **Optimal Transport: Old and New** (Villani) — Wasserstein 이론

---

## 💡 핵심 분석 대상

```
정보의 공리적 유도 (Shannon 1948)
  사건 x의 정보량 I(x) = -log p(x)
    ├── 확률이 작을수록 정보량 크다 (놀라움)
    ├── 독립 사건: I(x,y) = I(x) + I(y)
    └── 연속성
        │
        ▼
엔트로피 H(X) = 𝔼[-log p(X)] = -Σ p(x) log p(x)
  ├── 0 ≤ H(X) ≤ log|X|
  ├── 최대: 균등분포 (Jensen)
  ├── 최소: 결정적 분포 (H=0)
  └── 해석: 평균 최적 부호 길이의 하한
        │
        ▼
다변수 엔트로피
  H(X,Y) = H(X) + H(Y|X) = H(Y) + H(X|Y)
  I(X;Y) = H(X) + H(Y) - H(X,Y)
         = H(X) - H(X|Y)
         = D(p_{XY} ‖ p_X · p_Y)
  
Chain Rule:
  H(X_1,...,X_n) = Σ H(X_i | X_1,...,X_{i-1})

KL-divergence
  D(p ‖ q) = Σ p(x) log(p(x)/q(x)) = 𝔼_p[log(p/q)]
  ├── ≥ 0 (Jensen + -log convex)
  ├── = 0 iff p = q (a.s.)
  ├── 비대칭: D(p‖q) ≠ D(q‖p)
  └── cross-entropy: H(p, q) = H(p) + D(p ‖ q)

Shannon's Two Theorems
  1. Source Coding: L* ≥ H(X), L* < H(X) + 1
     압축의 한계는 엔트로피
  2. Channel Coding: R < C면 오류 확률 → 0 가능
     전송의 한계는 채널 용량 C = max I(X;Y)

AEP (Asymptotic Equipartition Property):
  -1/n log p(X_1,...,X_n) →^p H(X)
  
  전형적 집합 A_ε^(n):
  ├── |A_ε^(n)| ≤ 2^{n(H+ε)}
  ├── ℙ(X^n ∈ A_ε^(n)) → 1
  └── 거의 모든 확률이 2^{nH}개 시퀀스에 집중

AI/ML 응용 (모두 KL 기반)
  ├── Cross-Entropy 손실 = D(p_데이터 ‖ q_모델) + 상수
  │     ⇒ 최소화 = KL 최소화 = MLE
  ├── VAE ELBO:
  │     log p(x) = ELBO + D(q(z|x) ‖ p(z|x))
  │            = 𝔼_q[log p(x,z)] - 𝔼_q[log q(z|x)] + D(q‖p)
  │     ELBO 최대화 = KL(q‖p) 최소화 + 데이터 가능도 최대화
  ├── GAN 원 이론: JS-divergence 최소화
  ├── WGAN: Wasserstein 거리 최소화
  ├── Diffusion ELBO: 여러 단계의 KL 합
  ├── InfoNCE (Contrastive): MI 하한
  │     L_NCE ≥ -log N + I(X;Y)
  ├── Dropout = 근사 베이지안 = KL + 가능도
  ├── MDL = -log p(D|M) - log p(M)
  │     베이지안 모델 선택의 정보이론적 등가
  └── Information Bottleneck:
      min I(X;Z) s.t. I(Z;Y) 일정
      representation의 효율 vs 충실도 트레이드오프

거리·divergence의 선택
  KL(p‖q) = ∞ when p>0 and q=0 somewhere
    → GAN의 mode collapse, supports not overlapping
  
  Wasserstein W_1(p,q):
    ├── Supports 겹치지 않아도 의미 있는 값
    ├── 경사가 smooth → WGAN 학습 안정
    └── Kantorovich-Rubinstein 쌍대: sup_{f 1-Lip} 𝔼_p f - 𝔼_q f

Fisher Information과 Information Geometry 접점
  I(θ) = 𝔼_θ[(∂log p/∂θ)²] = -𝔼[∂²log p/∂θ²]
  
  D(p_θ ‖ p_{θ+dθ}) ≈ ½ dθ^T I(θ) dθ
  
  ⇒ KL의 2차 근사가 Fisher 정보계량
  ⇒ 통계다양체의 Riemann 계량
  ⇒ Natural Gradient = 기울기에 I^{-1} 곱
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 정리·증명·ML 응용 (3~4줄)
- 전체 문서 개수 확인 (32개 목표)
- Python + NumPy 실험 환경
- Probability Theory 레포의 어떤 정리를 전제로 사용하는지 명시
- 후속 레포(Information Geometry, Generative Model, LLM Alignment)와의 연결점

**준비됐으면 1단계 구조 설계부터 시작해줘!**
