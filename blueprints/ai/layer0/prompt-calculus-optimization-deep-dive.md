# Calculus & Optimization Deep Dive 레포지토리 제작 프롬프트

나는 "Calculus & Optimization Deep Dive" 레포지토리를 만들려고 해.
`loss.backward()`를 호출하는 것과, 역전파가 야코비안의 연쇄 행렬곱이라는 **기하학적 본질**을 아는 것은 다르다.
Adam 옵티마이저를 쓰는 것과, "Adam은 볼록이 아닌 문제에서 수렴을 보장하지 않는다"는 사실을 반례로 알 수 있는 것은 다르다.
`∇` 기호를 외우는 것과, 방향도함수가 어떻게 기울기와 연결되는지 코시-슈바르츠로 증명할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "신경망 학습은 고차원 공간에서의 기울기 기반 최적화다 — 그 수학을 처음부터 직접 유도한다"

**핵심 차별화**:
1. **ε-δ부터 시작한다** — 극한·미분·연속의 엄밀한 정의에서 출발
2. **역전파를 수식으로 유도한다** — "체인룰을 쓰면 된다"가 아니라, 야코비안 곱의 역순 계산이 forward 1회 + backward 1회인 이유를 증명
3. **옵티마이저의 수렴 증명** — SGD, Momentum, Adam의 수렴 조건·반례까지
4. **NumPy로 Autograd 구현** — PyTorch 없이 스칼라·벡터 Autograd 엔진을 직접 만들어 본다

**타겟 독자**:
- `.backward()` 호출 후 gradient가 어떻게 채워지는지 수학적으로 설명 못하는 개발자
- SGD, Adam, RMSProp이 어떤 수학적 가정 아래 수렴하는지 모르는 연구자
- 헤시안이 왜 "곡률"이고, 뉴턴 방법이 왜 2차 수렴인지 증명 못하는 사람
- 라그랑주 승수법을 쓰지만 왜 $\nabla f = \lambda \nabla g$에서 $\lambda$가 "그림자 가격"인지 모르는 사람
- Vanishing Gradient의 수치적 근거(Jacobian 스펙트럼 반경)를 말하지 못하는 사람

**선행 학습**:
- **Linear Algebra Deep Dive** (야코비안, 헤시안은 행렬이다 — 고유값·스펙트럼 이해 필수)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 해석학의 기초 — 극한·연속·미분의 엄밀한 정의 (5개 문서)
- **ε-δ 언어** — 극한의 엄밀한 정의, 왜 직관적 "가까워진다"로는 부족한가, 수렴 증명 예제 5개
- **연속성과 균등연속성** — 각 점에서의 연속 vs 구간 전체 균등연속, 하이네-보렐 정리와 콤팩트성
- **미분의 정의와 선형근사** — $f'(a) = \lim \frac{f(a+h)-f(a)}{h}$가 "최선의 선형근사"인 이유의 엄밀한 증명
- **평균값 정리(MVT)와 테일러 정리** — Rolle → MVT → Taylor 순서로 완전 유도, 여러 여분항(Lagrange, Cauchy, 적분형) 비교
- **미분 가능 함수의 성질** — 미분가능 ⇒ 연속, 역은 거짓인 반례(Weierstrass 함수 스케치)

### Chapter 2: 다변수 미적분 — AI가 사는 고차원 공간 (6개 문서)
- **편미분과 방향도함수** — 편미분이 특정 방향 도함수의 특수 경우임을 증명, 모든 방향 미분 가능이 전미분 가능과 동치가 아닌 반례
- **전미분(Total Derivative)과 선형사상** — $f: \mathbb{R}^n \to \mathbb{R}^m$의 미분이 선형사상임을 정의하고, 행렬 표현이 야코비안
- **Gradient와 코시-슈바르츠** — $\nabla f \cdot v$의 최대값이 $\|\nabla f\|$인 이유(코시-슈바르츠 등호 조건), "Gradient는 최대 증가 방향"의 엄밀한 증명
- **야코비안과 헤시안** — 야코비안은 1차 선형근사, 헤시안은 2차 곡률, $f: \mathbb{R}^n \to \mathbb{R}$의 테일러 2차 전개
- **연쇄법칙(Chain Rule)의 일반화** — 다변수 연쇄법칙의 행렬곱 표현 $J_{f\circ g} = J_f(g(x)) \cdot J_g(x)$, 증명과 의미
- **음함수 정리(Implicit Function Theorem)** — 자코비안 가역성 조건, 제약 있는 최적화에서 왜 필요한가

