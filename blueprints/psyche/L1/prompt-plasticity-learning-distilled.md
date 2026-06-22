# Plasticity & Learning Distilled 레포지토리 제작 프롬프트

나는 "Plasticity & Learning Distilled" 레포지토리를 만들려고 해.
"기억은 시냅스에 저장된다"고 말하는 것과, **헵 가소성이 정확히 어떤 분자 메커니즘에서 시냅스 효율을 변화시키는지** — NMDA 수용체가 어떻게 코인시던스 탐지기로 작동하고, LTP가 AMPA 수용체 삽입을 통해 어떻게 유지되는지를 — 직접 추적하는 것은 다르다.
STDP(스파이크 타이밍 의존 가소성)를 "인과율의 신경 구현"으로 듣는 것과, **사전 스파이크와 사후 스파이크의 시간 간격이 왜 -20ms에서 +20ms 창 안에서만 시냅스 강화를 일으키는지** — 그 비대칭 창이 어떤 계산적 이유를 가지는지를 — 이해하는 것은 다르다.
"엔그램 세포를 발견했다"는 뉴스를 보는 것과, **기억의 물리적 흔적(engram)이 어떻게 정의되고, 광유전학 실험이 어떻게 특정 세포 집단을 기억의 표현으로 확인하며, 엔그램과 인출이 왜 분리될 수 있는지** — 망각이 저장 실패가 아닌 인출 실패일 수 있다는 것을 — 아는 것은 다르다.
생물학적 학습이 역전파와 어떻게 다른지를 — 그리고 그 차이가 왜 단순한 구현 차이가 아니라 **학습의 본성에 관한 근본적 질문**인지를 — 끝까지 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "경험은 어떻게 물질에 새겨지나 — 헵 가소성·STDP·엔그램에서 생물학적 학습의 원리까지"

**핵심 차별화**:
1. **LTP/LTD를 분자 메커니즘에서** — "시냅스가 강해진다"가 아니라, NMDA 수용체·칼슘 유입·CaMKII 인산화·AMPA 삽입의 단계를 직접 추적, 유도 프로토콜과 분자 경로를 수치로
2. **STDP의 계산적 의미** — 시간 창의 비대칭이 인과율 탐지와 어떻게 연결되는지, 시냅스 학습 규칙으로서 STDP가 어떤 계산을 구현하는지
3. **엔그램 연구를 정밀하게** — Liu·Tonegawa 팀의 광유전학 실험이 무엇을 증명하고 무엇을 증명 못 하는지, "기억 세포"와 "기억 표상"의 차이, 인출 실패 이론
4. **생물학적 학습 vs 역전파** — 신경망의 역전파가 생물학적으로 비현실적인 이유(전역 오차 신호·가중치 대칭·잠금 전달), 생물학적 대안(예측 부호화·대조 헵 학습)의 현황

**타겟 독자**:
- LTP를 알지만 그 분자 메커니즘을 모르는 사람
- STDP를 "헵 학습의 시간 버전"으로만 아는 사람
- 엔그램 연구를 뉴스로 접했지만 실험 설계와 함의를 모르는 사람
- "역전파는 뇌에서 일어나지 않는다"는 말이 왜 중요한지 모르는 사람
- 경험이 물질에 새겨지는 방식과 그것이 경험과 어떻게 연결되는지 보고 싶은 사람

**선행 학습**:
- `neurons-neural-codes-distilled`(L1) — **필수**. NMDA 수용체, 시냅스 작동 원리
- `brain-architecture-distilled`(L1) — 해마·편도체·소뇌에서의 학습 회로

## 🔗 레포 연결

