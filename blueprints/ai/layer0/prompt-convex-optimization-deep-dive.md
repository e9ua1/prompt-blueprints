# Convex Optimization Deep Dive 레포지토리 제작 프롬프트

나는 "Convex Optimization Deep Dive" 레포지토리를 만들려고 해.
경사하강법을 쓰는 것과, "**볼록 함수라면 국소 최적 = 전역 최적**"이라는 전역성 보장의 근거를 아는 것은 다르다.
SVM의 Lagrange dual을 푸는 것과, **Slater 조건 하에서 강쌍대성이 성립**하는 이유를 증명할 수 있는 것은 다르다.
ADMM을 쓰는 것과, **Proximal Operator가 왜 경사하강의 일반화**인지 유도할 수 있는 것은 다르다.
이 모든 "다름"을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "볼록성(convexity)은 최적화에 전역성을 부여하는 유일한 구조다"

**핵심 차별화**:
1. **볼록 집합·함수의 공리적 정의에서 시작** — "epigraph가 볼록 집합"이라는 기하 정의
2. **쌍대 이론을 완전 유도** — Lagrange dual, Slater 조건, KKT 필요충분성 증명
3. **Proximal 방법 체계화** — Proximal Operator, Proximal Gradient, ADMM의 수학적 통합
4. **ML의 "잘못 알려진" 볼록성** — "딥러닝은 볼록이 아니다"라는 사실의 수학적 의미, 그럼에도 왜 작동하는가

**타겟 독자**:
- SVM의 이차계획(QP)을 풀면서 쌍대 문제가 왜 더 풀기 쉬운지 모르는 개발자
- L1 regularization의 sparsity가 왜 기하학적으로 "axis-aligned" 해를 선호하는지 모르는 사람
- ADMM, proximal gradient, FISTA 같은 알고리즘의 수렴률을 비교·선택하지 못하는 연구자
- "딥러닝은 non-convex라서 이론이 안 통한다"는 말 뒤의 수학을 정확히 모르는 사람
- 강쌍대성(strong duality)이 SVM 쌍대에서 왜 성립하는지 Slater 조건으로 증명 못하는 사람

**선행 학습**:
- **Linear Algebra Deep Dive** (양의 정부호 행렬, 고유값 분해)
- **Calculus & Optimization Deep Dive** (경사하강 수렴 분석, 라그랑주 승수 기초)
- **Probability Theory Deep Dive** — (Stochastic optimization 파트에서 사용)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 볼록 집합(Convex Sets) (5개 문서)
- **볼록 집합의 정의와 예제** — 임의의 두 점을 잇는 선분이 집합에 포함, 초평면·반공간·다면체·노름 볼·양의 정부호 콘
- **볼록 집합의 연산** — 교집합·아핀변환·합성곱 등이 볼록성을 보존, 볼록 포(convex hull)의 정의
- **분리 초평면 정리(Separating Hyperplane Theorem)** — 두 서로소 볼록 집합은 초평면으로 분리 가능, 지지 초평면 정리
- **볼록 콘(Convex Cone)과 쌍대 콘** — 정부호 콘 $\mathbb{S}^n_+$, 이차 콘, 쌍대 콘의 기하학적 의미
- **Extreme Point와 Krein-Milman 정리** — 콤팩트 볼록 집합은 극점의 볼록 포, LP 최적해가 꼭짓점에 있는 이유

### Chapter 2: 볼록 함수(Convex Functions) (6개 문서)
- **볼록 함수의 3개 동치 정의** — Jensen 부등식, epigraph가 볼록 집합, 1차·2차 조건, 세 정의가 동치임을 증명
- **일계·이계 조건** — 미분가능하면 $f(y) \geq f(x) + \nabla f(x)^\top (y-x)$, 이계 미분가능이면 $\nabla^2 f \succeq 0$ 완전 증명
- **강볼록(Strong Convexity)과 매끄러움(Smoothness)** — $\mu$-strongly convex와 $L$-smooth의 정의, $\kappa = L/\mu$가 조건수
- **볼록 함수의 연산** — 비음 계수 가중합, 포인트와이즈 상한, 아핀 합성, 볼록성 보존 규칙 목록
- **Conjugate Function과 Legendre 변환** — $f^*(y) = \sup_x (y^\top x - f(x))$의 정의와 기하, 이중 공액의 닫힘성질 $f^{**} = f$ (f가 볼록·닫힘)
- **주요 볼록 함수 카탈로그** — 노름, 엔트로피, 로그지수합(log-sum-exp), 스펙트럴 함수, 각각의 공액 유도

