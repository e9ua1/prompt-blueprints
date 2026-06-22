# IQ Lab · Prompt Blueprints

> **"X를 하는 것과, X가 왜·어떻게 그렇게 되는지 아는 것은 다르다."**

이 저장소는 **딥다이브(deep-dive) 학습 레포지토리를 통째로 생성하는 메타 프롬프트(meta-prompt) 모음**입니다.
`blueprints/` 아래의 각 `.md` 파일은 단순한 문서가 아니라, 코딩 에이전트(Claude Code, Cursor, Codex 등)에게 그대로 건네면 **하나의 완결된 학습 레포 한 채(수십 개 문서 + README + 실험 환경)** 를 설계·작성하게 만드는 **청사진(blueprint)** 입니다.

현재 **233개**의 블루프린트가 다섯 갈래로 정리되어 있습니다. 처음 둘은 **기술(만드는 법)**, 나머지 셋은 그리스어로 이름 붙인 **세계(돌아가는 원리)** — 자연·마음·실천지입니다.

| 갈래 | 개수 | 범위 |
|------|------|------|
| 🤖 **AI** (`blueprints/ai`) | **48** | 수학 기초(Layer 0)부터 LLM·해석가능성(Layer 6)까지, 11개 레이어의 수직 커리큘럼 |
| 🛠️ **Dev** (`blueprints/dev`) | **87** | JVM·Spring·DB·프론트엔드·모바일·인프라·아키텍처 등 17개 카테고리의 시스템 내부 원리 |
| ⚛️ **Physis** (`blueprints/physis`) | **36** | 자연(φύσις) — 변분원리·대칭에서 양자·상대성·양자중력·창발까지, 물리학 7레이어 |
| 🧠 **Psyche** (`blueprints/psyche`) | **31** | 마음(ψυχή) — 뉴런·인지에서 의식의 어려운 문제·자아까지, 마음 과학 7레이어 |
| 🧭 **Phronesis** (`blueprints/phronesis`) | **31** | 실천지(φρόνησις) — 의사결정·돈·전략에서 게임이론·복리까지, 세상 작동법 6레이어 |

---

## 🎯 이 저장소가 푸는 문제

대부분의 학습 자료는 **"무엇을, 어떻게 쓰는가"** 에서 멈춥니다.
이 블루프린트들이 만들어내는 레포는 한 단계 더 내려갑니다 — **"왜 이렇게 설계됐는가"**, 그리고 **"그 주장을 어떻게 증명하는가"**.

모든 블루프린트는 같은 첫 문장 패턴으로 시작합니다.

> *"`np.linalg.svd`를 **호출하는 것**과, 모든 행렬이 왜 회전·스케일·회전으로 분해되는지 **증명할 수 있는 것**은 다르다."*
> *"`synchronized`를 **쓰는 것**과, JVM이 삽입하는 Memory Barrier를 **어셈블리로 확인하는 것**은 다르다."*

이 "다름"을 끝까지 파고드는 것이 전부입니다.

### 세 가지 공통 원칙

1. **왜(Why) 중심** — "무엇인가"가 아니라 "왜 이렇게 설계됐는가"를 질문의 축으로 삼는다.
2. **모든 주장은 증거로** — 글로 설명하지 않는다. 도구 출력(`javap`, `JFR`, `JMH`, `EXPLAIN`, `strace`…), 실행 가능한 코드, 또는 수학적 증명으로 보인다.
3. **블랙박스 분해** — "GC가 메모리를 수거합니다" 같은 한 줄을 알고리즘 단계까지 해부한다.

---

## 🗂️ 디렉토리 구조

```
prompt-blueprints/
├── README.md              ← 지금 보는 문서 (전체 카탈로그 + 사용법)
├── core-style-guide.md    ← 모든 블루프린트·생성 레포가 따르는 공통 규약
└── blueprints/
    ├── ai/                ← 48개 · 11 레이어 수직 커리큘럼 (layer0 … layer6)
    │   ├── layer0/  …  layer6/
    ├── dev/               ← 87개 · 17 카테고리
    │   ├── java core/        spring ecosystem/    database/
    │   ├── frontend : web/   mobile — android/    mobile — ios/
    │   ├── infrastructure & devOps/   architecture & design/
    │   ├── data engineering/ messaging & streaming/  …
    │   └── synthesis/        ← 여러 기술을 가로지르는 횡단 비교 레포
    ├── physis/            ← 36개 · 물리/자연 (L0 … L6)
    ├── psyche/            ← 31개 · 마음/의식 (L0 … L6)
    └── phronesis/         ← 31개 · 실천지/세상 작동법 (L0 … L5)
```

> **파일명 규칙:** `…-deep-dive`(AI·Dev, 완전 분해형) · `…-distilled`(Physis·Psyche·Phronesis, 본질 증류형) · `…-everywhere`(각 갈래 상위 레이어의 **횡단 종합** — "이 원리가 모든 곳에 있다").

---

## 🚀 사용법 — 블루프린트 한 장으로 레포 만들기

1. **블루프린트 하나를 고른다.** (예: `blueprints/dev/java core/prompt-jvm-deep-dive.md`)
2. **파일 전체를 코딩 에이전트에 그대로 붙여넣는다.** 블루프린트는 이미 완결된 지시문이라 추가 설명이 거의 필요 없다.
3. 에이전트가 **디렉토리 생성 → README 작성 → 섹션별 문서 작성** 순서로 레포를 채워나간다.
4. 생성되는 모든 문서는 [`core-style-guide.md`](core-style-guide.md)의 **10섹션 표준 구조**를 따른다.

### 두 가지 블루프린트 변형

| 변형 | 1단계의 성격 | 대표 예시 |
|------|--------------|-----------|
| **구조 확정형** | 챕터·문서 목록이 블루프린트에 **고정**되어 있다. 1단계를 건너뛰고 바로 작성. | `prompt-jvm-deep-dive.md` (9섹션 69문서 확정) |
| **구조 설계형** | 1단계에서 에이전트가 **전체 구조를 먼저 설계**하고, 합의 후 작성. | `prompt-transformer-deep-dive.md` (7챕터 36문서 목표) |

> 💡 **Synthesis(횡단 비교) 블루프린트**는 한 기술이 아니라 여러 기술이 *같은 문제에 다르게 답한 방식*을 나란히 비교하는 특수 변형입니다. 선행 레포들을 입력으로 삼아 "수렴과 발산"을 그립니다. (`blueprints/dev/synthesis/`)

---

## 🤖 AI 카탈로그 (48개)

아래 **Layer 0 → 6**은 수학 기초에서 출발해 위로 쌓이는 **수직 커리큘럼**입니다. 상위 레이어 블루프린트는 하위 레이어를 *선행 학습*으로 명시적으로 참조합니다.

<details>
<summary><b>Layer 0 — 수학적 기초 (10)</b></summary>

> 모든 ML이 딛고 선 땅. 공리에서 출발해 직접 유도한다.