**⬆️ 선행**: `neurons-neural-codes-distilled`·`brain-architecture-distilled`(L1).
**🤝 시너지 (L1 내부)**: `neuromodulation-arousal-distilled`(도파민·아세틸콜린이 가소성을 어떻게 조절하는가).
**🪜 창발 (위 레이어가 다시 쓴다)**:
- `memory-distilled`(L2) — 시냅스 가소성에서 기억이 어떻게 창발하는가, 응고화·재응고화
- `learning-decision-distilled`(L2) — 강화학습의 신경 기반, 보상 예측 오차와 가소성
- `brains-vs-networks-distilled`(L3) — 생물학적 학습 vs 역전파의 구체적 비교
- `plasticity-learning-distilled`(→ IQ AI Lab) — 이 레포가 AI Lab의 학습 이론과 교차
- `prediction-everywhere`(L6) — STDP가 예측 원리의 구현 층위 본체

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 가소성 — 물질이 경험을 기록하다 (4개 문서)
- **가소성이란** — 시냅스 강도가 변할 수 있다는 것의 의미, 헵의 원래 제안
- **가소성의 종류** — 단기·장기, 시냅스 전·후, 구조적·기능적 가소성의 분류
- **가소성의 계산적 이유** — 왜 학습이 시냅스 가중치 변화로 구현되는가, 신경 학습 규칙의 일반 원리
- **가소성과 기억의 관계** — 시냅스 가소성이 기억의 충분 조건인가, 필요 조건인가

### Chapter 2: LTP와 LTD — 분자 수준의 학습 (6개 문서)
- **LTP의 발견과 프로토콜** — Bliss·Lømo의 원래 실험, 고빈도 자극이 어떻게 시냅스를 강화하는가
- **NMDA 수용체 — 코인시던스 탐지기** — 전압·리간드 이중 의존성, Mg²⁺ 블록의 제거 조건, 사전-사후 동시성 요구
- **LTP의 분자 경로** — Ca²⁺ 유입 → CaMKII 활성화 → AMPA 수용체 인산화·삽입 → 시냅스 효율 증가
- **LTP의 지속성** — 조기 LTP(E-LTP, 수 시간)와 후기 LTP(L-LTP, 단백질 합성 의존), 기억 응고화와의 연결
- **LTD — 시냅스 약화** — 낮은 빈도 자극에서의 LTD, PP1·PP2B를 통한 AMPA 내재화
- **시냅스 항상성** — 스케일링(전역 강화/약화)이 과활성·불활성을 어떻게 방지하는가

### Chapter 3: STDP — 타이밍이 인과를 배운다 (5개 문서)
- **STDP의 발견** — Markram·Bi & Poo의 실험, 스파이크 타이밍 창의 측정
- **STDP 창의 구조** — 사전 선행(+Δt): LTP, 사후 선행(-Δt): LTD, 비대칭 지수 감소
- **STDP의 계산적 의미** — 인과율 탐지 규칙, 선행하는 것이 강화된다, 헵 학습의 시간 정밀화
- **STDP와 시간 부호** — STDP가 시간 부호를 어떻게 지지하는가, 순서 학습에서의 역할
- **STDP의 한계와 변형** — 모든 시냅스에서 STDP가 작동하는가, 삼중 STDP·전압 의존 변형

### Chapter 4: 엔그램 — 기억의 물리적 흔적 (5개 문서)
- **엔그램의 개념** — Semon의 원래 제안, 기억 흔적이란 무엇인가, 현대적 재정의
- **광유전학으로 엔그램 찾기** — Liu·Tonegawa 팀의 실험 설계: 두려움 조건화 중 활성화 세포 표지 → 빛으로 재활성화 → 기억 재현
- **엔그램과 인출의 분리** — 엔그램 세포는 있지만 인출이 안 될 수 있다, 망각 = 저장 실패 vs 인출 실패
- **엔그램의 이동·재응고화** — 단기 기억(해마)에서 장기 기억(피질)으로의 이동, 재응고화 창과 기억 수정
- **엔그램 연구의 한계** — 어떤 세포가 엔그램인가, 엔그램이 기억의 충분 조건인가, 분산 표상과의 관계

### Chapter 5: 시스템 공고화 — 기억이 안정되다 (4개 문서)
- **해마 의존 기억** — H.M. 환자, 해마가 새 기억 형성에 필수인 이유
- **기억 응고화의 두 단계** — 세포 수준(단백질 합성, 수 시간)·시스템 수준(피질 전이, 수 주-년)
- **수면과 기억** — 수면 중 해마-피질 재활성화, 방추 진동·예파 복합체의 기억 이전 역할
- **재응고화 창** — 인출 후 기억이 다시 불안정해지는 현상, 외상 기억 치료의 가능성과 한계