### Chapter 3: 다변수 테일러 전개와 2차 근사의 기하 (4개 문서)
- **테일러 정리의 다변수 일반화** — $f(x+h) = f(x) + \nabla f(x)^\top h + \frac{1}{2} h^\top H(x) h + o(\|h\|^2)$ 완전 유도
- **헤시안의 고유값과 국소 기하** — 헤시안이 양의 정부호/음의 정부호/부정부호일 때 국소 최소/최대/안장점, Spectral Theorem 활용
- **안장점과 볼록성 판정** — 제2계 충분조건 증명, 왜 대부분의 딥러닝 Loss Landscape은 "안장점 투성이"인가(Dauphin 2014)
- **조건수의 의미** — 헤시안의 $\kappa(H) = \lambda_{\max}/\lambda_{\min}$가 최적화 속도에 미치는 영향, 등고선이 "타원"인 이유

### Chapter 4: 기울기 기반 최적화 — 경사하강법 완전 분석 (7개 문서)
- **경사하강법의 수렴 — 볼록·매끄러운 경우** — $L$-smooth 조건 하에서 $f(x_k) - f^* \leq O(1/k)$ 증명 (Nesterov 책 Thm 2.1.5)
- **학습률의 역할** — 너무 크면 발산, 너무 작으면 느림, $\eta < 2/L$가 수렴 조건인 이유 증명
- **모멘텀(Momentum)과 Nesterov Acceleration** — 가속의 $O(1/k^2)$ 수렴률, 왜 $\sqrt{\kappa}$ 의존성이 개선되는가
- **확률적 경사하강법(SGD)의 수렴** — 기댓값 관점의 $\mathbb{E}[f(x_k)] - f^* \leq O(1/\sqrt{k})$ 증명, 학습률 스케줄링
- **Adam, RMSProp, AdaGrad의 수학** — 각각의 업데이트 식 유도, Adam의 수렴 보장 반례 (Reddi 2018 "On the Convergence of Adam")
- **2차 방법 — 뉴턴 방법과 Quasi-Newton** — 뉴턴 방법의 2차 수렴성 증명, L-BFGS의 Hessian 근사 원리
- **Loss Landscape의 기하** — 딥러닝에서 왜 극소값이 "대부분 거의 같은 품질"인가, Mode Connectivity 현상

### Chapter 5: Backpropagation — 미적분의 걸작 (5개 문서)
- **계산 그래프(Computational Graph)와 자동미분** — 순방향·역방향 자동미분의 정의, 왜 역방향이 더 효율적인가 (출력이 스칼라일 때)
- **역전파의 수학적 유도** — 연쇄법칙을 뒤에서 앞으로 적용하는 것이 Vector-Jacobian Product 연속 계산임을 증명
- **왜 순전파 1회 + 역전파 1회로 모든 파라미터 gradient를 얻는가** — 계산량이 forward의 2~3배로 제한되는 이유 (Baur-Strassen 정리)
- **수치 안정성과 Vanishing/Exploding Gradient** — 야코비안의 스펙트럼 반경이 1보다 크면 폭발, 작으면 소실되는 수학적 근거
- **NumPy로 Autograd 엔진 구현** — 스칼라 Autograd → 벡터 Autograd → 텐서 Autograd 단계적 구현, PyTorch `.backward()`와 결과 일치 검증