| 블루프린트 | 컨셉 |
|------------|------|
| [Linear Algebra](blueprints/ai/layer0/prompt-linear-algebra-deep-dive.md) | 행렬은 숫자 상자가 아니라 벡터공간 사이의 선형 사상이다 |
| [Calculus & Optimization](blueprints/ai/layer0/prompt-calculus-optimization-deep-dive.md) | 신경망 학습은 고차원 공간에서의 기울기 기반 최적화다 |
| [Probability Theory](blueprints/ai/layer0/prompt-probability-theory-deep-dive.md) | 확률은 측도다 — 모든 확률 개념은 (Ω, ℱ, ℙ)에서 출발한다 |
| [Mathematical Statistics](blueprints/ai/layer0/prompt-mathematical-statistics-deep-dive.md) | 관측 데이터로 파라미터를 거꾸로 추론하는 수학 |
| [Information Theory](blueprints/ai/layer0/prompt-information-theory-deep-dive.md) | 정보는 '놀라움의 로그'다 — 엔트로피·KL·MI가 하나에서 유도된다 |
| [Convex Optimization](blueprints/ai/layer0/prompt-convex-optimization-deep-dive.md) | 볼록성은 최적화에 전역성을 부여하는 유일한 구조다 |
| [Functional Analysis](blueprints/ai/layer0/prompt-functional-analysis-deep-dive.md) | 무한차원의 선형대수 — 커널·NTK·SDE의 이론적 기반 |
| [Stochastic Processes](blueprints/ai/layer0/prompt-stochastic-processes-deep-dive.md) | 시간에 따라 진화하는 확률 구조 — 마르코프·정상성·에르고딕성 |
| [Stochastic Differential Equations](blueprints/ai/layer0/prompt-sde-deep-dive.md) | 브라운 운동을 미분·적분하는 수학, Diffusion의 토대 |
| [Information Geometry](blueprints/ai/layer0/prompt-information-geometry-deep-dive.md) | 확률분포 공간의 기하학 — Fisher 정보의 리만 구조 |

</details>

<details>
<summary><b>Layer 1 — 고전 ML & 학습 이론 (5)</b></summary>

> 딥러닝 이전, 그리고 그 밑에 깔린 학습의 수학.

| 블루프린트 | 컨셉 |
|------------|------|
| [ML Fundamentals](blueprints/ai/layer1/prompt-ml-fundamentals-deep-dive.md) | 고전 ML을 증명 가능하게 재구성 — 회귀·트리·앙상블 |
| [Statistical Learning Theory](blueprints/ai/layer1/prompt-statistical-learning-theory-deep-dive.md) | 왜 학습이 가능한가 — PAC·VC·Rademacher |
| [Kernel Methods](blueprints/ai/layer1/prompt-kernel-methods-deep-dive.md) | SVM·GP·MMD·Kernel PCA를 하나의 RKHS로 |
| [Bayesian ML](blueprints/ai/layer1/prompt-bayesian-ml-deep-dive.md) | 불확실성의 ML — 변분 추론·MCMC·Bayesian DL |
| [Graphical Models](blueprints/ai/layer1/prompt-graphical-models-deep-dive.md) | 조건부 독립이 그래프 기하로 나타나는 아름다움 |

</details>

<details>
<summary><b>Layer 2 — 딥러닝 이론 (4)</b></summary>

> 신경망이 *왜* 동작하고 *왜* 일반화하는가.

| 블루프린트 | 컨셉 |
|------------|------|
| [Neural Network Theory](blueprints/ai/layer2/prompt-neural-network-theory-deep-dive.md) | 표현력·학습·초기화·아키텍처의 엄밀한 분석 |
| [Optimization Theory](blueprints/ai/layer2/prompt-optimization-theory-deep-dive.md) | 수렴률·발산 반례·Loss Landscape·실전 튜닝 |
| [Generalization Theory](blueprints/ai/layer2/prompt-generalization-theory-deep-dive.md) | 왜 딥러닝이 일반화하는가 — 고전이론의 실패와 현대적 설명 |
| [Regularization Theory](blueprints/ai/layer2/prompt-regularization-theory-deep-dive.md) | Bayesian prior·앙상블·Normalization을 하나의 프레임으로 |

</details>

<details>
<summary><b>Layer 3 — 핵심 아키텍처 (5)</b></summary>

> 현대 딥러닝을 떠받치는 다섯 골격.

| 블루프린트 | 컨셉 |
|------------|------|
| [CNN](blueprints/ai/layer3/prompt-cnn-deep-dive.md) | 대칭성·지역성·계층성을 표현력 이론으로 재구성 |
| [RNN & LSTM](blueprints/ai/layer3/prompt-rnn-lstm-deep-dive.md) | RNN에서 Transformer까지의 진화를 BPTT·Gradient로 |
| [Transformer](blueprints/ai/layer3/prompt-transformer-deep-dive.md) | Attention의 수학 — self/Linear/Sparse/Flash 완전 분해 |
| [Graph Neural Network](blueprints/ai/layer3/prompt-gnn-deep-dive.md) | Spectral·Spatial·Message Passing 통합, WL 표현력 |
| [Generative Model](blueprints/ai/layer3/prompt-generative-model-deep-dive.md) | 암묵적(GAN) vs 명시적(VAE·Flow·Diffusion) likelihood |

</details>

<details>
<summary><b>Layer 4-A — 강화학습 (6)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [RL Foundations](blueprints/ai/layer4-A/prompt-rl-foundations-deep-dive.md) | MDP·Bellman·DP의 엄밀한 증명 |
| [Model-Free RL](blueprints/ai/layer4-A/prompt-model-free-rl-deep-dive.md) | Monte Carlo·TD·Q-Learning 수렴 증명 |
| [Policy Gradient](blueprints/ai/layer4-A/prompt-policy-gradient-deep-dive.md) | Log-Derivative Trick부터 Natural Gradient까지 |
| [Deep RL](blueprints/ai/layer4-A/prompt-deep-rl-deep-dive.md) | Tabular에서 Atari·Go까지, trick의 수학적 정당화 |
| [Advanced RL](blueprints/ai/layer4-A/prompt-advanced-rl-deep-dive.md) | TRPO·PPO·SAC·TD3를 monotonic improvement로 통합 |
| [RL Theory](blueprints/ai/layer4-A/prompt-rl-theory-deep-dive.md) | Bandit부터 PAC-MDP까지 sample complexity 이론 |

</details>

<details>
<summary><b>Layer 4-B — 대규모 언어 모델 (4)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [LLM Pretraining](blueprints/ai/layer4-B/prompt-llm-pretraining-deep-dive.md) | Scaling Law·훈련 안정성·데이터·평가의 이론적 기반 |
| [LLM Alignment](blueprints/ai/layer4-B/prompt-llm-alignment-deep-dive.md) | RLHF·DPO·CAI를 preference learning으로 통합 |
| [LLM Inference](blueprints/ai/layer4-B/prompt-llm-inference-deep-dive.md) | Memory·Throughput·Latency의 공학적 trade-off |
| [LLM Efficiency](blueprints/ai/layer4-B/prompt-llm-efficiency-deep-dive.md) | Parameter·Memory·Compute·Latency 모든 축의 효율화 |

</details>

<details>
<summary><b>Layer 4-C — 비전 & 생성 (4)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Vision Transformer](blueprints/ai/layer4-C/prompt-vision-transformer-deep-dive.md) | ViT부터 SSL·Hierarchical·Multimodal까지 |
| [Object Detection](blueprints/ai/layer4-C/prompt-object-detection-deep-dive.md) | Two-stage·One-stage·Transformer 계보 통합 |
| [Diffusion Model](blueprints/ai/layer4-C/prompt-diffusion-model-deep-dive.md) | DDPM·Score-SDE·DDIM·Flow Matching을 stochastic process로 |
| [3D & Neural Rendering](blueprints/ai/layer4-C/prompt-3d-neural-rendering-deep-dive.md) | PBR부터 NeRF·Gaussian Splatting·3D Generation까지 |

</details>