### Chapter 6: 생물학적 학습 vs 역전파 (5개 문서)
- **역전파의 생물학적 문제점** — 전역 오차 신호·가중치 대칭·잠금 전달·비국소 신호의 생물학적 비현실성
- **예측 부호화 기반 학습** — 예측 오차가 국소 학습 신호가 되는 방식, 역전파의 생물학적 근사(→ L0 information)
- **대조 헵 학습 (CHL)** — 자유 위상과 고정 위상의 대비가 가중치를 어떻게 갱신하는가
- **강화 학습의 생물학** — 도파민이 전역 보상 신호로 작동, 3인자 시냅스 규칙(→ neuromodulation)
- **현재의 최전선** — 생물학적으로 타당한 심층 학습의 현황, AI Lab과의 교차(→ brains-vs-networks L3)

### Chapter 7: 종합·계산 (5개 문서)
- **LTP 분자 경로 시뮬레이션** — Ca²⁺ 동역학·CaMKII 활성화·AMPA 삽입 수치 모델(Python)
- **STDP 학습 규칙 구현** — 스파이크 쌍에서 시냅스 가중치 변화 계산(Python)
- **엔그램 네트워크 모델** — Hopfield 네트워크에서 기억 저장·인출·망각 시뮬레이션(Python)
- **생물 vs 역전파 비교** — 같은 과제에서 헵 학습 vs 역전파 성능·수렴 비교(Python)
- **종합** — "경험이 물질에 새겨지는 방식과, 그 물질이 경험이 되는 방식의 간극"

→ **총 34개 문서** 목표.

## 📐 문서 구조 — Principle → Boundary → Experience

Psyche 표준 10섹션, 마지막 세 단 종결:

```markdown
## 🎯 핵심 질문      — 어떤 가소성 현상을, 어떤 분자·계산 원리에서 설명하나
## 🌍 어디서 마주치나  — 이 가소성이 작동하는 장면 (학습·기억·외상·치료)
## 🔍 직관의 함정     — "기억은 저장된다"·"망각은 삭제다" 같은 오해
## ⚙️ 메커니즘 유도   — 분자 경로·이온 동역학·학습 규칙을 단계별로 추적
## 🧪 실험·광유전학 증거 — LTP 유도, 엔그램 재활성화, 기억 이전 실험
## 🌉 설명적 간극     — 가소성이 학습을 설명해도 *기억된다는 경험*은 어디서 남는가
## 🧬 횡단 원리       — 예측·자기참조가 가소성에서 어떻게 나타나나
## 🪞 1인칭           — 기억이 형성되고 인출될 때 경험으로서 어떻게 느껴지는가
## 📐 예측·반증       — 이 가소성 이론이 내놓는 반증 가능한 예측
## 🤔 다음 질문       — 다음 문서/레이어로의 연결

---
🧩 **Principle** — (이 가소성 메커니즘이 어떻게 학습을 구현하는가)
🌉 **Boundary**  — (가소성 설명이 기억 경험을 설명하지 못하는 지점)
🪞 **Experience** — (학습·기억이 1인칭으로 어떻게 경험되는가)
```

## 🎨 스타일 가이드

1. **분자를 유도로** — "NMDA가 열리면 LTP"가 아니라, Ca²⁺ 유입이 어떻게 CaMKII를 활성화하고 AMPA를 삽입하는지를 단계별로
2. **STDP를 계산 규칙으로** — 시간 창의 비대칭이 "인과율"을 어떻게 학습하는지, 그리고 이것이 강화학습·예측 처리와 어떻게 연결되는지
3. **엔그램을 과장하지 않는다** — 광유전학 실험이 무엇을 증명하고 무엇을 증명 못 하는지를 정밀하게. "기억 세포 발견"은 단순화
4. **역전파 논쟁을 공정하게** — "역전파는 뇌에서 작동하지 않는다"를 확정 주장으로 하지 않는다. 무엇이 생물학적으로 비현실적이고, 어떤 근사가 가능한지를
5. **경험의 잔여를 매 챕터에** — 가소성이 학습 메커니즘을 설명해도, 기억이 *느껴지는 것*은 남는다
6. LTP 분자 경로·STDP 창·엔그램 네트워크·역전파 vs 헵 학습 비교는 다이어그램으로

