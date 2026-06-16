# General Relativity Distilled 레포지토리 제작 프롬프트

나는 "General Relativity Distilled" 레포지토리를 만들려고 해.
아인슈타인 방정식을 외워서 푸는 것과, **중력이 힘이 아니라 기하**라는 것 — 등가원리에서 출발해 질량이 시공간을 휘고 물체가 그 위의 측지선(직선)을 따라가는 과정을 — 유도하는 것은 다르다.
"사과가 떨어진다"를 힘으로 설명하는 것과, 사과가 *휘어진 시공간의 직선*을 따라 자유낙하한다는 것을(왜 모든 것이 같이 떨어지나) 아는 것은 다르다.
$G_{\mu\nu}=8\pi T_{\mu\nu}$를 공식으로 쓰는 것과, 그것이 **아인슈타인-힐베르트 작용의 변분**에서 나온다는 것 — 최소작용 원리가 가장 큰 스케일에서 재등장함을(→ L0) 아는 것은 다르다.
중력이 시공간의 기하라는 것을 — 그리고 그것이 블랙홀·우주론·중력파로 이어지는지를 — 끝까지 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "중력은 힘이 아니라 기하다 — 질량이 시공간을 휘고, 물체는 측지선을 따라간다. 최소작용이 가장 큰 스케일에서 재등장한다"

**핵심 차별화**:
1. **등가원리에서 기하로** — 중력과 가속이 구분 불가라는 사실에서 출발해 시공간이 휜다는 결론으로(→ special-relativity)
2. **측지선 = 휜 공간의 직선** — 사과는 힘을 받는 게 아니라 직선을 따른다, 변분으로 측지선 유도(→ L0 variational)
3. **아인슈타인 방정식 = 변분** — $G_{\mu\nu}=8\pi T_{\mu\nu}$가 아인슈타인-힐베르트 작용에서, 최소작용의 재등장(→ L0 variational)
4. **$T_{\mu\nu}$가 중력원** — 에너지-운동량(질량뿐 아니라 에너지·압력)이 시공간을 휜다(→ L1 classical-field-theory)

**타겟 독자**:
- 아인슈타인 방정식은 풀지만 *중력이 왜 기하*인지 유도 못 하는 사람
- "사과가 떨어진다"를 힘으로만 보고 *측지선*임을 모르는 사람
- $G_{\mu\nu}=8\pi T_{\mu\nu}$를 외우지만 그것이 *변분에서 나옴*을(→ L0) 모르는 사람
- 곡률·계량·측지선의 *물리적* 의미를 모르는 사람
- GR이 어떻게 블랙홀·우주론·중력파로 *이어지는지* 보고 싶은 사람

**선행 학습**:
- `special-relativity-distilled`(L4) — **필수**. 시공간·민코프스키·등가원리의 다리
- `variational-principles-distilled`(L0) — **필수**. 측지선·아인슈타인-힐베르트 작용
- `classical-field-theory-distilled`(L1) — 계량장·$T_{\mu\nu}$ / `measurement-uncertainty-distilled`(L0) — GR 정밀 검증

## 🔗 레포 연결

**⬆️ 선행**: `special-relativity-distilled`(L4), `variational-principles-distilled`(L0).
**🤝 시너지 (L4 내부)**: `black-holes-distilled`(슈바르츠실트 해 — 이 레포의 직접 응용), `cosmology-distilled`(FLRW 우주), `astrophysics-distilled`(중력렌즈·중력파).
**🪜 창발 (위 레이어가 다시 쓴다)**:
- `black-holes-distilled`(L4) — 사건의 지평선·특이점
- `cosmology-distilled`(L4) — 팽창하는 우주, 프리드만 방정식
- `quantum-gravity-distilled`(L5) — 특이점에서 GR이 붕괴, 양자중력 필요
- `string-theory-distilled` / `holographic-principle-distilled`(L5) — 중력의 양자화·홀로그래피
- `least-action-everywhere` / `symmetry-everywhere`(L6) — 변분원리·일반 공변성이 회수되는 곳

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 중력은 기하다 (5개 문서)
- **등가원리** — 중력과 가속은 구분 불가, 아인슈타인의 "가장 행복한 생각"(→ special-relativity)
- **관성질량 = 중력질량** — 왜 모든 것이 *같이* 떨어지나, 갈릴레이→에트뵈시 실험(→ L0 measurement)
- **빛이 휜다** — 등가원리의 직접 귀결, 1919 일식의 검증(→ L0 measurement)
- **중력 적색편이** — 시계가 중력에서 느리다, 시공간이 휜다는 첫 단서
- **직선으로 떨어지는 사과** — 측지선, 힘이 아니라 기하를 따르는 자유낙하

