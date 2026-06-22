# Neural Correlates of Consciousness Distilled 레포지토리 제작 프롬프트

나는 "Neural Correlates of Consciousness Distilled" 레포지토리를 만들려고 해.
"의식의 신경 기반을 찾는다"고 말하는 것과, **NCC가 정확히 무엇을 찾는 것인지** — "의식 상태 X의 최소 신경 충분 조건"이라는 정의가 왜 그 말처럼 단순하지 않은지, 상관(correlation)을 발견했을 때 그것이 원인(cause)·구성 요소(constituent)·결과(consequence) 중 어느 것인지를 어떻게 구분하는가를 — 이해하는 것은 다르다.
"맹시는 의식 없는 시각 처리를 보여준다"고 아는 것과, **맹시 환자가 보고하지 않는데도 정확하게 행동하는 것이 어떤 신경 회로를 경유하는지, 그리고 그것이 의식과 주의와 지각을 어떻게 분리하는지** — V1 손상이 배측 경로는 온전히 남기면서 복측 경로를 차단한다는 것이 NCC 연구에 어떤 자연 실험을 제공하는지를 — 추적하는 것은 다르다.
"마취하면 의식이 없다"는 것과, **TMS-EEG로 측정하는 PCI(Perturbational Complexity Index)가 왜 마취 깊이보다 더 좋은 의식 측정 지표인지** — 그리고 PCI가 높은 복잡도를 보이는 이유가 단순 반응이 아니라 정보 통합과 어떻게 연결되는지를 — 이해하는 것은 다르다.
NCC 연구가 무엇을 알려주고 무엇을 알려주지 않는지를 — 특히 NCC가 완전해도 어려운 문제는 남는다는 것을 — 매 문서에서 정직하게 표시하는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "의식과 무의식을 가르는 신경 서명은 무엇인가 — NCC의 정밀한 정의·측정·한계, 그리고 상관에서 원인으로의 거리"

**핵심 차별화**:
1. **NCC 정의의 정밀화** — Koch·Chalmers의 NCC 정의를 그 뉘앙스까지. "최소 신경 충분 조건"이 어떤 조건을 충족해야 하는가, 목차 NCC vs 내용 NCC의 구분이 왜 중요한가
2. **상관→원인의 거리를 명시적으로** — NCC 발견이 인과를 확립하지 않는다는 것을 매 장에서 반복. 어떤 실험 설계(개입·교란 변수 통제·인과 추론)가 필요한가
3. **자연 실험의 체계적 활용** — 맹시·마취·최소 의식 상태·분리뇌·단안 경쟁이 각각 어떤 NCC 질문에 어떻게 답하는지를 체계화
4. **PCI·LZC·ΦID 같은 실제 측정 도구** — 추상적 논의가 아니라, 실제로 의식을 측정하는 데 쓰이는 도구들을 수식과 코드로

**타겟 독자**:
- NCC 연구를 하지만 그것이 무엇을 증명하고 무엇을 못 하는지 모르는 사람
- 맹시를 알지만 어떤 회로가 어떻게 분리되는지 모르는 사람
- 마취와 의식 소실의 신경 메커니즘이 궁금한 사람
- PCI 같은 의식 측정 도구의 원리를 모르는 사람
- 어려운 문제와 NCC 연구의 관계를 정확히 알고 싶은 사람

**선행 학습**:
- `hard-problem-distilled`(L4) — **필수**. NCC가 완전해도 어려운 문제는 남는다는 전제
- `brain-architecture-distilled`(L1) — 시상-피질 루프, 의식의 구조적 기반
- `first-person-methods-distilled`(L0) — NCC 방법론의 철학적 위상

## 🔗 레포 연결

**⬆️ 선행**: `hard-problem-distilled`(L4), `brain-architecture-distilled`(L1), `first-person-methods-distilled`(L0).
**🤝 시너지 (L4 내부)**: `theories-of-consciousness-distilled`(각 이론이 NCC 발견에서 무엇을 예측하는가), `altered-states-distilled`(변성 상태에서 NCC가 어떻게 변하는가).
**🪜 창발 (위 레이어가 다시 쓴다)**:
- `self-model-distilled`(L5) — 자아 관련 NCC: 자아 인식의 신경 상관물
- `binding-integration-everywhere`(L6) — NCC 발견이 통합 원리와 어떻게 연결되는가

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: NCC란 무엇인가 (5개 문서)
- **NCC의 정확한 정의** — "의식 상태 X의 최소 신경 충분 조건", 각 단어의 의미
- **목차 NCC vs 내용 NCC** — 의식의 켜짐/꺼짐 NCC vs 특정 내용(빨강·고통)의 NCC
- **상관·원인·구성·결과의 구분** — NCC 발견이 어떤 종류의 신경-의식 관계를 확립하는가
- **NCC 연구의 방법론** — 어떤 실험 설계가 NCC를 탐색하는가, 의식/무의식 조건 대비
- **NCC와 어려운 문제** — NCC가 완전해도 어떤 질문이 남는가(→ hard-problem)