### Chapter 3: 볼록 최적화 문제의 형태 (5개 문서)
- **볼록 최적화 문제의 표준형** — $\min f(x)$ s.t. $g_i(x) \leq 0$, $Ax = b$에서 $f, g_i$ 볼록, 등식 제약은 아핀
- **LP, QP, QCQP, SOCP, SDP의 계층** — 각각의 정의, 표현력과 해결 방법, 언제 어느 형태로 변환하는가
- **Geometric Programming** — GP의 표준형, log 변환으로 볼록화하는 기법
- **모델링 기법** — Epigraph trick, 관점 함수(perspective), 슬랙 변수 도입 등 non-convex → convex 변환
- **CVXPY로 문제 표현** — DCP (Disciplined Convex Programming) 규칙, 실제 라이브러리로 문제 푸는 법

### Chapter 4: 쌍대 이론(Duality) (6개 문서)
- **Lagrangian과 쌍대 함수** — $L(x, \lambda, \nu) = f_0 + \sum \lambda_i f_i + \sum \nu_i h_i$, 쌍대 함수 $g(\lambda, \nu) = \inf_x L$의 오목성
- **약쌍대성(Weak Duality)** — $d^* \leq p^*$는 언제나 성립, 비볼록 문제에서도 유효
- **Slater 조건과 강쌍대성(Strong Duality)** — $d^* = p^*$의 충분조건, Slater 조건 증명 (분리 초평면 정리 응용)
- **KKT 조건 — 필요충분조건으로서** — 볼록 문제에서 KKT = 최적성 조건, 필요충분성 완전 증명
- **쌍대 해석(Dual Interpretation)** — 쌍대 변수의 그림자 가격(shadow price) 의미, 제약 완화 시 최적값 변화
- **SVM의 쌍대 유도 — 완전판** — Hard-margin SVM의 primal → dual 전개, Support Vector의 상보적 여유(complementary slackness) 의미

### Chapter 5: 알고리즘 — 경사 기반과 2차 방법 (6개 문서)
- **경사하강법 수렴 정리 완전판** — $L$-smooth 볼록: $O(1/k)$, $\mu$-strongly convex: $O((1-\mu/L)^k)$, 각각 완전 증명
- **Nesterov 가속 경사법(AGM)** — $O(1/k^2)$ 수렴률, 가속이 왜 동작하는지 Estimating Sequence로 유도
- **하한 경계(Lower Bound)** — Nemirovski-Yudin의 lower bound, $O(1/k^2)$와 $O((1-\sqrt{\mu/L})^k)$가 최적인 이유
- **뉴턴 방법의 국소·전역 수렴** — 2차 수렴 조건, Damped Newton의 전역 수렴 보장
- **Interior Point Method** — 장벽 함수(barrier), 중앙 경로(central path), 로그 장벽의 자기일치성(self-concordance)
- **Stochastic 방법과 분산 감소** — SGD 수렴 분석, SVRG·SAG의 분산 감소 기법, $O(1/k)$ 회복 가능한 이유

### Chapter 6: Proximal 방법과 분해 알고리즘 (6개 문서)
- **Proximal Operator의 정의** — $\text{prox}_f(v) = \arg\min_x (f(x) + \frac{1}{2}\|x-v\|^2)$, 볼록이면 유일 존재 증명
- **주요 Proximal 연산** — Soft-thresholding(L1), Euclidean projection, 간단한 함수들의 proximal 닫힌 형태
- **Proximal Gradient Method(ISTA)와 FISTA** — 매끄럽지 않은 볼록 문제 $\min f + g$ 에서의 수렴률, FISTA의 $O(1/k^2)$
- **Lasso의 완전 풀이** — $\min \frac{1}{2}\|Ax-b\|^2 + \lambda \|x\|_1$을 ISTA/FISTA로 푸는 전 과정, sparsity 발생 기하
- **ADMM(Alternating Direction Method of Multipliers)** — Augmented Lagrangian + 분리 최적화, 수렴률과 분산 ML에의 응용
- **Douglas-Rachford, Primal-Dual Splitting** — 일반 분해 알고리즘 체계, 언제 어느 방법을 선택하는가

