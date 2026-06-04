# GPU & Graphics Pipeline Deep Dive 레포지토리 제작 프롬프트

나는 "GPU & Graphics Pipeline Deep Dive" 레포지토리를 만들려고 해.
정점(vertex)이 화면의 픽셀이 되기까지 — GPU가 CPU와 왜 다르고, 래스터화·셰이더·합성이 어떻게 동작하며, 왜 프레임이 끊기는지를 완전히 파헤치는 레포야.
"프레임워크로 화면을 그리는 것"과 "GPU가 픽셀을 어떻게 만들고 60fps 예산이 어디서 깨지는지 알고 설계하는 것"의 차이를 만드는 레포다. Browser·Jetpack Compose·SwiftUI·Flutter의 렌더링이 **전부 이 파이프라인 위에 있다**는 걸 보여주는 공통 바닥이다.

## 📋 프로젝트 목표

**컨셉**: "UI 프레임워크로 화면을 그리는 것과, GPU가 정점을 픽셀로 바꾸는 과정을 알고 성능을 설계하는 것은 다르다"

**핵심 차별화**:
1. GPU는 CPU가 아니다 — 수천 코어의 SIMT, 처리량(throughput) 중심, 메모리 대역폭이 지배하는 구조
2. 파이프라인은 고정된 흐름이다 — vertex→rasterize→fragment→merge, 각 단계가 무엇을 하고 어디가 병목인가
3. 셰이더가 픽셀을 결정한다 — GLSL/WGSL이 GPU 위에서 병렬 실행되는 원리
4. 프레임 예산은 16.6ms — draw call·overdraw·대역폭이 60fps를 깨뜨리는 지점을 프로파일러로 증명

**타겟 독자**:
- React/Compose/SwiftUI로 UI를 그리지만 "왜 스크롤이 끊기는지" GPU 관점에서 설명 못하는 개발자
- WebGL/WebGPU를 들어봤지만 셰이더가 실제로 뭘 하는지 모르는 개발자
- "GPU 가속"을 단어로만 알고 무엇이 GPU로 가는지 모르는 프론트/모바일 개발자
- Browser Rendering·Compose·SwiftUI를 각각 봤지만 공통 GPU 바닥이 안 보이는 개발자
- 게임/시각화가 아니어도 렌더링 성능을 다뤄야 하는 앱 개발자

## 🔗 레포 연결

**⬆️ 선행**: `computer-architecture-deep-dive`(메모리 대역폭·SIMD·캐시가 GPU 이해의 토대).
**🤝 시너지**: `browser-rendering-deep-dive`(Composite 단계가 GPU로), `jetpack-compose-internals-deep-dive`/`swiftui-internals-deep-dive`(Skia/Core Animation의 GPU 활용), `flutter-deep-dive`(Impeller).
**🧬 수렴**: `rendering-pipelines-compared`(이 레포가 그 비교의 *공통 언어*를 제공 — Browser·Compose·SwiftUI·Flutter가 같은 GPU 파이프라인을 어떻게 쓰는지).

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 설계해줘:

### Chapter 1: GPU 아키텍처 — CPU와 무엇이 다른가 (5개 문서)
- GPU가 존재하는 이유 — 수많은 픽셀에 같은 연산을 동시에, "처리량 vs 지연시간"의 근본적 분업
- SIMT 실행 모델 — 워프/웨이브프런트, 수천 스레드가 같은 명령을 실행, CPU SIMD와의 차이
- 분기 발산(Divergence) — `if`가 GPU에서 비싼 이유, 워프 내 스레드가 갈라질 때의 직렬화
- 메모리 계층과 대역폭 — VRAM·공유 메모리·레지스터, GPU는 왜 memory-bound가 되기 쉬운가
- CPU↔GPU 경계 — PCIe 전송 비용, 명령 제출(command buffer), 동기화가 만드는 지연

### Chapter 2: 그래픽스 파이프라인 완전 분해 (6개 문서)
- 파이프라인 전체 조망 — 정점 입력→정점 셰이더→프리미티브 조립→래스터화→프래그먼트 셰이더→출력 병합
- 정점 셰이더 — 좌표 변환(모델→뷰→투영), 동차좌표, 클립 공간, 정점당 병렬 실행
- 프리미티브 조립과 클리핑 — 삼각형 구성, 화면 밖 잘라내기, 후면 제거(back-face culling)
- 래스터화 — 삼각형을 픽셀(프래그먼트)로, 보간(interpolation), 누가 어떤 픽셀을 덮는가
- 프래그먼트 셰이더 — 픽셀당 색 계산, 텍스처 샘플링, 픽셀 수만큼 병렬 실행(가장 무거운 단계)
- 출력 병합 — 깊이 테스트(Z-buffer), 블렌딩(투명도), 스텐실, 프레임버퍼 기록