<details>
<summary><b>Layer 4-D — 자연어 처리 (2)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [NLP Foundations](blueprints/ai/layer4-D/prompt-nlp-foundations-deep-dive.md) | Distributional Hypothesis부터 Subword Tokenization까지 |
| [Pretrained LM](blueprints/ai/layer4-D/prompt-pretrained-lm-deep-dive.md) | BERT·GPT·T5 objective 분해와 전이학습·ICL 이론 |

</details>

<details>
<summary><b>Layer 4-E — 오디오 & 음성 (1)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Audio & Speech](blueprints/ai/layer4-E/prompt-audio-speech-deep-dive.md) | Signal Processing부터 Neural Codec·Speech LLM까지 |

</details>

<details>
<summary><b>Layer 5 — 시스템 & 프로덕션 (4)</b></summary>

> 모델이 아니라 모델을 *돌리는* 인프라의 수학.

| 블루프린트 | 컨셉 |
|------------|------|
| [PyTorch Internals](blueprints/ai/layer5/prompt-pytorch-internals-deep-dive.md) | Autograd·Dispatcher·CUDA·AMP의 시스템 수학 |
| [Distributed Training](blueprints/ai/layer5/prompt-distributed-training-deep-dive.md) | Data/Model/Pipeline Parallelism의 메모리·통신 비용 |
| [Efficient ML](blueprints/ai/layer5/prompt-efficient-ml-deep-dive.md) | Compression·Acceleration·Kernel·Serving 통합 |
| [Experimental Statistics & MLOps](blueprints/ai/layer5/prompt-experimental-statistics-mlops-deep-dive.md) | Data·Model·Experiment 품질 보장과 statistical rigor |

</details>

<details>
<summary><b>Layer 6 — 프론티어 (3)</b></summary>

> 지금 가장 빠르게 움직이는 최전선.

| 블루프린트 | 컨셉 |
|------------|------|
| [LLM Reasoning](blueprints/ai/layer6/prompt-llm-reasoning-deep-dive.md) | Test-time Compute·Reasoning·Agent의 수학 |
| [Mechanistic Interpretability](blueprints/ai/layer6/prompt-mechanistic-interpretability-deep-dive.md) | Circuit·Feature·Mechanism의 수학적 증명 |
| [Retrieval & RAG](blueprints/ai/layer6/prompt-retrieval-rag-deep-dive.md) | BM25부터 Dense Retrieval·ANN·Hybrid·GraphRAG까지 |

</details>

---

## 🛠️ Dev 카탈로그 (87개)

시스템의 **내부 원리**를 도구 출력으로 증명하는 백엔드·프론트엔드·모바일·인프라 레포들입니다.
경로에 공백·특수문자가 있어 링크는 꺾쇠(`< >`)로 감쌌습니다.

<details>
<summary><b>☕ Java Core (7)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [JVM Deep Dive](<blueprints/dev/java core/prompt-jvm-deep-dive.md>) | Java 코드를 작성하는 것과, 그것이 어떻게 살아 움직이는지 아는 것은 다르다 |
| [Java Concurrency](<blueprints/dev/java core/prompt-java-concurrency-deep-dive.md>) | Virtual Thread가 캐리어 스레드 위에서 어떻게 스케줄링되는가 |
| [Modern Java in Action](<blueprints/dev/java core/prompt-modern-java-in-action.md>) | 자바 8~21, 함수형 패러다임이 자바를 어떻게 바꿨는가 |
| [Java API Reference](<blueprints/dev/java core/prompt-java-api-reference.md>) | 암기가 아닌 이해, 요약이 아닌 통찰 |
| [Java Design Patterns](<blueprints/dev/java core/prompt-java-design-patterns.md>) | 확장 가능하고 유지보수하기 쉬운 설계로 |
| [오브젝트 (Object)](<blueprints/dev/java core/prompt-object.md>) | 이론이 먼저가 아니라 코드가 먼저다 |
| [Unit Testing](<blueprints/dev/java core/prompt-unit-testing.md>) | 테스트를 작성하는 것과, 올바른 테스트를 작성하는 것은 다르다 |

</details>

<details>
<summary><b>🌱 Spring Ecosystem (8)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Spring Core](<blueprints/dev/spring ecosystem/prompt-spring-core-deep-dive.md>) | 스프링 컨테이너가 Bean을 관리하는 모든 비밀 |
| [Spring Boot Internals](<blueprints/dev/spring ecosystem/prompt-spring-boot-internals.md>) | Spring Boot가 어떻게 마법을 부리는지 아는 것은 다르다 |
| [Spring MVC](<blueprints/dev/spring ecosystem/prompt-spring-mvc-deep-dive.md>) | HTTP 요청이 Controller 메서드에 도달하는 전체 여정 |
| [Spring Data & Transaction](<blueprints/dev/spring ecosystem/prompt-spring-data-transaction-deep-dive.md>) | Repository가 어떻게 작동하는지 아는 것은 다르다 |
| [Spring Security](<blueprints/dev/spring ecosystem/prompt-spring-security-deep-dive.md>) | 요청이 Filter Chain을 거쳐 인증되는 전체 여정 |
| [Spring WebFlux](<blueprints/dev/spring ecosystem/prompt-spring-webflux-deep-dive.md>) | 스레드를 늘리는 것과, I/O 대기 시간을 없애는 것은 다르다 |
| [Spring Batch](<blueprints/dev/spring ecosystem/prompt-spring-batch-deep-dive.md>) | 수백만 건 데이터를 안정적으로 처리하는 메커니즘 |
| [Spring Cloud](<blueprints/dev/spring ecosystem/prompt-spring-cloud-deep-dive.md>) | 분산 시스템을 위한 Spring Cloud 패턴 완전 정복 |

</details>

<details>
<summary><b>🗄️ Database (6)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Database Internals](<blueprints/dev/database/prompt-database-internals.md>) | SQL을 쓰는 것과, DB가 내부에서 무슨 일을 하는지 아는 것은 다르다 |
| [MySQL](<blueprints/dev/database/prompt-mysql-deep-dive.md>) | 운영하는 것과, 장애에서 원인을 찾고 튜닝하는 것은 다르다 |
| [PostgreSQL](<blueprints/dev/database/prompt-postgresql-deep-dive.md>) | PostgreSQL이 MVCC를 완전히 다른 방식으로 구현하는 트레이드오프 |
| [Redis](<blueprints/dev/database/prompt-redis-deep-dive.md>) | 캐시로 쓰는 것과, 데이터가 어떻게 저장·만료되는지 아는 것 |
| [Elasticsearch](<blueprints/dev/database/prompt-elasticsearch-deep-dive.md>) | Lucene이 역색인을 구축하고 샤드에서 스코어를 병합하는 방식 |
| [DB Migration](<blueprints/dev/database/prompt-db-migration-deep-dive.md>) | 스키마 변경을 코드처럼 관리하고 안전하게 적용하는 것 |

</details>

