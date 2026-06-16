# Emergent Quantum Matter Distilled 레포지토리 제작 프롬프트

나는 "Emergent Quantum Matter Distilled" 레포지토리를 만들려고 해.
"전자가 많이 모이면 금속"이라고 말하는 것과, **다수의 전자가 *새로운 입자*(준입자)를 만들어낸다**는 것 — "More is Different"가 빈말이 아니라 물리적 사실임을 — 이해하는 것은 다르다.
초전도를 "저항이 0"으로 외우는 것과, 쿠퍼 쌍·거시적 양자 결맞음이 *어떻게 창발*하는지(BCS) 아는 것은 다르다.
양자홀 효과의 전도도가 정확히 양자화된다는 것과, 그것이 **위상(topology)** — 대칭으로는 분류 안 되는 새 질서 — 의 귀결이고 분수 전하·애니온까지 낳는다는 것을 아는 것은 다르다.
응집물질이 **창발의 살아있는 증거**라는 것을 — 게이지장조차 창발할 수 있고, 얽힘이 물질의 상을 규정한다는 것을 — 끝까지 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "'More is Different'의 살아있는 증거 — 준입자가 창발하고, 위상이 물질을 분류하고, 게이지장조차 창발한다"

**핵심 차별화**:
1. **준입자 = 창발하는 입자** — 다수의 전자가 새 "입자"(포논·마그논·쿠퍼 쌍·애니온)를 만든다, 무엇이 근본인가(→ L6 emergence)
2. **초전도·초유체의 창발** — 거시적 양자 결맞음, BCS·BEC, 질서변수(→ L2 statistical-mechanics·phase-transitions)
3. **위상이 분류하는 물질** — 대칭만으론 부족, 천(Chern) 수·위상 불변량, 양자홀의 정밀 양자화, 분수 전하·애니온(→ quantum-information)
4. **창발 게이지장의 역설** — 스핀 액체에서 광자·게이지장이 창발, "근본"이 창발일 수도(→ standard-model과 대비), 얽힘이 상을 규정(→ L5)

**타겟 독자**:
- "전자 많으면 금속"으로만 알고 *준입자가 창발*함을 모르는 사람
- 초전도를 "저항 0"으로만 알고 쿠퍼 쌍·거시 양자성을 모르는 사람
- 양자홀 양자화가 *위상*의 귀결임을, 분수 전하·애니온의 존재를 모르는 사람
- 게이지장이 *창발*할 수 있음(근본 vs 창발의 역설)을 모르는 사람
- 응집물질이 어떻게 양자정보·홀로그래피로 *이어지는지* 보고 싶은 사람

**선행 학습**:
- `statistical-mechanics-distilled`(L2) — **필수**. 페르미 액체·보스-아인슈타인 응축
- `phase-transitions-distilled`(L2) — **필수**. 질서변수·자발 대칭 깨짐·양자 상전이
- `quantum-mechanics-distilled`(L3) — **필수**. 다체 양자·밴드 / `symmetry-conservation-distilled`(L0) — 골드스톤·대칭

## 🔗 레포 연결

**⬆️ 선행**: `statistical-mechanics-distilled`·`phase-transitions-distilled`(L2), `quantum-mechanics-distilled`(L3).
**🤝 시너지 (L3 내부)**: `quantum-information-distilled`(애니온·위상 양자컴퓨팅·얽힘 엔트로피로 본 상), `standard-model-distilled`(창발 게이지장 ↔ 근본 게이지장의 역설적 대비), `entanglement-measurement-distilled`(위상 질서 = 긴 거리 얽힘).
**🪜 창발 (위 레이어가 다시 쓴다)**:
- `holographic-principle-distilled`(L5) — 얽힘과 기하, 강상관계의 홀로그래피적 기술
- `emergent-spacetime-multiverse-distilled`(L5) — 얽힘에서 창발하는 시공간(같은 아이디어의 극단)
- `emergence-distilled`(L6) — **이 레포가 "창발의 정수"의 살아있는 증거로 회수되는 본체**

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: More is Different (5개 문서)
- **환원주의의 한계** — 앤더슨의 "More is Different", 다수에서 *새 법칙*이 창발(→ L6 emergence)
- **준입자(quasiparticle)** — 다수의 전자가 만드는 새 "입자", 창발적 실체의 정의
- **창발 vs 근본** — 무엇이 진짜 기본인가, 유효 자유도, 같은 물리의 다른 기술(→ L6)
- **대칭 깨짐과 질서** — 응집물질의 질서변수, 강자성·결정(→ L0 symmetry, → L2 phase-transitions)
- **위상(topology)의 등장** — 대칭만으론 부족, 위상이 분류하는 새 물질 상(예고)