### Chapter 3: 셰이더 프로그래밍 (5개 문서)
- 셰이더 언어 — GLSL/WGSL 기본, GPU에서 컴파일되는 작은 프로그램, uniform/attribute/varying
- 좌표계와 변환 — MVP 행렬, NDC, 뷰포트 변환, 왜 행렬 곱으로 변환하나
- 텍스처 샘플링 — UV 좌표, 필터링(bilinear/mipmap), 텍스처가 대역폭을 먹는 이유
- 라이팅 기초 — 디퓨즈/스페큘러, 프래그먼트 단위 계산, "픽셀마다 도는 코드"의 비용 감각
- 셰이더 디버깅 — 값 시각화, Spector.js로 셰이더·드로우콜 캡처, 단계별 출력 확인

### Chapter 4: 버퍼·텍스처·리소스 관리 (5개 문서)
- 정점/인덱스 버퍼 — 정점 데이터 GPU 업로드, 인덱스로 정점 재사용, 메모리 레이아웃
- 유니폼/스토리지 버퍼 — UBO/SSBO, CPU→GPU 데이터 전달, 업데이트 빈도와 비용
- 텍스처 메모리 — 포맷·압축(ETC/ASTC), mipmap, 텍스처 아틀라스로 대역폭·드로우콜 절감
- 리소스 바인딩 — 바인드 그룹/디스크립터, 상태 전환 비용, 바인딩을 줄이는 설계
- 프레임버퍼와 오프스크린 — 렌더 타깃, 오프스크린 렌더링, 포스트 프로세싱 패스

### Chapter 5: WebGL → WebGPU (5개 문서)
- WebGL 모델 — OpenGL ES 기반, 전역 상태 머신, 드로우콜마다 상태 설정의 오버헤드
- WebGPU의 등장 — 명시적 파이프라인·바인드 그룹, 커맨드 인코더, Vulkan/Metal/D3D12를 추상화
- 명령 제출 모델 — 커맨드 버퍼를 미리 구성→일괄 제출, CPU 오버헤드를 줄이는 원리
- 컴퓨트 셰이더 — 그래픽 외 범용 GPU 연산(GPGPU), 워크그룹, 병렬 데이터 처리
- 네이티브 대조 — WebGPU↔Metal/Vulkan 매핑, 모바일(Compose/SwiftUI)이 쓰는 백엔드와의 관계

### Chapter 6: 렌더링 성능 (6개 문서)
- 프레임 예산 — 60fps = 16.6ms/120fps = 8.3ms, CPU 시간 + GPU 시간 분리해서 보기
- 드로우콜 비용 — 드로우콜이 비싼 이유, 배칭/인스턴싱으로 줄이기, CPU-bound vs GPU-bound 판별
- Overdraw — 같은 픽셀을 여러 번 칠하는 낭비, 투명도·레이어 누적, 측정·시각화
- 대역폭 병목 — 텍스처·프레임버퍼 트래픽, 해상도가 비용에 미치는 영향, 모바일 타일 기반 GPU
- GPU 프로파일링 — chrome://tracing, WebGPU timestamp query, 프레임 캡처로 병목 단계 특정
- 끊김(Jank) 진단 — 프레임 드롭의 원인 분류(CPU 준비·GPU 처리·합성), 케이스 진단

### Chapter 7: UI 프레임워크와 GPU — 수렴 (6개 문서)
- 2D도 결국 GPU다 — UI 사각형/텍스트/이미지가 삼각형+텍스처로 그려지는 원리
- 합성(Compositing) — 레이어를 GPU에서 합치기, 브라우저 Composite 단계의 실체
- Skia / Impeller — 2D 렌더 엔진이 GPU 명령으로 번역하는 방식, Flutter의 셰이더 컴파일 jank 문제
- Core Animation / RenderThread — iOS Render Server·Android RenderThread가 GPU에 그리는 흐름
- 텍스트 렌더링 — 글리프 아틀라스, 서브픽셀, 텍스트가 의외로 비싼 이유
- 종합: 같은 파이프라인, 다른 포장 — Browser·Compose·SwiftUI·Flutter를 GPU 관점으로 재해석(→ rendering-pipelines-compared로 연결)

→ 각 챕터 5~6개 문서. **총 38개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 파이프라인 단계·셰이더 코드, `💻 실전 실험`은 WebGPU/WebGL 데모 + Spector.js 캡처 + chrome://tracing. `📊`는 드로우콜·overdraw·대역폭의 정량 비교.

## 🎨 스타일 가이드

1. **항상 픽셀까지 추적** — "그려진다"가 아니라 정점→래스터→프래그먼트의 어느 단계 일인지 명시
2. **브라우저에서 재현** — WebGPU/WebGL은 설치 없이 브라우저로 돌려보게, 셰이더 스니펫 제공
3. **프로파일러로 증명** — 끊김·overdraw를 chrome://tracing·프레임 캡처로 *눈으로* 보여준다
4. **UI 프레임워크로 착지** — 각 개념이 Compose/SwiftUI/Browser에서 어떻게 나타나는지 연결
5. 파이프라인·좌표 변환은 항상 그림으로