<details>
<summary><b>🌐 Frontend : Web (13)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [JavaScript](<blueprints/dev/frontend : web/prompt-javascript-deep-dive.md>) | 실행 컨텍스트·스코프·프로토타입이 명세 수준에서 동작하는 방식 |
| [TypeScript Type System](<blueprints/dev/frontend : web/prompt-typescript-type-system-deep-dive.md>) | 타입 시스템이 어떻게 추론·검사하고 무엇을 보장(못)하는가 |
| [V8 Engine](<blueprints/dev/frontend : web/prompt-v8-engine-deep-dive.md>) | V8이 JS를 바이트코드→JIT로 변환하고 객체를 표현하는 방식 |
| [Event Loop & Async](<blueprints/dev/frontend : web/prompt-event-loop-async-deep-dive.md>) | 이벤트 루프가 어떤 순서·어떤 큐로 처리하는가 |
| [React Internals](<blueprints/dev/frontend : web/prompt-react-internals-deep-dive.md>) | Fiber·재조정·Hooks가 내부에서 어떻게 동작하는가 |
| [Frontend State Management](<blueprints/dev/frontend : web/prompt-frontend-state-management-deep-dive.md>) | 각 패러다임이 변경을 어떻게 추적·전파하는가 |
| [Browser Rendering](<blueprints/dev/frontend : web/prompt-browser-rendering-deep-dive.md>) | 브라우저가 CSS를 픽셀로 바꾸는 파이프라인 |
| [CSS Engine & Layout](<blueprints/dev/frontend : web/prompt-css-engine-layout-deep-dive.md>) | 레이아웃 엔진이 박스의 크기·위치를 계산하는 알고리즘 |
| [Rendering Strategy](<blueprints/dev/frontend : web/prompt-rendering-strategy-deep-dive.md>) | HTML이 어디서·언제 생성되고 어떻게 인터랙티브해지는가 |
| [Frontend Build Tools](<blueprints/dev/frontend : web/prompt-frontend-build-tools-deep-dive.md>) | 모듈 해석·변환·번들링·트리셰이킹의 내부 동작 |
| [Web Performance](<blueprints/dev/frontend : web/prompt-web-performance-deep-dive.md>) | 각 성능 지표가 어느 메커니즘에서 나오는지 알고 근본을 고치기 |
| [Web APIs & WASM](<blueprints/dev/frontend : web/prompt-web-apis-wasm-deep-dive.md>) | DOM·fetch·Worker의 내부 구현과 WASM 경계 |
| [Real-time & Client Networking](<blueprints/dev/frontend : web/prompt-realtime-client-networking-deep-dive.md>) | WebSocket·SSE·WebRTC가 내부에서 동작하는 방식 |

</details>

<details>
<summary><b>📱 Mobile — Android (8)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Android Architecture](<blueprints/dev/mobile — android/prompt-android-architecture-deep-dive.md>) | 단방향 흐름·의존성 규칙·모듈 경계가 무엇을 보장하는가 |
| [Android Framework Internals](<blueprints/dev/mobile — android/prompt-android-framework-internals-deep-dive.md>) | 시스템이 Binder로 앱과 통신하며 콜백을 호출하는 흐름 |
| [Android Runtime (ART)](<blueprints/dev/mobile — android/prompt-android-runtime-deep-dive.md>) | ART가 DEX를 컴파일·실행하고 메모리를 관리하는 방식 |
| [Android Build System](<blueprints/dev/mobile — android/prompt-android-build-system-deep-dive.md>) | 빌드 단계·R8 최적화·KSP가 어떻게 동작하는가 |
| [Android Performance](<blueprints/dev/mobile — android/prompt-android-performance-deep-dive.md>) | 프레임·시작·메모리 비용을 측정으로 특정해 고치기 |
| [Jetpack Compose Internals](<blueprints/dev/mobile — android/prompt-jetpack-compose-internals-deep-dive.md>) | Composer가 Slot Table에 기록하고 Recomposition을 좁히는 방식 |
| [Kotlin](<blueprints/dev/mobile — android/prompt-kotlin-deep-dive.md>) | suspend가 상태머신으로 컴파일되고 구조적 동시성이 보장하는 것 |
| [Local-first & Sync (CRDT)](<blueprints/dev/mobile — android/prompt-local-first-sync-deep-dive.md>) | 로컬 우선 + 최종 수렴으로 오프라인에서도 즉각 반응하는 앱 |

</details>

<details>
<summary><b>🍎 Mobile — iOS (7)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Swift](<blueprints/dev/mobile — ios/prompt-swift-deep-dive.md>) | ARC·값 타입·프로토콜이 메모리와 디스패치를 다루는 방식 |
| [Swift Concurrency](<blueprints/dev/mobile — ios/prompt-swift-concurrency-deep-dive.md>) | async/await가 상태머신으로 컴파일되고 Actor가 격리하는 방식 |
| [Objective-C Runtime](<blueprints/dev/mobile — ios/prompt-objc-runtime-deep-dive.md>) | objc_msgSend가 런타임에 메서드를 찾아 디스패치하는 방식 |
| [iOS Lifecycle & RunLoop](<blueprints/dev/mobile — ios/prompt-ios-lifecycle-runloop-deep-dive.md>) | RunLoop가 메인 스레드를 돌리고 상태를 전환하는 방식 |
| [UIKit & Core Animation](<blueprints/dev/mobile — ios/prompt-uikit-core-animation-deep-dive.md>) | CALayer가 렌더 서버에서 GPU로 합성되는 파이프라인 |
| [SwiftUI Internals](<blueprints/dev/mobile — ios/prompt-swiftui-internals-deep-dive.md>) | View Identity·상태 저장·AttributeGraph 갱신의 내부 |
| [iOS Performance](<blueprints/dev/mobile — ios/prompt-ios-performance-deep-dive.md>) | hitch·hang·런치·메모리 비용을 Instruments로 특정해 고치기 |

</details>

<details>
<summary><b>🏗️ Infrastructure & DevOps (9)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Linux for Backend](<blueprints/dev/infrastructure & devOps/prompt-linux-for-backend-deep-dive.md>) | 커널이 I/O·메모리를 관리해 앱 성능을 결정하는 방식 |
| [Network](<blueprints/dev/infrastructure & devOps/prompt-network-deep-dive.md>) | HTTP를 쓰는 것과, TCP가 연결을 맺고 끊는 방식을 아는 것 |
| [Docker](<blueprints/dev/infrastructure & devOps/prompt-docker.md>) | 컨테이너를 실행하는 것을 넘어 Docker가 실제로 동작하는 방식 |
| [Kubernetes](<blueprints/dev/infrastructure & devOps/prompt-kubernetes-deep-dive.md>) | Control Plane이 파드 상태를 선언적으로 일치시키는 방식 |
| [CI/CD Pipeline](<blueprints/dev/infrastructure & devOps/prompt-cicd-deep-dive.md>) | 파이프라인이 코드를 신뢰할 수 있는 결과물로 만드는 방식 |
| [IaC](<blueprints/dev/infrastructure & devOps/prompt-iac-deep-dive.md>) | state·의존성 그래프·Provider가 선언을 자원으로 수렴시키는 원리 |
| [Observability](<blueprints/dev/infrastructure & devOps/prompt-observability-deep-dive.md>) | 분산 추적이 서비스 경계를 넘어 컨텍스트를 전파하는 방식 |
| [Service Mesh](<blueprints/dev/infrastructure & devOps/prompt-service-mesh-deep-dive.md>) | Sidecar가 트래픽을 가로채고 컨트롤 플레인이 제어하는 방식 |
| [Git In-Depth](<blueprints/dev/infrastructure & devOps/prompt-git-in-depth.md>) | `.git` 디렉토리 안에서 무슨 일이 — Plumbing부터 Porcelain까지 |

</details>

<details>
<summary><b>🧱 Architecture & Design (5)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Architecture Patterns](<blueprints/dev/architecture & design/prompt-architecture-patterns.md>) | 아키텍처는 폴더 구조가 아니다 — 변경이 전파되는 방향이다 |
| [DDD](<blueprints/dev/architecture & design/prompt-ddd-deep-dive.md>) | 도메인의 불변식(Invariant)을 보호하는 객체 설계 |
| [CQRS + Event Sourcing](<blueprints/dev/architecture & design/prompt-cqrs-event-sourcing.md>) | 왜 이벤트가 진실의 원천이 되어야 하는가 |
| [MSA](<blueprints/dev/architecture & design/prompt-msa-deep-dive.md>) | 왜 그 경계에서 쪼개고, 이후 데이터 일관성을 어떻게 보장하는가 |
| [System Design](<blueprints/dev/architecture & design/prompt-system-design-deep-dive.md>) | 기술을 아는 것과, 수백만 사용자용 시스템을 설계하는 것 |