### Chapter 2: 다체 양자와 준입자 (5개 문서)
- **다체 문제** — $10^{23}$개 상호작용 전자, 왜 풀 수 없고 왜 풀 필요가 없나
- **페르미 액체** — 상호작용에도 살아남는 준입자, 란다우의 통찰(→ L2 statistical-mechanics)
- **포논** — 격자 진동의 양자, 가장 친숙한 창발 입자(→ L1 oscillations)
- **마그논·플라스몬** — 다양한 집단 들뜸, 깨진 대칭의 골드스톤(→ L0 symmetry)
- **평균장과 그 너머** — BCS·강상관, 평균장이 언제 무너지나(→ L0 approximation)

### Chapter 3: 초전도와 초유체 (5개 문서)
- **초유체** — 점성 없는 흐름, 보스-아인슈타인 응축의 거시 양자성(→ L2 statistical-mechanics)
- **초전도 현상** — 저항 0·마이스너 효과, 거시적 양자 상태
- **BCS 이론** — 쿠퍼 쌍, 포논 매개 인력, 에너지 갭(→ phonon)
- **거시적 양자 결맞음** — 질서변수·긴즈부르크-란다우, 위상 강성(→ L2 phase-transitions)
- **조셉슨 효과** — 양자 위상의 *관측 가능성*, SQUID·양자 표준(→ L3 quantum-mechanics)

### Chapter 4: 위상 물질 (6개 문서)
- **양자홀 효과** — 정수 양자화, *왜 그렇게 정밀한가*(불순물에 무관)
- **천(Chern) 수와 위상 불변량** — 밴드의 위상, 베리 위상, 위상이 정수인 이유
- **분수 양자홀** — 분수 전하·애니온, 창발 입자의 극단(→ quantum-information)
- **위상 절연체** — 안은 절연·겉은 도체, 보호된 표면 상태, 벌크-경계 대응
- **위상 보호** — 왜 무질서·교란에 강한가, 위상의 견고함(→ quantum-information 오류정정)
- **마요라나와 비아벨 애니온** — 비아벨 통계, 위상 양자컴퓨팅의 자원(→ quantum-information)

### Chapter 5: 창발하는 게이지장 (4개 문서)
- **창발 게이지장** — 스핀 액체에서 광자·게이지장이 *창발*, 게이지가 근본이 아닐 수도(→ standard-model 대비)
- **분수화(fractionalization)** — 전자가 쪼개진다, 스피논·홀론, 창발 입자
- **위상 질서** — 대칭 깨짐 *없는* 질서, 긴 거리 얽힘(→ entanglement-measurement)
- **얽힘과 물질의 상** — 얽힘 엔트로피로 본 상(phase), 면적 법칙(→ quantum-information, → L5 holography)

### Chapter 6: 양자 상전이와 강상관 (5개 문서)
- **양자 상전이** — $T=0$ 전이, 양자 요동이 일으키는 전이(→ L2 phase-transitions)
- **양자 임계성** — 양자 임계점, 이상 금속(non-Fermi liquid)
- **모트 절연체** — 상호작용이 만드는 절연체, 허바드 모형
- **고온 초전도** — 미해결의 강상관 문제, 구리·철 기반
- **강상관의 최전선** — 풀리지 않은 다체 문제들(정직한 미해결의 영역)

### Chapter 7: 종합·계산 (5개 문서)
- **밴드 구조 계산** — 긴밀결합 모형, 밴드와 띠틈(Python)
- **천 수 계산** — 위상 불변량의 수치 계산, 베리 곡률(Python)
- **BCS 갭 방정식** — 자기일관 풀이, 임계온도(Python)
- **허바드/이징 다체** — 작은 계 정확 대각화, 상관(Python)
- **종합** — "More is Different의 살아있는 증거"와 창발의 정수(→ L6, → L5)

→ **총 35개 문서** 목표.

## 📐 문서 구조 — Principle → Boundary → Emergence

Physis 표준 10섹션, 마지막 세 단 종결:

```markdown
## 🎯 핵심 질문      — 어떤 창발 현상을, 어떤 다체 원리에서 유도하나
## 🌍 어디서 나타나나  — 이 물질 상이 작동하는 장면
## 🔍 직관의 함정     — "준입자는 비유"·"위상=모양" 같은 오해
## ⚙️ 제1원리 유도    — 다체·BCS·위상 불변량에서 결과를 끌어낸다
## 🧪 실험·관측 증거  — 양자홀 양자화·초전도·분수 전하의 실측
## 🔬 경계           — 이 기술이 깨지는 곳 (강상관·고온 초전도 미해결)
## 🧬 횡단 원리       — 대칭(SSB)·정보(얽힘)·창발이 어떻게 반복되나
## 🪜 창발           — 위 레이어(홀로그래피·창발 시공간)에서 무엇이 나타나나
## 📐 예측·반증       — 양자화 전도도·갭·임계온도 등 검증 가능한 예측
## 🤔 다음 질문       — 다음 문서/레이어로의 연결

---
🧩 **Principle** — (다수에서 무엇이 어떻게 창발하나)
🔬 **Boundary**  — (이 기술이 깨지는 조건: 강상관·고온 초전도)
🪜 **Emergence** — (위 레이어에서 같은 구조가 낳는 것 + 검증 실험)
```

이 레포에서 **⚙️ 제1원리 유도**는 *다체 해밀토니안·BCS 갭 방정식·위상 불변량*에서 끌어내며, **🔬 경계**는 "고온 초전도는 미해결", "강상관에서 페르미 액체 붕괴", "위상 분류의 한계" 같은 한계를 정직하게 표시한다.

## 🎨 스타일 가이드

1. **창발을 유도로** — 준입자가 다수에서 *나오는* 과정을 보인다. "More is Different"를 물리로(→ L6)
2. **위상 = 견고한 질서** — 양자홀 양자화가 *위상*의 귀결이고 불순물에 무관함을(대칭 너머의 질서)
3. **근본 vs 창발의 역설** — 게이지장조차 창발할 수 있음을, 표준모형의 "근본" 게이지와 대비(→ standard-model)
4. **얽힘이 상을 규정** — 위상 질서 = 긴 거리 얽힘, 얽힘 엔트로피로 상을 본다(→ quantum-information, → L5)
5. **수치로 부순다** — 밴드·천 수·BCS 갭·다체 대각화를 직접 계산
6. 밴드 구조·천 수·BCS 갭·위상 가장자리 상태는 다이어그램으로

## 🔬 검증 프로토콜

> 긴밀결합 밴드 + 천 수(위상 불변량) + BCS 갭 + 다체 정확 대각화.

```python
# 환경: Python 3 + numpy + scipy + matplotlib

# 1) 긴밀결합 밴드 구조
import numpy as np
def tight_binding_1d(t=1.0, N=200):
    k=np.linspace(-np.pi,np.pi,N)
    E=-2*t*np.cos(k)                  # 1D 밴드
    return k,E
# 2D/SSH 모형으로 확장 → 띠틈·가장자리 상태

# 2) 천 수 (위상 불변량)
#    2밴드 모형 d(k)의 단위벡터가 브릴루앙 영역을 감는 횟수
#    C = (1/4π)∫ d̂·(∂_kx d̂ × ∂_ky d̂) d²k → 정수
#    → 위상이 정수로 양자화됨을 수치로 (양자홀 전도도 = C·e²/h)

# 3) BCS 갭 방정식
#    1 = g Σ_k 1/(2E_k) tanh(E_k/2kT), E_k=√(ξ_k²+Δ²) 자기일관 풀이
def bcs_gap(g, T, ...):
    # Δ를 반복으로 수렴 → 임계온도 Tc, 갭 Δ(T)
    ...
# → 갭이 Tc에서 0이 되는 거동, BCS 비 Δ(0)/kTc≈1.76 확인

# 4) 허바드/이징 정확 대각화
#    작은 격자(예: 4 사이트) 해밀토니안을 전부 대각화 → 바닥상태·상관
#    → 모트 절연체·강상관의 맛보기

# 5) 베리 위상 (1D)
#    SSH 모형에서 짝-홀짝 위상에 따른 윈딩(Zak phase) 계산
#    → 위상 절연체 vs 보통 절연체의 구분

# 6) 양자홀 가장자리 상태
#    띠 기하에서 위상 밴드의 가장자리 모드 → 벌크 띠틈 속 전도 채널
#    → 벌크-경계 대응 시연
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md 작성**:
   - 🌌 Physis 톤(atom·인디고 `#6366f1`), "Derive, don't accept", `From the fewest principles, the widest world.`
   - 포지셔닝: "전자 많으면 금속 vs 준입자가 창발·위상이 분류 — More is Different의 살아있는 증거"
   - **The Knowledge Tetrad** 각주
   - `🔗 레포 연결`: L2 `statistical-mechanics`·`phase-transitions`·L3 `quantum-mechanics`(선행), L3 quantum-information(애니온·위상 QC)·standard-model(창발 게이지 대비)·entanglement(위상 질서), L5 홀로그래피·창발 시공간, L6 `emergence`(본체)로의 화살 명시
   - **Document Format — Principle → Boundary → Emergence** 안내