### Chapter 6: 제약 최적화 — 라그랑주와 KKT (5개 문서)
- **등식 제약과 라그랑주 승수법** — $\nabla f = \lambda \nabla g$의 기하학적 유도, 왜 $\lambda$가 "제약을 한 단위 완화했을 때의 목적함수 변화율"인가
- **부등식 제약과 KKT 조건** — KKT 4개 조건(stationarity, primal feasibility, dual feasibility, complementary slackness) 도출
- **라그랑주 쌍대성의 맛보기** — 원시·쌍대 문제 정의, 약쌍대성 $g(\lambda) \leq f^*$ 증명 (Convex Optimization 레포에서 완전 전개)
- **제약 최적화와 음함수 정리의 연결** — 제약 곡면 위의 접평면이 $\ker(Dg)$임을 음함수 정리로 증명
- **AI 응용: Constrained GAN, Lagrangian Neural Network** — 물리 법칙을 제약으로 넣는 PINN·LNN 구조의 수학

### Chapter 7: AI/ML에서의 미적분·최적화 (5개 문서)
- **Softmax의 야코비안** — 교차 엔트로피 + Softmax 조합이 왜 gradient가 깔끔한 $(p - y)$ 형태가 되는가 유도
- **Batch/Layer Normalization의 미분** — 평균·분산을 통한 미분이 어떻게 계산되는가, 역전파 시 추가되는 항들
- **Neural Tangent Kernel (NTK) 맛보기** — 무한 폭 신경망의 학습이 선형 회귀로 환원되는 이유, 야코비안이 고정되는 극한
- **Meta-Learning과 고차 미분** — MAML의 "gradient of gradient", 2차 미분이 필요한 이유와 계산 비용
- **Implicit Differentiation in Deep Learning** — Deep Equilibrium Models, Optimization as a Layer, 음함수 정리의 실전 응용

---

각 챕터는 **4~7개 문서**로 구성해줘. 총 **37개 문서** 목표.

**문서 구조** (미적분·최적화의 증명과 구현을 병행):
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 정리가 AI에서 중요한가
## 📐 수학적 선행 조건 (LA 레포 참조 링크)
## 📖 직관적 이해 (그림 + 물리적 비유)
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — 보조정리(Lemma)부터 차근차근
   — "자명하다"로 생략 금지
## 💻 NumPy / SymPy 구현으로 검증
   — 기호 미분(SymPy)과 수치 미분 비교
   — 자동 미분 직접 구현
## 🔗 AI/ML 연결
   — 역전파·옵티마이저·정규화 등 구체 사례
## ⚖️ 가정과 한계
   — 수렴 정리의 가정이 깨지는 순간
   — 수치적 불안정성 조건
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **미분을 "기호 조작"으로 다루지 않는다** — 정의에서 출발해 증명
2. **수렴 정리에는 반드시 반례 제시** — 가정이 깨지면 어떻게 발산하는지 실제 시뮬레이션
3. **PyTorch 없이 Autograd 구현** — 체인룰이 왜 자동화되는지를 직접 만들어서 체득
4. **시각화 필수** — Loss Landscape, 등고선, 경사하강 궤적을 matplotlib으로 플롯
5. **수치 미분 vs 해석 미분 비교** — 유한차분의 오차를 실험적으로 확인

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
matplotlib==3.8.0
sympy==1.12           # 기호 미분
jax==0.4.20           # 자동미분 비교용 (직접 구현 결과 검증)
torch==2.1.0          # 같은 목적 (PyTorch와 일치 확인)
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (Vanishing Gradient 재현)
import numpy as np
import matplotlib.pyplot as plt

# 깊은 네트워크의 야코비안 곱
def simulate_depth_gradient(depth, sigma_w=1.0):
    """각 층의 가중치 야코비안을 곱해가며 스펙트럼 반경 관찰"""
    J = np.eye(100)
    spectral_radii = []
    for l in range(depth):
        W = np.random.randn(100, 100) * sigma_w / np.sqrt(100)
        J = W @ J
        spectral_radii.append(np.linalg.norm(J, 2))
    return spectral_radii

# σ_w = 0.5 → vanishing, σ_w = 2.0 → exploding
# σ_w = 1.0 (적절한 초기화) → 안정
for s in [0.5, 1.0, 2.0]:
    sr = simulate_depth_gradient(depth=100, sigma_w=s)
    plt.plot(sr, label=f'σ_w={s}')
plt.yscale('log')
plt.xlabel('Depth'); plt.ylabel('Spectral norm of accumulated Jacobian')
plt.legend(); plt.show()
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "Linear Algebra 선행" 명시
   - "Convex Optimization으로 이어짐" 명시
   - 역전파·옵티마이저 수렴 분석이 핵심 차별점임을 강조