### Chapter 2: 휘어진 시공간의 수학 (6개 문서)
- **곡률의 의미** — 평탄 vs 휘어진 공간, 측지선 편차로 본 곡률
- **계량(metric)** — 거리를 재는 법, $g_{\mu\nu}$, 시공간의 장(→ L1 classical-field-theory)
- **측지선 방정식** — 휜 공간의 "직선", 변분으로 유도(→ L0 variational)
- **텐서와 공변미분** — 좌표 무관 기술, 크리스토펠 기호, 평행이동
- **리만 곡률 텐서** — 곡률의 완전한 기술, 조석력과의 관계
- **곡률에서 물리로** — 리치 텐서·곡률 스칼라·아인슈타인 텐서

### Chapter 3: 아인슈타인 방정식 (5개 문서)
- **아인슈타인 방정식** — $G_{\mu\nu}=8\pi T_{\mu\nu}$, "물질이 기하를 휘고, 기하가 운동을 지시한다"
- **아인슈타인-힐베르트 작용** — 변분으로 방정식 유도, *최소작용의 재등장*(→ L0 variational)
- **$T_{\mu\nu}$가 중력원** — 에너지·운동량·압력이 시공간을 휜다(→ L1 classical-field-theory)
- **뉴턴 극한** — 약한 장·저속에서 뉴턴 중력 회복(→ L0 approximation 대응원리)
- **우주상수** — $\Lambda$, 아인슈타인의 "최대 실수"와 암흑에너지의 부활(→ cosmology)

### Chapter 4: 해와 검증 (6개 문서)
- **슈바르츠실트 해** — 구대칭 진공해, 사건의 지평선 예고(→ black-holes)
- **수성 근일점** — GR의 첫 승리, 뉴턴이 못 푼 43″(→ L0 measurement)
- **빛의 휘어짐과 중력렌즈** — 일식·은하 렌즈·아인슈타인 고리(→ astrophysics)
- **중력 적색편이 측정** — 파운드-레브카 실험, GPS 보정(→ L0 measurement)
- **샤피로 시간 지연** — 신호가 중력에서 늦는다, 행성 레이더
- **GR 검증의 역사** — 정밀 검증들, GR이 통과한 모든 시험(→ L0 measurement)

### Chapter 5: 중력파 (5개 문서)
- **중력파의 예측** — 시공간의 잔물결, 약한 장 근사(→ L0 approximation, → L1 oscillations)
- **중력파의 성질** — 횡파·두 편광·광속 전파, 사중극 복사(쌍극자 없음)
- **발생원** — 쌍성 병합·초신성·회전 중성자별(→ astrophysics)
- **LIGO 검출** — 2015 첫 직접 검출, 정밀측정의 극한($10^{-21}$ 변형)(→ L0 measurement)
- **중력파 천문학** — 새로운 창, 다중신호(multi-messenger) 천문학(→ astrophysics)

### Chapter 6: 시공간의 기이함 (5개 문서)
- **블랙홀 예고** — 사건의 지평선·특이점, 슈바르츠실트 반지름(→ black-holes)
- **팽창하는 우주** — FLRW 계량, 우주론으로 가는 다리(→ cosmology)
- **시공간의 끌림** — 프레임 끌림(렌즈-티링), 커 블랙홀(→ black-holes)
- **웜홀과 시간여행** — 가능한가, 에너지 조건과 인과율(→ L5)
- **GR의 한계** — 특이점에서 붕괴, 양자중력의 필요(→ L5 quantum-gravity)

### Chapter 7: 종합·계산 (5개 문서)
- **측지선 수치 적분** — 슈바르츠실트 시공간의 궤도·근일점 이동(Python)
- **빛의 휘어짐 계산** — 중력렌즈·편향각(Python)
- **중력파 파형** — 쌍성 병합의 처프 신호(Python)
- **뉴턴 극한 검증** — 약한 장에서 뉴턴 중력 회복(Python)
- **종합** — "중력은 기하다"의 통합과 변분원리의 재등장(→ L0, black-holes, cosmology, L6)