</details>

<details>
<summary><b>🔀 Foundations (5)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Computer Architecture](<blueprints/dev/foundations/prompt-computer-architecture-deep-dive.md>) | 코드가 CPU·캐시·메모리에서 실제로 무슨 일을 하는가 |
| [Compiler & Language Implementation](<blueprints/dev/foundations/prompt-compiler-deep-dive.md>) | 컴파일러가 언어를 해석·검증·번역하는 방식 |
| [Distributed Systems Theory](<blueprints/dev/foundations/prompt-distributed-systems-theory-deep-dive.md>) | 분산 시스템이 왜 어렵고 무엇이 보장 가능한가 |
| [Cryptography](<blueprints/dev/foundations/prompt-cryptography-deep-dive.md>) | 그 암호가 왜 안전하고 어떻게 깨지는가 |
| [GPU & Graphics Pipeline](<blueprints/dev/foundations/prompt-gpu-graphics-deep-dive.md>) | GPU가 정점을 픽셀로 바꾸는 과정과 성능 설계 |

</details>

<details>
<summary><b>🔌 Data Engineering (3)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Spark Internals](<blueprints/dev/data engineering/prompt-spark-internals-deep-dive.md>) | DAG가 Stage로 나뉘고 Shuffle·메모리가 성능을 결정하는 방식 |
| [Stream Processing](<blueprints/dev/data engineering/prompt-stream-processing-deep-dive.md>) | 이벤트 시간·Watermark·상태·체크포인트가 정확성을 보장하는 방식 |
| [Columnar & Storage Format](<blueprints/dev/data engineering/prompt-columnar-storage-format-deep-dive.md>) | 컬럼 저장·인코딩·통계가 분석 쿼리를 빠르게 하는 방식 |

</details>

<details>
<summary><b>📨 Messaging & Streaming (2)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Kafka](<blueprints/dev/messaging & streaming/prompt-kafka-deep-dive.md>) | Kafka가 순서와 내구성을 보장하는 방식 |
| [RabbitMQ](<blueprints/dev/messaging & streaming/prompt-rabbitmq-deep-dive.md>) | Exchange·Routing Key·Queue, 그리고 실패한 메시지의 행방 |

</details>

<details>
<summary><b>🦀 Languages (Rust, Go) (2)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Rust](<blueprints/dev/languages (rust, go)/prompt-rust-deep-dive.md>) | 컴파일을 통과시키는 것과, 소유권 모델이 무엇을 증명하는지 아는 것 |
| [Go](<blueprints/dev/languages (rust, go)/prompt-go-deep-dive.md>) | 런타임 스케줄러가 고루틴을 OS 스레드에 다중화하는 방식 |

</details>

<details>
<summary><b>🌉 Cross Platform (3)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Flutter](<blueprints/dev/cross platform/prompt-flutter-deep-dive.md>) | 3-Tree와 자체 렌더 엔진이 픽셀을 직접 그리는 방식 |
| [React Native](<blueprints/dev/cross platform/prompt-react-native-deep-dive.md>) | JS와 네이티브가 통신하는 경계와 그 진화 |
| [Kotlin Multiplatform](<blueprints/dev/cross platform/prompt-kotlin-multiplatform-deep-dive.md>) | 한 Kotlin 소스가 여러 타깃 백엔드로 컴파일되는 방식 |

</details>

<details>
<summary><b>🔗 API & Communication (1)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [gRPC + Protocol Buffers](<blueprints/dev/api & communication/prompt-grpc-deep-dive.md>) | 타입 안전한 계약으로 서비스를 연결하고 통신 비용을 줄이기 |

</details>

<details>
<summary><b>🛡️ Security Engineering (1)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Security Engineering](<blueprints/dev/security engineering/prompt-security-engineering-deep-dive.md>) | 보안 설정을 복붙하는 것과, 공격 원리를 알고 방어를 설계하는 것 |

</details>

<details>
<summary><b>📈 Performance & Quality (1)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Performance Testing](<blueprints/dev/performance & quality/prompt-performance-testing-deep-dive.md>) | 병목 지점을 수치로 특정하고 개선을 증명하기 |

</details>

<details>
<summary><b>🧬 Synthesis — 횡단 비교 (6)</b></summary>

> 개별 기술 레포가 아니라, 여러 기술이 *같은 문제에 다르게 답한 방식*을 나란히 놓는 레포. 위 레포들을 입력으로 삼는다.

| 블루프린트 | 컨셉 |
|------------|------|
| [Concurrency Models Compared](<blueprints/dev/synthesis/prompt-concurrency-models-compared.md>) | 모든 모델이 '블로킹 회피 + 상태 보호'에 다르게 답한 방식 |
| [Memory Management Compared](<blueprints/dev/synthesis/prompt-memory-management-compared.md>) | '언제 회수하나 + 누가 비용을 치르나'에 다르게 답한 방식 |
| [Caching & Memory Hierarchy Compared](<blueprints/dev/synthesis/prompt-caching-memory-hierarchy-compared.md>) | 모든 캐시가 '지역성·축출·쓰기·무효화'에 다르게 답한 방식 |
| [Compilation Strategies Compared](<blueprints/dev/synthesis/prompt-compilation-strategies-compared.md>) | '언제·얼마나 최적화하나'에 다르게 답한 방식 |
| [Rendering Pipelines Compared](<blueprints/dev/synthesis/prompt-rendering-pipelines-compared.md>) | '선언→레이아웃→페인트→합성→GPU'의 다른 구현들 |
| [Reactivity & State Compared](<blueprints/dev/synthesis/prompt-reactivity-state-compared.md>) | '의존성 추적 + 일관된 스냅샷'이 UI와 DB에서 같은 구조임 |

</details>

---

## ⚛️ Physis 카탈로그 — 자연 (36개)

물리학을 **L0(사고의 도구) → L6(관통 원리)** 로 쌓는 수직 커리큘럼. 변분원리·대칭에서 출발해 양자·상대성·양자중력을 지나 "창발"로 종합합니다. (`…-distilled` 본질 증류형 · L6의 `…-everywhere`는 횡단 종합)

<details>
<summary><b>L0 — 사고의 도구 (5)</b></summary>

> 물리를 푸는 게 아니라, 물리를 보는 눈. 모든 위층이 이 다섯을 다시 쓴다.

| 블루프린트 | 컨셉 |
|------------|------|
| [Variational Principles](blueprints/physis/L0/prompt-variational-principles-distilled.md) | 자연은 작용을 극값으로 — 모든 레이어 운동방정식의 씨앗 |
| [Symmetry & Conservation](blueprints/physis/L0/prompt-symmetry-conservation-distilled.md) | 대칭이 보존을 낳고(뇌터), 국소화하면 힘이 나온다(게이지) |
| [Dimensional Analysis & Scaling](blueprints/physis/L0/prompt-dimensional-scaling-distilled.md) | 차원분석은 답의 절반을 공짜로 — 풀기 전에 형태를 본다 |
| [Approximation & Perturbation](blueprints/physis/L0/prompt-approximation-perturbation-distilled.md) | 물리는 근사의 예술 — 근사가 깨지는 곳에 새 물리가 있다 |
| [Measurement & Uncertainty](blueprints/physis/L0/prompt-measurement-uncertainty-distilled.md) | 관측 가능한 것만 물리 — 측정에 살아남은 것만 신뢰 |