### Chapter 7: AI/ML에서의 볼록 최적화 (5개 문서)
- **Logistic Regression은 볼록이다** — 로그 가능도의 이계 미분이 양의 준정부호임을 증명, MLE의 전역 최적 보장
- **Support Vector Machine의 완전 유도** — Hard-margin + Soft-margin, 커널 트릭의 쌍대 관점 해석
- **Regularization의 기하 — L1 vs L2** — 등고선과 제약 영역의 접점에서 L1은 axis-aligned(sparsity) 해 선호, L2는 회전 대칭
- **딥러닝은 왜 비볼록인데 동작하는가** — Over-parameterization, Neural Tangent Kernel, Loss Landscape 기하, Mode Connectivity
- **Online Convex Optimization과 Regret 경계** — OCO 프레임워크, Online Gradient Descent의 $O(\sqrt{T})$ regret, AdaGrad의 적응형 경계

---

각 챕터는 **5~6개 문서**로 구성해줘. 총 **39개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 이론이 AI에서 중요한가
## 📐 수학적 선행 조건 (LA, Calc, Prob 레포 참조)
## 📖 직관적 이해
   — 2D·3D 그림으로 볼록성의 기하 직관
## ✏️ 엄밀한 정의
## 🔬 정리와 증명
   — 분리 초평면, KKT 필요충분성, 수렴률 등
## 💻 NumPy/CVXPY 구현
   — 직접 알고리즘 구현 + CVXPY로 검증
   — 수렴 곡선 그리기
## 🔗 AI/ML 연결
## ⚖️ 가정과 한계
   — 볼록성이 깨질 때, 강볼록이 아닐 때
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. **기하학적 시각화 필수** — 2D 볼록 집합, 등고선, 서포트 벡터 등 모두 matplotlib으로 그림
2. **알고리즘은 직접 구현** — ISTA, FISTA, ADMM을 NumPy로 구현 후 CVXPY 결과와 비교
3. **수렴률 실험적 검증** — "O(1/k^2)"가 실제로 관찰되는지 로그-로그 플롯으로 확인
4. **KKT 해석** — 모든 KKT 조건 문서에서 "이 조건이 깨지면 무엇이 최적성을 잃는가"를 반례로 보임
5. **ML 연결은 논문 수식과 대응** — SVM 논문의 식과 쌍대 유도 완전 일치

**실험 환경**:
```bash
# requirements.txt
numpy==1.26.0
scipy==1.11.0
cvxpy==1.4.0       # 볼록 최적화 DSL
matplotlib==3.8.0
scikit-learn==1.3.0  # SVM, LogReg 비교
cvxopt==1.3.0      # 내부점법 low-level
jupyter==1.0.0
```

```python
# 실험 템플릿 예시 (ISTA vs FISTA 수렴률 비교)
import numpy as np
import matplotlib.pyplot as plt

def soft_thresh(v, lam):
    return np.sign(v) * np.maximum(np.abs(v) - lam, 0)

def ista(A, b, lam, max_iter=500):
    x = np.zeros(A.shape[1])
    L = np.linalg.norm(A, 2)**2  # Lipschitz 상수
    losses = []
    for _ in range(max_iter):
        grad = A.T @ (A @ x - b)
        x = soft_thresh(x - grad/L, lam/L)
        losses.append(0.5*np.linalg.norm(A@x - b)**2 + lam*np.linalg.norm(x, 1))
    return x, losses

def fista(A, b, lam, max_iter=500):
    x = np.zeros(A.shape[1])
    y = x.copy()
    t = 1.0
    L = np.linalg.norm(A, 2)**2
    losses = []
    for _ in range(max_iter):
        grad = A.T @ (A @ y - b)
        x_new = soft_thresh(y - grad/L, lam/L)
        t_new = (1 + np.sqrt(1 + 4*t**2)) / 2
        y = x_new + ((t - 1)/t_new) * (x_new - x)
        x, t = x_new, t_new
        losses.append(0.5*np.linalg.norm(A@x - b)**2 + lam*np.linalg.norm(x, 1))
    return x, losses

# 수렴 그래프: FISTA는 O(1/k²), ISTA는 O(1/k)
# CVXPY로 정확한 최적값을 구해 loss gap을 로그-로그 플롯
```

---

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**
2. **README.md 작성**:
   - "LA, Calc & Opt 선행" 명시
   - "쌍대성과 KKT가 전체의 중심축" 메시지
   - 딥러닝이 비볼록임에도 왜 이 이론이 중요한지 서술
3. **챕터별 문서 작성**: 볼록 집합 → 볼록 함수 → 문제 → 쌍대 → 알고리즘 → proximal → ML

---

## 📚 참고 자료