3. **챕터별 문서 작성**: Chapter 1부터 순서대로, 각 문서 2500~3500 단어 + 코드 + 그림

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Principles of Mathematical Analysis** (Rudin) — ε-δ, 연속, 미분의 표준
- **Convex Optimization** (Boyd & Vandenberghe) — 제약 최적화와 쌍대성의 고전 (다음 레포에서 심화)
- **Numerical Optimization** (Nocedal & Wright) — Quasi-Newton, L-BFGS
- **Introductory Lectures on Convex Optimization** (Nesterov) — 수렴률 분석의 교과서
- **Deep Learning** (Goodfellow) Chapter 4, 8 — 수치 방법과 최적화
- **Automatic Differentiation in Machine Learning** (Baydin 2018) — AD 이론

---

## 💡 핵심 분석 대상

```
해석학 기초 (ε-δ)
  │
  ▼
단변수 미분 → 다변수 편미분 → 방향도함수 → Gradient
  │
  ▼
다변수 테일러 전개
  f(x+h) ≈ f(x) + ∇f·h + ½ h^T H h + o(‖h‖²)
  │
  ├── H > 0 → 국소 최소 (볼록)
  ├── H < 0 → 국소 최대
  └── H 부정부호 → 안장점 (딥러닝의 대부분!)
        │
        ▼
기울기 기반 최적화
  ├── GD: x_{k+1} = x_k - η∇f(x_k)
  │     수렴률 (L-smooth, 볼록): O(1/k)
  │     (L-smooth, μ-strongly convex): O((1-μ/L)^k)
  ├── Momentum: velocity 항 추가
  ├── Nesterov: 미리 한 스텝 앞에서 gradient 계산
  ├── SGD: 확률적 추정치 사용, O(1/√k)
  ├── Adam: 1·2차 모멘트 추정 + 바이어스 보정
  │     ※ 반례: 볼록 문제에서도 수렴 실패할 수 있음
  └── Newton: x_{k+1} = x_k - H^{-1}∇f, 2차 수렴

역전파 = 역방향 자동미분
  Forward:  x → h_1 → h_2 → ... → y → L
  Backward: ∂L/∂y ← ∂L/∂h_n ← ... ← ∂L/∂x
           (각 단계는 야코비안-벡터 곱)
  
  계산량: Forward의 2~3배 (Baur-Strassen)
  메모리: 중간 활성값 저장 필요
        → Checkpointing, Reversible Net으로 절약

Vanishing/Exploding Gradient 수학:
  ∂L/∂x = ∏_l J_l  (층별 야코비안의 곱)
  
  spectral radius ρ(J):
    ρ(J) < 1 → 기하급수적 소실
    ρ(J) > 1 → 기하급수적 폭발
    ρ(J) = 1 → 안정 (He/Xavier 초기화의 목표)
  
  해결:
  ├── Residual Connection (항등 사상 + 야코비안)
  ├── Batch/Layer Norm (조건수 개선)
  ├── Gradient Clipping (‖g‖ 제한)
  └── 초기화 설계 (He, Xavier)

제약 최적화:
  minimize f(x) s.t. g_i(x) ≤ 0, h_j(x) = 0
  
  Lagrangian: L(x, λ, μ) = f + Σ λ_i g_i + Σ μ_j h_j
  
  KKT 조건:
  ├── ∇_x L = 0         (stationarity)
  ├── g_i(x*) ≤ 0       (primal feasibility)
  ├── λ_i ≥ 0           (dual feasibility)
  └── λ_i g_i(x*) = 0    (complementary slackness)
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~7개씩)
- 각 문서가 다루는 핵심 정리·증명 (3~4줄)
- 전체 문서 개수 확인 (37개 목표)
- Python + NumPy + SymPy 실험 환경
- Linear Algebra 레포의 어떤 정리를 전제로 사용하는지 명시
- 후속 레포(Convex Optimization, Probability Theory)와의 연결점

**준비됐으면 1단계 구조 설계부터 시작해줘!**