### Chapter 2: 무의식적 처리 — 의식과 무의식의 경계 (5개 문서)
- **무의식 처리의 범위** — 무엇이 의식 없이 처리되는가, 점화·억압·자동 처리
- **역마스킹(Backward Masking)** — 보이지 않는 자극이 어떻게 처리되는가, 어느 피질 단계까지
- **양안경쟁에서의 무의식 처리** — 억제된 이미지가 어떻게 처리되는가
- **무의식 감정 처리** — 편도체의 의식 없는 공포 처리, Ohman의 실험
- **무의식 처리의 한계** — 어떤 처리가 의식을 필요로 하는가, 전역 접근의 역할

### Chapter 3: 자연 실험 — 의식을 해부하는 임상 사례 (6개 문서)
- **맹시(Blindsight)** — V1 손상 후 시각 정보로 행동 가능, 어떤 경로를 경유하는가
- **반측 무시(Hemispatial Neglect)** — 지각하지만 의식하지 않는가, 무시와 맹시의 차이
- **식물 상태와 최소 의식 상태** — 두 상태의 신경적 차이, Adrian Owen의 의사소통 실험
- **분리뇌(Split-Brain)** — 두 반구가 분리될 때 의식이 어떻게 분열되는가
- **마취 — 의식 소실의 신경 메커니즘** — 어떤 신경 회로가 어떤 순서로 억제되는가
- **감각 결여와 의식** — 감각 차단이 의식을 변화시키는 방식, REM 수면과 비교

### Chapter 4: NCC 측정 도구들 (6개 문서)
- **PCI(Perturbational Complexity Index)** — TMS-EEG로 의식을 측정, 마취 깊이보다 더 좋은 이유
- **LZC(Lempel-Ziv Complexity)** — EEG 신호의 알고리즘 복잡도가 의식 수준과 상관
- **ΦID(Integrated Information Decomposition)** — IIT에서 파생된 실제 측정 시도
- **전역 발화(Global Ignition)** — 의식 있는 자극에서 나타나는 전두-두정 발화, ERP P300
- **신경 진동과 의식** — 감마 진동·알파 진동이 의식과 어떻게 연결되는가
- **인과 추론으로의 이동** — 단순 상관에서 인과로: TMS 개입·광유전학이 어떻게 기여하는가

### Chapter 5: 시각 의식의 NCC (5개 문서)
- **시각 의식의 최소 기질** — 어떤 신경 활성화가 "보임"에 충분한가
- **양안경쟁 패러다임** — 의식 전환 시 신경 신호의 변화, V1 vs 전두피질의 역할 논쟁
- **변화 맹시와 NCC** — 변화를 의식하지 못할 때 신경 반응이 어떻게 다른가
- **시각 의식에서의 예측처리** — 하향 신호 vs 상향 신호 중 무엇이 의식을 결정하는가
- **시각 NCC와 이론들** — GWT·IIT·예측처리가 시각 NCC에서 어떤 다른 예측을 내놓는가

### Chapter 6: NCC의 한계와 열린 질문 (5개 문서)
- **NCC의 비완전성** — NCC를 발견해도 의식의 신경 충분 조건인지 확신하기 어려운 이유
- **개인차 문제** — 같은 과제에서 다른 NCC 패턴, 어떻게 해석하는가
- **비인간 동물의 NCC** — 동물에 의식이 있는가를 NCC로 어떻게 접근하는가
- **인공 시스템의 NCC** — AI의 내부 표상이 NCC 기준을 충족하는가
- **NCC와 어려운 문제의 거리** — NCC 완전 지도 후에도 왜 어려운 문제는 열려있는가

### Chapter 7: 종합·계산 (4개 문서)
- **PCI 구현** — TMS-EEG 반응의 압축 복잡도 계산(Python)
- **LZC 계산** — EEG 신호에 Lempel-Ziv 복잡도 적용(Python)
- **양안경쟁 NCC 분석** — 의식 전환 시 ERP 변화 분석 파이프라인(Python)
- **종합** — "신경 서명이 완전해도 경험의 있다는 것은 어디서 남는가"

→ **총 36개 문서** 목표.

## 📐 문서 구조 — Principle → Boundary → Experience