</details>

<details>
<summary><b>L1 — 고전 물리 (5)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Classical Mechanics](blueprints/physis/L1/prompt-classical-mechanics-distilled.md) | 같은 운동, 세 시선 — Newton·Lagrange·Hamilton |
| [Electromagnetism](blueprints/physis/L1/prompt-electromagnetism-distilled.md) | 네 방정식이 빛을 예언 — 게이지 대칭이 전자기를 지정 |
| [Oscillations & Waves](blueprints/physis/L1/prompt-oscillations-waves-distilled.md) | 평형 근처에선 모든 게 진동 — 정규모드·공명·푸리에 |
| [Classical Field Theory](blueprints/physis/L1/prompt-classical-field-theory-distilled.md) | 입자에서 장으로 — 장을 양자화하면 입자가 나온다 |
| [Nonlinear Dynamics & Chaos](blueprints/physis/L1/prompt-chaos-distilled.md) | 결정론인데 예측 불가, 그러나 카오스 ≠ 무작위 |

</details>

<details>
<summary><b>L2 — 열·통계·시간 (5)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Thermodynamics](blueprints/physis/L2/prompt-thermodynamics-distilled.md) | 온도는 분포의 성질, 엔트로피는 시간의 화살 |
| [Statistical Mechanics](blueprints/physis/L2/prompt-statistical-mechanics-distilled.md) | 미시에서 거시로 — 엔트로피는 무지의 척도 |
| [Phase Transitions & Criticality](blueprints/physis/L2/prompt-phase-transitions-distilled.md) | 물이 끓는 순간 = 창발의 정수 — 보편성·재규격화군 |
| [Arrow of Time](blueprints/physis/L2/prompt-arrow-of-time-distilled.md) | 시간의 화살은 법칙이 아니라 초기조건이다 |
| [Fluctuations & Information](blueprints/physis/L2/prompt-fluctuations-information-distilled.md) | 요동과 소산은 하나, 정보는 물리다 (맥스웰 도깨비) |

</details>

<details>
<summary><b>L3 — 양자 (6)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Quantum Mechanics](blueprints/physis/L3/prompt-quantum-mechanics-distilled.md) | 왜 세계는 이산적인가 — 중첩·불확정성·측정 |
| [Quantum Field Theory](blueprints/physis/L3/prompt-quantum-field-theory-distilled.md) | 입자는 장의 들뜸 — 경로적분·재규격화, 진공은 비어있지 않다 |
| [Standard Model](blueprints/physis/L3/prompt-standard-model-distilled.md) | 대칭이 힘을 낳는다 — 힉스=깨진 대칭, 세 힘의 통일과 빈틈 |
| [Entanglement & Measurement](blueprints/physis/L3/prompt-entanglement-measurement-distilled.md) | 벨 부등식 — 자연은 국소적이지 않다, 결잃음이 고전성을 창발 |
| [Quantum Information](blueprints/physis/L3/prompt-quantum-information-distilled.md) | 얽힘을 자원으로 — 양자 알고리즘은 간섭을 쓴다 |
| [Emergent Quantum Matter](blueprints/physis/L3/prompt-emergent-quantum-matter-distilled.md) | 'More is Different' — 준입자·위상물질·창발 게이지장 |

</details>

<details>
<summary><b>L4 — 상대성 & 우주 (5)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Special Relativity](blueprints/physis/L4/prompt-special-relativity-distilled.md) | 빛의 속도는 시공간의 구조 상수 — 질량은 에너지다 |
| [General Relativity](blueprints/physis/L4/prompt-general-relativity-distilled.md) | 중력은 힘이 아니라 기하 — 질량이 시공간을 휜다 |
| [Astrophysics](blueprints/physis/L4/prompt-astrophysics-distilled.md) | 우리는 별의 먼지 — 별의 일생은 양자+열역학+중력 |
| [Black Holes](blueprints/physis/L4/prompt-black-holes-distilled.md) | 모든 레이어가 충돌 — 호킹 복사, 엔트로피는 면적, 정보 역설 |
| [Cosmology](blueprints/physis/L4/prompt-cosmology-distilled.md) | 우주는 어떻게 시작했나 — 빅뱅·암흑우주·인플레이션 |

</details>

<details>
<summary><b>L5 — 양자중력 최전선 (5)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Quantum Gravity](blueprints/physis/L5/prompt-quantum-gravity-distilled.md) | 두 기둥이 충돌하는 곳 — 답이 아니라 질문이다 |
| [String Theory](blueprints/physis/L5/prompt-string-theory-distilled.md) | 점에서 끈으로 — 중력이 자동으로, 쌍대성이 '근본'을 흔든다 |
| [Loop Quantum Gravity](blueprints/physis/L5/prompt-loop-quantum-gravity-distilled.md) | 시공간 자체를 양자화 — 배경 독립·이산 기하 |
| [Holographic Principle](blueprints/physis/L5/prompt-holographic-principle-distilled.md) | 정보가 면적에 산다 — 얽힘이 시공간을 만든다 |
| [Emergent Spacetime & Multiverse](blueprints/physis/L5/prompt-emergent-spacetime-multiverse-distilled.md) | 시공간은 창발하는가, 우주는 여럿인가 — 이것은 과학인가 |

</details>

<details>
<summary><b>L6 — 관통 원리 (5 · everywhere)</b></summary>

> 아래 레이어 전체를 하나의 원리로 회수하는 횡단 종합.

| 블루프린트 | 컨셉 |
|------------|------|
| [Least Action Everywhere](blueprints/physis/L6/prompt-least-action-everywhere.md) | 하나의 변분 원리가 고전·장·상대성·양자·끈을 관통 |
| [Symmetry Everywhere](blueprints/physis/L6/prompt-symmetry-everywhere.md) | 법칙의 대칭이 보존·힘·질량·분류·쌍대성을 잇는다 |
| [Entropy & Time Everywhere](blueprints/physis/L6/prompt-entropy-time-everywhere.md) | 통계·우주·블랙홀·창발을 잇는 하나의 화살 |
| [Information Everywhere](blueprints/physis/L6/prompt-information-everywhere.md) | 정보는 물리다 (it-from-bit) |
| [Emergence](blueprints/physis/L6/prompt-emergence-distilled.md) | More is Different — 창발의 사다리, 연구소 전체의 마무리 |

</details>

---

## 🧠 Psyche 카탈로그 — 마음 (31개)

뉴런이라는 **3인칭의 단단한 바닥**에서 출발해 인지·기계·의식을 거쳐 자아·자유의지·"의식의 어려운 문제"까지 올라가는 마음 과학 7레이어.

<details>
<summary><b>L0 — 설명의 틀 (5)</b></summary>

> 마음을 연구하기 전에, 무엇을 어떤 층위에서 설명할지부터.

| 블루프린트 | 컨셉 |
|------------|------|
| [Mind–Body Map](blueprints/psyche/L0/prompt-mind-body-map-distilled.md) | 마음-몸 문제의 지형도 — 어디서 설명적 간극에 부딪히나 |
| [Levels of Explanation](blueprints/psyche/L0/prompt-levels-of-explanation-distilled.md) | Marr의 세 층위로 마음 과학의 설명 구조를 해부 |
| [Computation & Representation](blueprints/psyche/L0/prompt-computation-representation-distilled.md) | 물질이 '무언가에 관한 것'이 되나 — 지향성·표상·계산 |
| [Information & Prediction](blueprints/psyche/L0/prompt-information-prediction-distilled.md) | 뇌는 예측 기계 — 정보이론·베이즈로 마음의 알고리즘 |
| [Methods for the First Person](blueprints/psyche/L0/prompt-first-person-methods-distilled.md) | 경험을 측정하는 방법 — 정신물리·내성·현상학·NCC 비교 |