## 🔬 검증 프로토콜

> LTP 분자 시뮬레이션 + STDP 구현 + Hopfield 기억 모델 + 학습 알고리즘 비교.

```python
# 환경: Python 3 + numpy + scipy + matplotlib

# 1) LTP 분자 경로 — Ca²⁺ 동역학
import numpy as np
def ca_dynamics(pre_spikes, post_spikes, dt=0.1, tau_ca=20.0):
    """
    NMDA 수용체: pre·post 동시 활성 시 Ca²⁺ 유입
    tau_ca: Ca²⁺ 감쇠 시간상수 (ms)
    """
    T = len(pre_spikes)
    Ca = np.zeros(T)
    for t in range(1, T):
        if pre_spikes[t] and post_spikes[t]:  # 코인시던스
            Ca[t] = Ca[t-1] + 1.0  # Ca²⁺ 유입
        Ca[t] -= Ca[t-1] * dt / tau_ca  # 감쇠
    # Ca > threshold_high → LTP (CaMKII), Ca < threshold_low → LTD
    return Ca

# 2) STDP 학습 규칙
def stdp_update(delta_t, A_plus=0.01, A_minus=0.01,
                tau_plus=20.0, tau_minus=20.0):
    """
    delta_t = t_post - t_pre (ms)
    delta_t > 0: pre 선행 → LTP
    delta_t < 0: post 선행 → LTD
    """
    if delta_t > 0:
        dw = A_plus * np.exp(-delta_t / tau_plus)
    else:
        dw = -A_minus * np.exp(delta_t / tau_minus)
    return dw

# STDP 창 시각화
delta_ts = np.linspace(-100, 100, 1000)
dws = [stdp_update(dt) for dt in delta_ts]
# → 비대칭 창: 양수(LTP) + 음수(LTD)

# 3) Hopfield 네트워크 — 기억 저장·인출
class HopfieldNetwork:
    def __init__(self, n_neurons):
        self.n = n_neurons
        self.W = np.zeros((n_neurons, n_neurons))
    
    def store(self, patterns):
        """헵 학습: W_ij += (1/N) * Σ_μ ξ_i^μ * ξ_j^μ"""
        for p in patterns:
            self.W += np.outer(p, p) / self.n
        np.fill_diagonal(self.W, 0)
    
    def recall(self, query, n_iter=10):
        """비동기 갱신으로 가장 가까운 저장 패턴 복원"""
        state = query.copy()
        for _ in range(n_iter):
            i = np.random.randint(self.n)
            state[i] = np.sign(self.W[i] @ state)
        return state
    
    def capacity(self):
        """이론적 용량: ~0.138 * N 패턴"""
        return int(0.138 * self.n)

# 4) 역전파 vs 헵 학습 비교
def hebbian_update(x, y, lr=0.01):
    """국소 헵 학습: Δw = lr * x * y"""
    return lr * np.outer(x, y)

# 같은 XOR 과제에서 두 방식의 학습 곡선 비교
# → 헵 학습의 한계(비선형 문제에서 실패), 역전파의 성능
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md 작성**:
   - 🧠 Psyche 톤(brain·핑크 `#ec4899`), "Explain it, don't explain it away"
   - 포지셔닝: "기억은 시냅스에 저장된다 vs 경험이 물질에 새겨지는 분자·계산 원리"
   - **The Knowledge Pentad** 각주, IQ AI Lab과의 교차 표시
   - `🔗 레포 연결`: L1 뉴런·부호·구조(선행), L1 neuromodulation(시너지), L2 기억·학습, L3 뇌 vs AI(→ IQ AI Lab), L6 prediction-everywhere로의 화살 명시
   - **Document Format — Principle → Boundary → Experience** 안내