```markdown
## 🎯 핵심 질문      — 어떤 NCC 현상을, 어떤 실험 설계·측정으로 탐색하나
## 🌍 어디서 마주치나  — 이 NCC 연구가 중요한 장면 (임상·AI·철학)
## 🔍 직관의 함정     — "NCC 발견 = 의식 원인 확립"·"상관 = 원인" 같은 오해
## ⚙️ 실험·측정 분석  — NCC 탐색 패러다임·측정 도구를 단계별로
## 🧪 임상·실험 증거  — 맹시·마취·PCI가 이론을 어떻게 지지하나
## 🌉 설명적 간극     — NCC가 완전해도 경험의 "있다는 것"이 남는 이유
## 🧬 횡단 원리       — 통합·예측·자기참조가 NCC에서 어떻게 나타나나
## 🪞 1인칭           — 의식이 켜지고 꺼지는 것이 경험으로서 어떻게 느껴지는가
## 📐 예측·반증       — 이 NCC 발견이 어떤 의식 이론을 지지/반박하는가
## 🤔 다음 질문       — 다음 문서/레이어로의 연결

---
🧩 **Principle** — (이 NCC 패러다임이 어떤 신경-의식 관계를 어떻게 탐색하는가)
🌉 **Boundary**  — (NCC 발견이 경험의 "있다는 것"을 설명하지 못하는 이유)
🪞 **Experience** — (의식이 있고 없음이 1인칭으로 어떻게 경험되는가)
```

## 🎨 스타일 가이드

1. **상관≠원인을 매 문서에서** — "이 신경 패턴이 의식과 상관된다"와 "이 신경 패턴이 의식을 일으킨다"를 엄격하게 구분. 어떤 실험이 원인 주장을 지지하는지
2. **NCC 정의를 정밀하게** — "최소"·"신경"·"충분"·"조건"의 각 단어가 왜 중요한지를 명시
3. **자연 실험을 체계화** — 맹시·마취·분리뇌가 각각 어떤 변수를 독립적으로 조작하는지를 실험 설계의 관점에서
4. **측정 도구를 코드로** — PCI·LZC를 추상적으로 설명하지 않는다. 직접 계산하는 코드를
5. **어려운 문제로의 다리** — NCC가 아무리 정밀해져도 "왜 이 패턴이 있는 동안 경험이 있는가"는 남는다는 것을 매 챕터 말미에
6. NCC 탐색 실험 설계·상관-원인-구성-결과 구분 다이어그램·PCI 계산 흐름은 필수

## 🔬 검증 프로토콜