README 작성 시 다음을 Reference로 포함해줘:
- **Convex Optimization** (Boyd & Vandenberghe) — 레포의 기본 뼈대, 무료 PDF 공개
- **Lectures on Convex Optimization** (Nesterov) — 수렴률 분석의 바이블
- **Proximal Algorithms** (Parikh & Boyd) — Proximal 방법 통합 참고서
- **First-Order Methods in Optimization** (Amir Beck)
- **Online Convex Optimization** (Elad Hazan) — OCO 이론
- Stanford EE364a/b 강의노트 (Boyd) — 부교재

---

## 💡 핵심 분석 대상

```
볼록성의 계층
  
  Convex Set (볼록 집합)
    ├── λx + (1-λ)y ∈ C for λ ∈ [0,1]
    └── 분리 초평면 정리 (Hahn-Banach의 유한차원 버전)
          │
          ▼
  Convex Function (볼록 함수) ↔ epigraph가 볼록 집합
    ├── Jensen: f(𝔼X) ≤ 𝔼f(X)
    ├── 1차: f(y) ≥ f(x) + ∇f(x)ᵀ(y-x)
    ├── 2차: ∇²f ≽ 0
    └── Strongly Convex (μ > 0)
          │
          ▼
  Convex Problem: min f(x) s.t. g_i(x) ≤ 0, Ax = b
    ├── LP ⊂ QP ⊂ QCQP ⊂ SOCP ⊂ SDP (표현력 계층)
    └── 국소 최적 = 전역 최적 (볼록성의 선물)
          │
          ▼
Duality Theory (쌍대 이론)
  Primal: p* = min f₀(x) s.t. f_i(x) ≤ 0, h_j(x) = 0
  Lagrangian: L(x, λ, ν) = f₀ + Σλ_i f_i + Σν_j h_j
  Dual: g(λ, ν) = inf_x L(x, λ, ν), 오목
  Dual problem: d* = max g(λ, ν) s.t. λ ≥ 0
  
  약쌍대성 (weak duality): d* ≤ p*    (항상)
  강쌍대성 (strong duality): d* = p*  (Slater 조건 등)
  
  KKT 조건 (볼록에서 필요충분):
  ├── ∇_x L(x*, λ*, ν*) = 0       (stationarity)
  ├── f_i(x*) ≤ 0, h_j(x*) = 0    (primal feasibility)
  ├── λ*_i ≥ 0                     (dual feasibility)
  └── λ*_i · f_i(x*) = 0           (complementary slackness)

알고리즘 계보

  1차 방법 (First-Order)
  ├── Gradient Descent: O(1/k) convex, O((1-μ/L)^k) strongly convex
  ├── Nesterov Accelerated: O(1/k²), O((1-√(μ/L))^k)
  └── Hadn-Strauss Lower Bound: 1차 방법의 최적 수렴률 증명
  
  Proximal 방법
  ├── Proximal Operator: prox_f(v) = argmin_x (f(x) + ½‖x-v‖²)
  ├── Proximal Gradient (ISTA): smooth + nonsmooth
  ├── FISTA: 가속된 Proximal Gradient, O(1/k²)
  ├── ADMM: Augmented Lagrangian + 분리 업데이트
  └── Primal-Dual Splitting: 일반 분해
  
  2차 방법 (Second-Order)
  ├── Newton: 2차 수렴, 비용 O(n³) per iter
  ├── Quasi-Newton (L-BFGS): 근사 헤시안
  └── Interior Point: Barrier 방법, 다항 시간 LP

AI/ML 응용
  ├── Logistic Regression: -log likelihood는 볼록 (Hessian PSD)
  ├── SVM: Hard/Soft margin은 QP (convex), 쌍대 유도 → 커널 트릭
  ├── Lasso: L1 regularization → FISTA/ISTA, sparsity는 기하적
  ├── Matrix Completion: Nuclear norm minimization (SDP)
  ├── 딥러닝: 비볼록, 하지만 over-parameterization으로 "실용적" 최적화
  └── OCO: Online Gradient Descent → Regret O(√T)
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~6개씩)
- 각 문서가 다루는 핵심 정리·증명·수렴률·응용 (3~4줄)
- 전체 문서 개수 확인 (39개 목표)
- Python + NumPy + CVXPY 실험 환경
- LA, Calc & Opt 레포의 어떤 정리를 전제로 사용하는지 명시
- 후속 레포(Stat Learning Theory, Information Geometry, LLM Alignment의 RLHF 등)와의 연결점

**준비됐으면 1단계 구조 설계부터 시작해줘!**