## 🔬 검증 환경

> docker-compose 불필요. **브라우저(WebGPU/WebGL) + Spector.js + Chrome 추적**이 검증 도구. 코드는 정적 HTML로 바로 실행.

```bash
# 환경: 최신 Chrome (WebGPU 지원). 빌드 도구 없이 .html 직접 열기 가능.

# 1) WebGPU 지원 확인
#    chrome://gpu  → "WebGPU: Hardware accelerated" 확인

# 2) 프레임/드로우콜 캡처 — Spector.js
#    브라우저 확장 설치 → 캔버스 프레임 캡처 → 드로우콜·셰이더·상태 단계별 확인

# 3) CPU/GPU 타임라인 — chrome://tracing (또는 DevTools Performance)
#    record → 'gpu', 'cc'(compositor), 'blink' 카테고리에서 프레임 단계 분석

# 4) WebGPU 정밀 측정 — timestamp query
#    파이프라인에 timestamp-query 기능으로 GPU 패스 시간 직접 측정

# 5) overdraw 시각화 실험
#    반투명 사각형을 N겹 쌓아 프래그먼트 셰이더 호출 폭증을 프레임타임으로 관찰
```

핵심 실험 예: 같은 화면을 드로우콜 1000개 vs 인스턴싱 1개로 그려 CPU 시간 비교 / 반투명 레이어 겹침으로 overdraw가 프레임타임에 주는 영향 측정 / WebGL 상태 전환 vs WebGPU 커맨드 버퍼의 CPU 오버헤드 대조.

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `demos/`(WebGPU/WebGL 실행 예제 HTML)
2. **README.md**: 🪨 Foundations 톤, "프레임워크로 그린다 vs GPU가 픽셀을 만든다" 포지셔닝, Browser/Compose/SwiftUI/Flutter가 이 파이프라인 위에 있음을 명시, `🔗 레포 연결`
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어, demos에 재현 예제 누적

## 📚 참고 자료

- WebGPU Fundamentals — https://webgpufundamentals.org/
- WebGL Fundamentals — https://webglfundamentals.org/
- *Real-Time Rendering* (Akenine-Möller et al.)
- LearnOpenGL — https://learnopengl.com/
- The Book of Shaders — https://thebookofshaders.com/
- Chrome GPU 문서·Spector.js — https://spector.babylonjs.com/

## 💡 핵심 분석 대상

```
그래픽스 파이프라인 (정점 → 픽셀):

  정점 데이터 ──► [Vertex Shader] ──► 좌표 변환 (MVP)
                                          │
                                          ▼
                              [Primitive Assembly] 삼각형 구성
                                          │
                                          ▼ Rasterization
                              삼각형 → 프래그먼트(픽셀 후보)
                                          │
                                          ▼
                              [Fragment Shader] 픽셀당 색 (가장 무거움)
                                          │
                                          ▼ Output Merger
                              깊이 테스트 + 블렌딩 → 프레임버퍼

CPU vs GPU (처리량 vs 지연):
  CPU: 소수 코어, 빠른 단일 스레드, 분기·복잡 로직   → 지연시간 최적화
  GPU: 수천 코어, 같은 명령 동시 실행(SIMT)          → 처리량 최적화
  → "픽셀 200만 개에 같은 셰이더" = GPU의 영역

분기 발산:
  워프(32스레드)가 같은 명령 실행
  if(cond) {A} else {B} 에서 스레드마다 cond 다르면
    → A도 실행하고 B도 실행(양쪽 직렬화) → 비용 2배

프레임 예산 (60fps = 16.6ms):
  [ CPU: 씬 준비·드로우콜 제출 ] + [ GPU: 래스터·셰이더·합성 ]
  둘 중 하나라도 16.6ms 초과 → 프레임 드롭(jank)
  드로우콜 1000개 → CPU-bound / overdraw 심함 → GPU-bound

UI 프레임워크의 수렴 (전부 같은 바닥):
  Browser Composite ─┐
  Jetpack Compose ───┤
  SwiftUI ───────────┼──► Skia/Impeller/CoreAnimation ──► GPU 파이프라인
  Flutter ───────────┘                                    (위 그림)
  → "사각형·텍스트·이미지"도 결국 삼각형 + 텍스처 + 셰이더
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목(5~6개) + 핵심 내용(3~4줄) + 총 38개 확인 + WebGPU/Spector.js/tracing 검증 환경 + Browser/Compose/SwiftUI/Flutter 연결 지점 명시(특히 Chapter 7이 rendering-pipelines-compared로 어떻게 수렴하는지).

**준비됐으면 1단계 구조 설계부터 시작해줘!**