</details>

<details>
<summary><b>L1 — 뇌의 하드웨어 (4)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Neurons & Neural Codes](blueprints/psyche/L1/prompt-neurons-neural-codes-distilled.md) | 뉴런은 무엇을 계산하나 — 활동전위에서 집단 부호화 |
| [Neuromodulation & Arousal](blueprints/psyche/L1/prompt-neuromodulation-arousal-distilled.md) | 도파민은 행복이 아니다 — 계산 상태 전환, 각성 스위치 |
| [Plasticity & Learning](blueprints/psyche/L1/prompt-plasticity-learning-distilled.md) | 경험이 물질에 새겨지는 법 — 헵·STDP·엔그램 |
| [Brain Architecture](blueprints/psyche/L1/prompt-brain-architecture-distilled.md) | 왜 뇌는 이렇게 조직됐나 — 피질·시상·기저핵·소뇌의 분업 |

</details>

<details>
<summary><b>L2 — 인지 (6)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Perception](blueprints/psyche/L2/prompt-perception-distilled.md) | 본다는 것은 추론 — 예측처리·베이즈 지각 |
| [Attention & Working Memory](blueprints/psyche/L2/prompt-attention-working-memory-distilled.md) | 한 번에 조금밖에 못 잡는 계산적 이유 |
| [Memory](blueprints/psyche/L2/prompt-memory-distilled.md) | 기억은 재생이 아니라 재구성 — 거짓기억의 불가피성 |
| [Learning & Decision](blueprints/psyche/L2/prompt-learning-decision-distilled.md) | 마음은 어떻게 가치를 계산하나 — 강화학습의 심리학 |
| [Emotion & Motivation](blueprints/psyche/L2/prompt-emotion-motivation-distilled.md) | 감정은 합리의 적이 아니라 평가 시스템 |
| [Language & Concepts](blueprints/psyche/L2/prompt-language-concepts-distilled.md) | 언어는 사고를 가두나 해방하나 — 합성성·언어 상대성 |

</details>

<details>
<summary><b>L3 — 마음과 기계 (3)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Computational Theory of Mind](blueprints/psyche/L3/prompt-computational-mind-distilled.md) | 마음은 소프트웨어인가 — 기능주의·중국어 방 |
| [Embodied & Enactive Mind](blueprints/psyche/L3/prompt-embodied-mind-distilled.md) | 마음은 뇌 안에만 있지 않다 — 4E 인지 |
| [Brains vs Artificial Networks](blueprints/psyche/L3/prompt-brains-vs-networks-distilled.md) | 인공 신경망은 뇌를 얼마나 닮았나 |

</details>

<details>
<summary><b>L4 — 의식 (4)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [The Hard Problem](blueprints/psyche/L4/prompt-hard-problem-distilled.md) | 왜 정보 처리에 느낌이 따라붙나 — 좀비 논변·설명적 간극 |
| [Neural Correlates of Consciousness](blueprints/psyche/L4/prompt-ncc-distilled.md) | 의식을 가르는 신경 서명 — 상관에서 원인으로의 거리 |
| [Theories of Consciousness](blueprints/psyche/L4/prompt-theories-of-consciousness-distilled.md) | GWT·IIT·고차·예측처리 — 정직한 비교와 반증 예측 |
| [Altered & Edge States](blueprints/psyche/L4/prompt-altered-states-distilled.md) | 의식이 흐트러질 때 구조가 드러난다 — 수면·마취·환각제·명상 |

</details>

<details>
<summary><b>L5 — 자아·자유·타인 (4)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [The Self & Self-Model](blueprints/psyche/L5/prompt-self-model-distilled.md) | '나'는 실체인가 과정인가 — 자기모델·신체적/서사적 자아 |
| [Free Will & Agency](blueprints/psyche/L5/prompt-free-will-agency-distilled.md) | 리벳 실험은 자유의지를 부정하나 — 양립가능론·행위주체감 |
| [Other Minds & Social Cognition](blueprints/psyche/L5/prompt-other-minds-distilled.md) | 타인의 마음을 어떻게 아나 — 마음이론·거울뉴런 |
| [Meaning, Mortality & the Examined Life](blueprints/psyche/L5/prompt-meaning-mortality-distilled.md) | 의미는 어디서 오고 유한한 자아는 어떻게 견디나 |

</details>

<details>
<summary><b>L6 — 관통 원리 (5 · everywhere)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Prediction Everywhere](blueprints/psyche/L6/prompt-prediction-everywhere.md) | 하나의 추론 원리가 마음 전체를 관통 — RPE·자유에너지·능동추론 |
| [Representation Everywhere](blueprints/psyche/L6/prompt-representation-everywhere.md) | 물질이 '무언가에 관한 것'이 되는 동형성 |
| [Binding & Integration Everywhere](blueprints/psyche/L6/prompt-binding-integration-everywhere.md) | 분산된 처리가 하나의 경험이 되는 문제 |
| [The Strange Loop](blueprints/psyche/L6/prompt-strange-loop-everywhere.md) | 자신을 가리키는 고리 — 메타인지·자기참조 횡단 |
| [Emergence — Mind from No-Mind](blueprints/psyche/L6/prompt-mind-emergence-distilled.md) | 물질에서 마음이 나오는데 물질-언어로 환원 안 되는 이유 |

</details>

---

## 🧭 Phronesis 카탈로그 — 실천지 (31개)

세상이 실제로 돌아가는 법 — 의사결정·돈·설득·전략·역사를 **L0(판단의 토대) → L5(관통 원리)** 로. 모든 블루프린트가 "아는 것"과 "실제로 행동을 바꾸는 것"의 간극을 겨냥합니다.

<details>
<summary><b>L0 — 사고·판단의 토대 (5)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Mental Models](blueprints/phronesis/L0/prompt-mental-models-distilled.md) | 어떤 문제에 어떤 모델, 어디서 모델이 깨지나 |
| [Decision Theory](blueprints/phronesis/L0/prompt-decision-theory-distilled.md) | 결과가 아니라 결정 시점의 정보로 결정의 질을 평가 |
| [Probabilistic Thinking](blueprints/phronesis/L0/prompt-probabilistic-thinking-distilled.md) | 다음 결정의 성공 확률·기댓값·베팅 크기를 숫자로 |
| [Statistics Literacy](blueprints/phronesis/L0/prompt-statistics-literacy-distilled.md) | '분모가 뭐지? 누가 빠졌지? 우연일 확률은?'을 반사적으로 |
| [Cognitive Bias](blueprints/phronesis/L0/prompt-cognitive-bias-distilled.md) | 편향 지식이 행동을 못 바꾸는 이유 — 시스템으로 방어 |

</details>

<details>
<summary><b>L1 — 돈·금융 (7)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Money & Interest](blueprints/phronesis/L1/prompt-money-and-interest-distilled.md) | 금리가 모든 가치평가의 분모가 되는 구조 |
| [Personal Finance](blueprints/phronesis/L1/prompt-personal-finance-distilled.md) | 저축률×시간×복리가 부의 방정식 대부분을 결정 |
| [Financial Statements](blueprints/phronesis/L1/prompt-financial-statements-distilled.md) | 이익은 의견, 현금은 사실 — 세 표를 교차 심문 |
| [Valuation](blueprints/phronesis/L1/prompt-valuation-distilled.md) | 가치 = 미래 현금흐름의 할인, 가정 1%p가 결론을 뒤흔든다 |
| [Unit Economics](blueprints/phronesis/L1/prompt-unit-economics-distilled.md) | 고객 한 단위의 경제가 양수이고 규모에서 유지되는가 |
| [Markets & Investing](blueprints/phronesis/L1/prompt-markets-investing-distilled.md) | 알파는 제로섬·베타는 포지티브섬, 설 자리를 고른다 |
| [Macro & Cycles](blueprints/phronesis/L1/prompt-macro-cycles-distilled.md) | 예측은 포기, 사이클의 위치를 추정해 어디서도 죽지 않게 |