3. **챕터별 문서**: Chapter 1부터, 문서당 2500~3500 단어 + 분자 메커니즘 + 코드/다이어그램

## 📚 참고 자료

- *Synaptic Self* (Joseph LeDoux, 2002) — 시냅스 가소성과 자아의 연결, 이 레포의 철학적 기반
- "LTP and the Biological Basis of Learning" (Bliss & Collingridge, 1993, Nature) — LTP의 정전
- "Spike Timing–Dependent Plasticity" (Dan & Poo, 2004, Physiol Rev) — STDP의 포괄적 리뷰
- "Creating a False Memory in the Hippocampus" (Ramirez et al., 2013, Science) — 광유전학 엔그램 원전
- *Memory: From Mind to Molecules* (Squire & Kandel) — 기억 연구의 표준 교과서
- "Towards Biologically Plausible Deep Learning" (Bengio et al., 2015) — 역전파 vs 생물학

## 💡 핵심 분석 대상

```
LTP의 분자 연쇄:
  사전 스파이크 → 글루타메이트 방출
       +
  사후 탈분극 → Mg²⁺ 제거 (NMDA 개방 조건)
       ↓
  Ca²⁺ 유입 → CaMKII 활성화 (코인시던스 탐지)
       ↓
  CaMKII → AMPA 수용체 인산화·삽입
       ↓
  시냅스 효율 증가 (EPSP 크기 ↑)
  → 장기 유지: 단백질 합성 → L-LTP (→ 기억 응고화)

STDP 창 — 인과율의 학습:
  Δt = t_post - t_pre
  
  Δt > 0 (pre 선행):  "A가 B를 일으켰다" → LTP (+dw)
  Δt < 0 (post 선행): "B는 A와 무관했다" → LTD (-dw)
  
  ⇒ 헵의 원칙("함께 발화하면 함께 연결된다")의
    시간 정밀 버전: *순서*와 *인과*를 배운다

엔그램의 정밀한 정의:
  엔그램 세포 = 기억 형성 중 활성화 + 기억 표현 중 재활성화가 필요
  
  증명된 것:
  ├─ 특정 세포 집단이 공포 기억 형성 중 활성화
  ├─ 그 세포의 광유전학 재활성화가 행동 반응 재현
  └─ 기억이 "없어진 것처럼" 보여도 엔그램은 존재
  
  증명되지 않은 것:
  └─ 이 세포들이 기억의 *충분* 조건인가
  └─ 어떻게 분산 표상과 양립하는가

역전파의 생물학적 문제:
  역전파가 필요로 하는 것:
  ├─ 전역 오차 신호 (어느 층에나 동시에 전달)
  ├─ 가중치 대칭 (전방·후방 가중치 동일)
  ├─ 잠금 전달 (전방 계산 끝날 때까지 후방 대기)
  └─ 계속적 미분 가능 활성함수
  
  뇌의 현실:
  ├─ 오차 신호는 국소적
  ├─ 전방·후방 연결은 비대칭
  └─ 스파이크는 불연속
  ⇒ 역전파는 뇌의 학습이 아니지만, 근사가 가능할 수 있다
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 가소성·학습 메커니즘 (3~4줄)
- 전체 문서 개수 확인 (33개 목표)
- Python + numpy + scipy 환경(LTP 분자·STDP·Hopfield·역전파 vs 헵 비교)
- L1 시너지(neuromodulation) / L2 기억·학습·의사결정 / L3 뇌 vs AI(IQ AI Lab) / L6 prediction-everywhere로 이어지는 *창발 연결* 명시
- **LTP/LTD(분자 경로)** · **STDP(인과율 학습)** · **엔그램(기억의 물리적 흔적, 정밀 정의)** · **시스템 응고화·수면** · **역전파 vs 생물학적 학습(공정한 대결)** · **가소성에서 경험으로의 간극**을 골격으로 분명히

**준비됐으면 1단계 구조 설계부터 시작해줘!**