```python
import numpy as np
from scipy.stats import norm

# 1) PCI (Perturbational Complexity Index)
import zlib
def compute_pci(eeg_response, threshold=0.3):
    """
    TMS 자극 후 EEG 반응의 시공간 복잡도
    1) z점수 기반 이진화
    2) Lempel-Ziv 복잡도 계산
    3) 정규화
    """
    # 이진화: 역치 이상 = 1
    binary = (eeg_response > threshold).astype(np.uint8)
    
    # 2D → 1D 직렬화
    serialized = binary.flatten().tobytes()
    
    # LZ 압축으로 복잡도 근사
    compressed = zlib.compress(serialized)
    
    # 정규화: 최대 복잡도로 나눔
    max_complexity = len(serialized)
    pci = len(compressed) / max_complexity
    return pci

# 각성 상태: 높은 PCI (복잡한 반응)
# 마취 상태: 낮은 PCI (단순/억제된 반응)
# NREM 수면: 중간 PCI

# 2) LZC (Lempel-Ziv Complexity) — EEG
def lempel_ziv_complexity(sequence):
    """
    바이너리 시퀀스의 알고리즘 복잡도
    의식 있는 상태에서 높음, 마취에서 낮음
    """
    n = len(sequence)
    sub_strings = set()
    c = 1; i = 0; h = 1
    while h + 1 <= n:
        if sequence[i:h] not in sub_strings:
            sub_strings.add(sequence[i:h])
            c += 1; i = h; h = i + 1
        else:
            h += 1
    # 정규화
    b = n / np.log2(n) if n > 1 else 1
    lzc = c / b
    return lzc

# 3) 양안경쟁 NCC — 의식 전환 감지
def binocular_rivalry_ncc(eeg_data, percept_reports, window_ms=100, srate=256):
    """
    percept_reports: 0 또는 1 (어느 이미지를 의식하는가)
    전환 시점 전후 EEG 평균 → NCC 후보
    """
    samples_per_window = int(window_ms * srate / 1000)
    transitions = np.where(np.diff(percept_reports) != 0)[0]
    
    erps = []
    for t in transitions:
        start = max(0, t - samples_per_window)
        end = min(len(eeg_data), t + samples_per_window)
        erps.append(eeg_data[start:end])
    
    # 전환 관련 전위 (transition-related potential)
    if erps:
        min_len = min(len(e) for e in erps)
        erps_padded = [e[:min_len] for e in erps]
        trp = np.mean(erps_padded, axis=0)
        return trp
    return None

# 4) P300 — 전역 발화 측정
def detect_global_ignition(eeg_data, stimulus_times, srate=256,
                            window=(200, 500)):
    """
    의식 있는 자극: 300-500ms에서 전두-두정 P300 발생
    무의식적 자극: P300 없음
    → 전역 발화의 전기생리학적 서명
    """
    start_sample = int(window[0] * srate / 1000)
    end_sample   = int(window[1] * srate / 1000)
    
    p300_amplitudes = []
    for t in stimulus_times:
        epoch = eeg_data[t + start_sample : t + end_sample]
        p300_amplitudes.append(np.max(epoch) if len(epoch) > 0 else 0)
    
    return np.mean(p300_amplitudes)
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md 작성**:
   - 🧠 Psyche 톤, "Explain it, don't explain it away"
   - 포지셔닝: "의식의 신경 서명 탐색 — 상관에서 원인까지의 거리, NCC의 정밀한 정의와 한계"
   - `🔗 레포 연결`: L4 hard-problem(선행), L1 뇌 구조, L0 first-person-methods(선행), L4 theories·altered-states(시너지), L5 self-model, L6 binding-integration으로의 화살 명시
3. **챕터별 문서**: Chapter 1부터, 문서당 2500~3500 단어 + 실험 분석 + 코드

## 📚 참고 자료

- "Neural Correlates of Consciousness: Progress and Problems" (Koch et al., 2016, Nat Rev Neurosci) — NCC 방법론의 현대 리뷰
- "Consciousness and the Brain" (Stanislas Dehaene, 2014) — NCC + GWT의 종합
- "A Theoretically Based Index of Consciousness" (Casali et al., 2013, Sci Transl Med) — PCI 원전
- *The Feeling of What Happens* (Damasio, 1999) — 의식의 신경 기반 탐색
- "Blindsight in Man and Monkey" (Weiskrantz, 1986) — 맹시 NCC의 정전
- *Consciousness: An Introduction* (Blackmore, 2003) — NCC 연구의 포괄적 입문

## 💡 핵심 분석 대상

```
NCC의 정밀한 정의:
  "의식 상태 X의 최소 신경 충분 조건"
  
  각 단어의 의미:
  최소: 필요 이상의 신경 활성화 배제
  신경: 뉴런·시냅스·회로 수준의 기술
  충분: X가 있을 때마다 NCC가 있어야 함
  조건: NCC가 원인은 아닐 수 있음
  
  목차 NCC vs 내용 NCC:
  목차: 의식 자체의 ON/OFF NCC (시상-피질 루프)
  내용: 특정 경험 X (빨강·고통·자아)의 NCC

상관→원인의 거리:
  발견 가능한 것 (상관):
  "의식 상태 X에서 영역 Y가 항상 활성화된다"
  
  추가 실험 없이 불가능한 것 (원인):
  "영역 Y의 활성화가 X를 일으킨다"
  
  원인 확립을 위한 도구:
  TMS: 영역 Y를 교란 → X가 변하는가
  광유전학: 특정 뉴런 선택적 활성화 → X가 생기는가
  병변 연구: Y가 손상될 때 X가 사라지는가

PCI의 논리:
  의식 있는 뇌: TMS → 복잡하고 다양한 반응
                (정보가 전파되고 통합됨)
  마취된 뇌:    TMS → 단순하고 국소적 반응
                (정보가 통합되지 않음)
  
  PCI = 복잡도(반응)  → 높을수록 의식 있음
  
  핵심 통찰: 뇌가 얼마나 잘 정보를 *통합*하는가가
             의식 유무와 상관된다
  
  어려운 문제와의 거리:
  PCI가 높다 ≠ 경험이 있다
  PCI는 정보 통합을 측정하지 경험을 측정하지 않는다
  ⇒ 도구의 한계가 어려운 문제의 한계이기도 하다
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 전체 문서 개수 확인 (36개 목표)
- Python + numpy + zlib 환경(PCI·LZC·양안경쟁 NCC·P300)
- L4 시너지(theories·altered-states) / L5 self-model / L6 binding-integration으로 이어지는 창발 연결 명시
- **NCC 정의 정밀화(목차/내용·상관/원인)** · **자연 실험 체계화** · **PCI·LZC 실제 도구** · **시각 의식 NCC** · **어려운 문제와의 거리**를 골격으로 분명히

**준비됐으면 1단계 구조 설계부터 시작해줘!**