</details>

<details>
<summary><b>L2 — 설득·성장·시장 (6)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Persuasion Psychology](blueprints/phronesis/L2/prompt-persuasion-psychology-distilled.md) | 각 기법이 어떤 인지적 지름길에 올라타나 — 만들고 막기 |
| [Copy & Narrative](blueprints/phronesis/L2/prompt-copy-narrative-distilled.md) | 글의 목적은 행동 — 주목→이해→설득→행동 설계 |
| [Positioning & Brand](blueprints/phronesis/L2/prompt-positioning-brand-distilled.md) | 브랜드 = 기억 구조 — 멘탈 어베일러빌리티 설계 |
| [Pricing](blueprints/phronesis/L2/prompt-pricing-distilled.md) | 고객마다 다른 WTP 분포를 측정, 자기 선택으로 잉여 회수 |
| [Funnel & Growth](blueprints/phronesis/L2/prompt-funnel-growth-distilled.md) | 성장 방정식의 어느 항이 병목인지 진단·실험 |
| [Negotiation](blueprints/phronesis/L2/prompt-negotiation-distilled.md) | 협상력 = 대안의 함수, 준비가 8할, 과정으로 채점 |

</details>

<details>
<summary><b>L3 — 전략·게임·제도 (5)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Microeconomics](blueprints/phronesis/L3/prompt-microeconomics-distilled.md) | 수요·공급 곡선이 무엇의 집계인가, 시장이 실패하는 세 곳 |
| [Game Theory](blueprints/phronesis/L3/prompt-game-theory-distilled.md) | 균형이 최선이 아님을 알고 보수 구조를 바꾼다 |
| [Incentive Design](blueprints/phronesis/L3/prompt-incentive-design-distilled.md) | 보상이 정확히 무엇을 움직이나(대개 의도와 다른 것) |
| [Strategy & Moats](blueprints/phronesis/L3/prompt-strategy-moats-distilled.md) | 해자의 메커니즘·측정·침식 조건을 정직하게 감사 |
| [Law for Founders](blueprints/phronesis/L3/prompt-law-for-founders-distilled.md) | 법 = 게임의 규칙, 확률×손해의 기댓값으로 리스크 관리 |

</details>

<details>
<summary><b>L4 — 역사·세상 읽기 (4)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Tech Adoption](blueprints/phronesis/L4/prompt-tech-adoption-distilled.md) | 성능이 아니라 확산의 구조(곡선·보완재·생태계·타이밍) |
| [Corporate Rise & Fall](blueprints/phronesis/L4/prompt-corporate-rise-fall-distilled.md) | 생존편향·후광효과를 방어하며 승자·패자를 쌍으로 해부 |
| [Economic History](blueprints/phronesis/L4/prompt-economic-history-distilled.md) | 역사를 이론의 검증 데이터로 — 후견지명의 함정 방어 |
| [Geopolitics](blueprints/phronesis/L4/prompt-geopolitics-distilled.md) | 국가 행동을 게임 구조로, 노출을 측정해 예측 없이 대비 |

</details>

<details>
<summary><b>L5 — 관통 원리 (4 · everywhere)</b></summary>

| 블루프린트 | 컨셉 |
|------------|------|
| [Compounding Everywhere](blueprints/phronesis/L5/prompt-compounding-everywhere.md) | 재투자 구조가 있는 모든 곳의 같은 지수 과정 |
| [Feedback Loops Everywhere](blueprints/phronesis/L5/prompt-feedback-loops-everywhere.md) | 뱅크런·바이럴·플라이휠이 같은 두 루프의 변주 |
| [Incentives Everywhere](blueprints/phronesis/L5/prompt-incentives-everywhere.md) | 처음 보는 시스템의 행동을 보수 구조에서 역산 |
| [Leverage Everywhere](blueprints/phronesis/L5/prompt-leverage-everywhere.md) | 자본·코드·미디어·신뢰가 전부 배율 자산, 양면을 관리 |

</details>

---

## 🧩 블루프린트 해부 (Anatomy)

모든 블루프린트는 공통 골격을 공유합니다. 자세한 규약은 [`core-style-guide.md`](core-style-guide.md)에 정의되어 있습니다.

```
# {주제} Deep Dive 레포지토리 제작 프롬프트
  나는 "{주제} Deep Dive" 레포지토리를 만들려고 해. … (X를 하는 것과 Y를 아는 것은 다르다)

## 📋 프로젝트 목표      — 컨셉 · 핵심 차별화 · 타겟 독자 · 선행 학습/레포 연결
## 🎯 1단계: 전체 구조    — (확정형) 고정 목록  /  (설계형) 먼저 설계
## 📐 문서 구조           — 모든 문서가 따르는 10섹션 표준
## 🎨 스타일 가이드       — "모든 주장은 도구 출력/코드/증명으로"
## 🛠️ 실험·검증 환경      — javap·JMH·EXPLAIN·docker-compose·requirements.txt …
## 🎯 2단계: 작업 순서    — 디렉토리 → README → 섹션별 문서
## 📚 참고 자료           — 1차 출처(스펙·논문·소스코드)
## 💡 핵심 분석 대상       — 이 레포의 하이라이트 증명/분해
## 🎯 진행 방식           — 에이전트에게 주는 시작 신호
```

생성되는 **각 문서**는 두 표준 구조 중 하나를 따릅니다.

- **표준 v2 (시스템/Dev):** 핵심 질문 → 왜 필요한가 → 흔한 오해(Before) → 올바른 이해(After) → 내부 동작 원리 → 실험으로 확인 → 측정 결과 → 트레이드오프 → 핵심 정리 → 생각해볼 문제
- **수학 변형 (AI/이론):** 핵심 질문 → 왜 핵심인가 → 수학적 선행 조건 → 직관적 이해 → 엄밀한 정의 → 정리와 증명 → 구현 검증 → 실전 활용 → 가정과 한계 → 핵심 정리 → 생각해볼 문제

---

## 🤝 새 블루프린트 추가하기

1. 알맞은 폴더에 갈래 규칙에 맞는 이름으로 생성한다 — AI·Dev는 `prompt-{주제}-deep-dive.md`, Physis·Psyche·Phronesis는 `prompt-{주제}-distilled.md`, 상위 레이어 횡단 종합은 `prompt-{주제}-everywhere.md`. (대소문자·하이픈 컨벤션 준수)
2. [`core-style-guide.md`](core-style-guide.md)의 골격과 두 표준 문서 구조 중 하나를 채택한다.
3. 첫 문장은 반드시 **"…하는 것과 …아는 것은 다르다"** 패턴으로 *사용 vs 본질*의 간극을 못 박는다.
4. 이 README의 해당 카테고리 표에 한 줄(`블루프린트 | 컨셉`)을 추가한다.

> 이 저장소는 **IQ Lab**(`iq-lab`) 학습 생태계의 일부입니다 — 블루프린트가 레포의 *설계도*라면, 그 레포들은 *건물*입니다.