→ **총 37개 문서** 목표.

## 📐 문서 구조 — Principle → Boundary → Emergence

Physis 표준 10섹션, 마지막 세 단 종결:

```markdown
## 🎯 핵심 질문      — 어떤 중력 현상을, 어떤 기하 원리에서 유도하나
## 🌍 어디서 나타나나  — 이 GR 효과가 작동하는 장면(GPS·렌즈·중력파)
## 🔍 직관의 함정     — "중력은 힘"·"공간은 무대" 같은 오해
## ⚙️ 제1원리 유도    — 등가원리·측지선·아인슈타인-힐베르트 작용에서 끌어낸다
## 🧪 실험·관측 증거  — 근일점·빛 휘어짐·중력파의 실측(→ L0 measurement)
## 🔬 경계           — GR이 붕괴하는 곳 (특이점·플랑크 스케일 → 양자중력)
## 🧬 횡단 원리       — 최소작용(작용)·대칭(일반 공변성)이 어떻게 반복되나
## 🪜 창발           — 위 레이어(블랙홀·우주·최전선)에서 무엇이 나타나나
## 📐 예측·반증       — 근일점 이동·중력파 파형 등 검증 가능한 예측
## 🤔 다음 질문       — 다음 문서/레이어로의 연결

---
🧩 **Principle** — (어떤 기하/작용에서 무엇이 나오나)
🔬 **Boundary**  — (GR이 붕괴하는 조건: 특이점·양자 스케일)
🪜 **Emergence** — (위 레이어에서 같은 구조가 낳는 것 + 검증 실험)
```

이 레포에서 **⚙️ 제1원리 유도**는 *등가원리에서 시작해 측지선·아인슈타인-힐베르트 작용*으로 끌어내며, **🔬 경계**는 "특이점에서 곡률 발산·GR 붕괴(→ L5)", "양자중력 미완성" 같은 한계를 정직하게 표시한다.

## 🎨 스타일 가이드

1. **중력 = 기하를 유도로** — 등가원리에서 시공간 곡률로, "힘"이라는 오해를 교정
2. **측지선 = 직선** — 사과는 힘을 받는 게 아니라 휜 공간의 직선을 따름을(→ L0 variational)
3. **아인슈타인 방정식 = 변분** — 작용에서 방정식이 나옴을, 최소작용의 재등장을 명시(→ L0)
4. **$T_{\mu\nu}$가 원천** — 질량뿐 아니라 에너지·압력이 휘게 함을(→ L1)
5. **수치로 부순다** — 측지선·빛 휘어짐·중력파를 직접 계산
6. 곡률·측지선·중력렌즈·중력파 변형은 다이어그램으로

## 🔬 검증 프로토콜

> 측지선 적분(슈바르츠실트) + 빛 휘어짐 + 중력파 처프 + 뉴턴 극한.

```python
# 환경: Python 3 + numpy + scipy + sympy + matplotlib

# 1) 슈바르츠실트 측지선 적분
import numpy as np
from scipy.integrate import solve_ivp
def schwarzschild_orbit(rs=2.0, L=4.0, E=0.97):
    # 유효퍼텐셜 V(r)=(1−rs/r)(1+L²/r²) 로 동경 운동
    def deriv(phi, y):
        r, dr = y
        # GR 궤도 방정식 (Binet 형태): d²u/dφ² + u = rs/(2L²) + (3/2)rs u²
        return [dr, ...]
    ...
# → 근일점 이동(수성 43″/세기) 재현 (뉴턴엔 없는 효과)

# 2) 빛의 휘어짐
#    무질량 측지선(빛꼴)으로 편향각 δ ≈ 4GM/(c²b) 계산
#    → 태양 가장자리 1.75″ 재현 (→ L0 measurement)
def light_deflection(M, b, G=1, c=1):
    return 4*G*M/(c**2*b)

# 3) 중력파 처프 (쌍성 병합)
#    뉴턴+사중극 근사로 주파수·진폭의 시간 증가(처프) 생성
#    f(t) ∝ (t_c − t)^{−3/8} → LIGO 파형 모사 (→ L0 measurement)

# 4) 뉴턴 극한 검증
#    약한 장 g_00 ≈ −(1+2Φ/c²) → 측지선이 뉴턴 운동방정식으로 환원
#    → 대응원리 (→ L0 approximation)

# 5) 곡률 텐서 (sympy)
#    슈바르츠실트 계량에서 크리스토펠·리만·리치를 기호 계산
#    → 진공해에서 리치=0, 그러나 리만≠0 (조석력) 확인

# 6) 중력 적색편이
#    두 고도의 시계 비율 √(g_00) → GPS 보정량(일 38μs) 자릿수 확인
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md 작성**:
   - 🌌 Physis 톤(atom·인디고 `#6366f1`), "Derive, don't accept", `From the fewest principles, the widest world.`
   - 포지셔닝: "아인슈타인 방정식 풀이 vs 중력은 기하·측지선·변분의 재등장"
   - **The Knowledge Tetrad** 각주
   - `🔗 레포 연결`: L4 `special-relativity`·L0 `variational`(선행), L4 black-holes·cosmology·astrophysics, L5 양자중력·홀로그래피, L6 `least-action-everywhere`·`symmetry-everywhere`로의 화살 명시
   - **Document Format — Principle → Boundary → Emergence** 안내