3. **챕터별 문서**: Chapter 1부터, 문서당 2500~3500 단어 + 유도 + 코드/그림

## 📚 참고 자료

- *Basic Notions of Condensed Matter Physics* (P.W. Anderson) — "More is Different"의 원천, 이 레포의 영혼
- *Condensed Matter Field Theory* (Altland & Simons) — 다체·창발의 표준
- *Topological Insulators and Topological Superconductors* (Bernevig & Hughes) — 위상 물질의 표준
- *Principles of Condensed Matter Physics* (Chaikin & Lubensky) — 질서·창발
- *Quantum Field Theory of Many-Body Systems* (Xiao-Gang Wen) — 위상 질서·창발 게이지장
- "More is Different" (P.W. Anderson, Science 1972) — 창발의 선언문(→ L6)

## 💡 핵심 분석 대상

```
More is Different (창발의 살아있는 증거):
  근본 법칙 (전자 + 쿨롱)  →  *다수*가 모이면
       ▼
  새 자유도·새 입자(준입자)·새 법칙이 *창발*
  ├─ 포논 (격자 진동의 양자, → L1 oscillations)
  ├─ 쿠퍼 쌍 (전자 둘이 묶인 보손, → 초전도)
  ├─ 마그논·플라스몬 (집단 들뜸, 골드스톤)
  └─ 애니온 (분수 전하·분수 통계!)
  ⇒ "전체는 부분의 합 이상" — 물리적 사실 (→ L6 emergence)

초전도의 창발 (BCS):
  포논 매개 인력 → 쿠퍼 쌍 (페르미면 근처)
       │  거시적 수가 같은 양자상태로 응축
       ▼
  거시적 양자 결맞음 → 저항 0, 마이스너, 에너지 갭 Δ
  질서변수 ψ=|ψ|e^{iφ} (긴즈부르크-란다우, → L2 phase-transitions)

위상이 분류하는 물질 (대칭 너머):
  양자홀: σ_xy = C·e²/h    (C = 천 수, 정수!)
       │  C는 *위상* 불변량 → 불순물·모양에 무관 → 극도로 정밀
       ▼
  분수 양자홀: 분수 전하 e/3, 애니온 (비아벨 통계)
  위상 절연체: 벌크 띠틈 + 보호된 가장자리 상태
  ⇒ 대칭 깨짐으로 분류 안 되는 새 질서 (위상 질서 = 긴 거리 얽힘)

창발 게이지장의 역설:
  스핀 액체 → 광자·게이지장이 *창발*
       ↕  표준모형의 "근본" 게이지장(→ standard-model)
  ⇒ 무엇이 근본이고 무엇이 창발인가? (→ L5, L6)
  얽힘 엔트로피 ~ 경계 면적 → 상(phase)을 얽힘으로 분류 (→ L5 holography)
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~6개씩)
- 각 문서가 다루는 핵심 창발·유도 (3~4줄)
- 전체 문서 개수 확인 (35개 목표)
- Python + numpy + scipy 검증 환경(긴밀결합 밴드·천 수·BCS 갭·다체 대각화)
- L3 quantum-information(애니온)·standard-model(창발 게이지 대비) / L5 홀로그래피·창발 시공간 / L6 `emergence`(본체)로 이어지는 *창발 연결* 명시
- **More is Different(준입자 창발)** · **초전도/초유체(거시 양자)** · **위상이 분류(천 수·애니온)** · **창발 게이지장(근본 vs 창발 역설)**을 골격으로 분명히

**준비됐으면 1단계 구조 설계부터 시작해줘!**