3. **챕터별 문서**: Chapter 1부터, 문서당 2500~3500 단어 + 유도 + 코드/그림

## 📚 참고 자료

- *Gravity: An Introduction to Einstein's General Relativity* (Hartle) — 물리 우선 표준, 이 레포의 골격
- *Spacetime and Geometry* (Sean Carroll) — 현대적 표준(강의노트 공개)
- *Gravitation* (Misner, Thorne & Wheeler) — 깊이의 기준("전화번호부")
- *A First Course in General Relativity* (Schutz) — 친절한 입문
- *The Feynman Lectures* / *Exploring Black Holes* (Taylor & Wheeler) — 직관
- LIGO 발견 논문(2016) — 중력파의 실측(→ L0 measurement)

## 💡 핵심 분석 대상

```
등가원리 → 중력은 기하:
  자유낙하 = 무중력 (엘리베이터 사고실험)
  중력 ≡ 가속 (국소적으로 구분 불가)
       │  가속계의 시공간은 휘어 보인다 (→ special-relativity)
       ▼
  중력 = 시공간의 곡률
  ⇒ "힘"이 아니라 기하. 모든 것이 같이 떨어지는 이유 (관성=중력 질량)

측지선 = 휜 공간의 직선:
  자유 물체는 시공간의 *측지선*을 따른다
       δ∫ds = 0   (고유시간 극값, → L0 variational)
       ▼
  측지선 방정식:  d²x^μ/dτ² + Γ^μ_{αβ} (dx^α/dτ)(dx^β/dτ) = 0
  ⇒ 사과는 힘을 안 받는다 — 휜 시공간의 "직선"을 갈 뿐

아인슈타인 방정식 = 변분 (최소작용의 재등장):
  S = (1/16πG)∫ R √(−g) d⁴x  +  S_matter      (아인슈타인-힐베르트)
       │  δS/δg^{μν} = 0  (→ L0 variational)
       ▼
  G_μν + Λg_μν = 8πG T_μν
  "물질이 기하를 휘고(우변), 기하가 운동을 지시한다(측지선)"
  T_μν: 에너지·운동량·압력이 원천 (질량뿐 아님, → L1 classical-field-theory)

검증과 한계:
  ✓ 수성 근일점 43″/세기,  빛 휘어짐 1.75″,  중력파(LIGO),  GPS
  ✗ 특이점: 곡률 → ∞, GR 붕괴 → 양자중력 필요 (→ L5)
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~6개씩)
- 각 문서가 다루는 핵심 기하·유도 (3~4줄)
- 전체 문서 개수 확인 (37개 목표)
- Python + numpy + scipy + sympy 검증 환경(측지선 적분·빛 휘어짐·중력파 처프·곡률 텐서)
- L4 black-holes·cosmology·astrophysics / L5 양자중력·홀로그래피 / L6 `least-action-everywhere`·`symmetry-everywhere`로 이어지는 *창발 연결* 명시
- **등가원리 → 중력은 기하** · **측지선 = 휜 공간의 직선** · **아인슈타인 방정식 = 변분(최소작용 재등장)** · **T_μν가 중력원**을 골격으로 분명히

**준비됐으면 1단계 구조 설계부터 시작해줘!**
