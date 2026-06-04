# Web APIs & WASM Deep Dive 레포지토리 제작 프롬프트

나는 "Web APIs & WASM Deep Dive" 레포지토리를 만들려고 해.
브라우저가 JS에게 제공하는 능력 — DOM·이벤트·Worker·Storage·네트워크 API가 *엔진이 아니라 브라우저 쪽*에서 어떻게 구현되는지, 그리고 WebAssembly가 어떻게 거의 네이티브 속도로 실행되고 JS와 상호작용하는지를 완전히 파헤치는 레포야.
"Web API를 호출하는 것"과 "그 API가 어느 스레드·어느 프로세스에서 무슨 비용으로 동작하는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "DOM·fetch·Worker를 호출하는 것과, 그것들이 브라우저 내부에서 어떻게 구현되고 WASM과 어떻게 경계를 넘나드는지 아는 것은 다르다"

**핵심 차별화**:
1. Web API는 엔진이 아니다 — V8은 DOM을 모른다, API는 브라우저가 바인딩으로 노출
2. 진짜 멀티스레딩 — Web Worker로 메인 스레드를 벗어나는 법, 메시지 패싱·전송 가능 객체
3. WASM 실행 모델 — 선형 메모리·스택 머신·검증, 왜 안전하고 빠른가
4. JS ↔ WASM 경계 — 타입 변환·메모리 공유 비용, 언제 WASM이 이득인가

**타겟 독자**:
- DOM API를 쓰지만 그게 V8이 아니라 브라우저 바인딩이란 걸 모르는 개발자
- Web Worker를 들어봤지만 메인 스레드 오프로딩을 안 해본 개발자
- WASM을 "빠르다"고만 알고 실행 모델·JS 경계 비용을 모르는 개발자
- fetch·Streams를 쓰지만 백프레셔·스트리밍 처리를 모르는 개발자
- `event-loop` 레포를 봤고 그 위의 API 계층을 보고 싶은 개발자

## 🔗 레포 연결

**⬆️ 선행**: `v8-engine-deep-dive`(엔진/런타임 경계), `event-loop-async-deep-dive`(API 콜백이 루프로).
**🤝 시너지**: `browser-rendering-deep-dive`(DOM 변경→렌더), `compiler-deep-dive`(WASM은 컴파일 타깃), `rust-deep-dive`(Rust→WASM의 대표 소스).
**🧬 수렴**: `concurrency-models-compared`(Worker 메시지 패싱 모델), `rendering-pipelines-compared`(OffscreenCanvas).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: DOM 내부 표현 (5개 문서)
- DOM은 어디에 사는가 — V8이 아닌 브라우저(C++) 객체, JS 바인딩으로 노출
- 노드 트리 표현 — C++ 노드 ↔ JS 래퍼, 접근마다 경계를 넘는 비용
- DOM 조작 비용 — 대량 변경의 비용, DocumentFragment, 리플로 유발(browser-rendering 연결)
- 라이브 vs 스냅샷 — HTMLCollection(라이브) vs NodeList, querySelectorAll의 차이
- 가상 DOM의 동기 — 왜 라이브러리들이 DOM 접근을 줄이려 하나

### Chapter 2: 이벤트 시스템 (5개 문서)
- 이벤트 흐름 — 캡처→타깃→버블, 단계별 핸들러 실행
- 이벤트 위임 — 상위에서 한 번에 처리, 동적 요소 대응, 성능 이점
- 이벤트와 이벤트 루프 — 이벤트가 태스크로 큐잉되는 방식(event-loop 연결)
- passive 리스너 — 스크롤 성능, preventDefault와 합성 스크롤의 충돌
- 커스텀 이벤트·옵저버 — CustomEvent, IntersectionObserver/ResizeObserver/MutationObserver

### Chapter 3: Web Worker와 멀티스레딩 (5개 문서)
- Worker 기본 — 별도 스레드·별도 전역, 메인과 메모리 비공유
- 메시지 패싱 — postMessage, 구조화된 복제(structured clone)의 비용
- Transferable Objects — ArrayBuffer 소유권 이전(복사 없이), 언제 쓰나
- SharedArrayBuffer & Atomics — 진짜 공유 메모리, 락·동기화, Spectre 완화(COOP/COEP)
- Worker 활용 — 무거운 연산·파싱·이미지 처리 오프로딩, 메인 스레드 보호

### Chapter 4: Storage와 네트워크 API (5개 문서)
- 저장소 비교 — localStorage(동기·작음) vs IndexedDB(비동기·큼) vs Cache API
- IndexedDB 내부 — 트랜잭션·객체 저장소·인덱스, 비동기 모델의 이유
- fetch와 Request/Response — 스트림 기반 바디, 헤더, 취소(AbortController)
- Streams API — ReadableStream·백프레셔, 스트리밍 처리(다운로드 중 처리)
- Service Worker — 네트워크 프록시·캐시 전략·오프라인, 별도 생명주기

### Chapter 5: WebAssembly 실행 모델 (6개 문서)
- WASM이란 — 이식 가능한 바이트코드, 거의 네이티브 속도, JS의 대체가 아닌 보완
- 모듈 구조 — 함수·메모리·테이블·임포트/익스포트, 텍스트 형식(WAT)
- 선형 메모리 — 단일 연속 ArrayBuffer, 포인터 = 오프셋, 메모리 안전 경계
- 스택 머신과 검증 — 스택 기반 명령, 로드 시 타입 검증으로 안전 보장
- 컴파일·인스턴스화 — 스트리밍 컴파일, JIT로 기계어, 시작 비용
- 무엇을 잘하나 — CPU 집약 연산(코덱·암호·이미지), DOM 접근이 약한 이유

### Chapter 6: JS ↔ WASM 인터롭 (5개 문서)
- 경계 넘기 — 숫자는 싸고, 문자열·객체는 비싼 이유(선형 메모리 복사)
- 메모리 공유 — WASM 메모리를 JS에서 읽기/쓰기, 뷰(TypedArray)로 접근
- 도구 체인 — Rust(wasm-bindgen)·Emscripten·AssemblyScript, glue 코드의 역할
- 성능 측정 — 같은 작업을 JS vs WASM으로, 경계 비용 포함한 실제 이득 판별
- 함정 — 잦은 경계 왕복·작은 작업의 WASM은 손해, 적합 워크로드 판단

### Chapter 7: 고급 — 그래픽·미디어·미래 (5개 문서)
- OffscreenCanvas — Worker에서 캔버스 렌더링, 메인 스레드 분리(gpu 레포 연결)
- WebGPU 접점 — 컴퓨트로 GPGPU, WASM과의 조합
- 미디어 API — Web Audio·MediaStream의 실시간 처리 모델
- WASM의 진화 — GC 제안·스레드·SIMD·컴포넌트 모델, 서버사이드(WASI)
- 종합 — 이미지 필터를 JS·Worker·WASM·OffscreenCanvas로 구현·측정 비교

→ **총 36개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 브라우저 바인딩·WASM 메모리 모델, `💻 실전 실험`은 Worker/WASM 데모 + DevTools. `📊`는 JS vs WASM vs Worker의 정량 비교.

## 🎨 스타일 가이드

1. **"어디서 도는가"를 명시** — 메인 스레드·Worker·GPU·브라우저 프로세스 어디인지
2. **경계 비용 정량화** — 구조화 복제·WASM 경계 비용을 측정으로
3. **WAT로 까보기** — WASM 개념을 텍스트 형식으로 직접 읽기
4. **적합/부적합 판단** — "WASM 쓰면 빠르다" 신화를 측정으로 깨기
5. 메모리·스레드 모델은 다이어그램으로

## 🔬 검증 환경

> docker 불필요. **브라우저 + WASM 툴(wat2wasm/wasmtime)**이 검증 도구.

```dockerfile
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y nodejs npm
RUN npm i -g wat-wasm   # 또는 wabt(wat2wasm), binaryen(wasm-opt)
# Rust→WASM 실습 시: rustup + wasm-pack
```

```bash
# WAT → WASM 직접 변환·관찰
wat2wasm demo.wat -o demo.wasm
wasm2wat demo.wasm           # 역변환으로 구조 확인
wasm-objdump -x demo.wasm    # 섹션·임포트·익스포트

# Rust → WASM
wasm-pack build --target web

# 브라우저에서:
# - Worker 메시지 비용: structured clone vs Transferable 시간 측정
# - WASM vs JS: 같은 연산(예: 만델브로/이미지 필터)을 둘로 구현해 performance.now() 비교
# - DevTools Performance에서 Worker 스레드 타임라인 확인
# - SharedArrayBuffer는 COOP/COEP 헤더 필요 → 로컬 서버로 헤더 설정
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `demos/`(Worker·WASM 예제)
2. **README.md**: 🌐 Frontend 톤, "API를 호출한다 vs 어디서 무슨 비용으로 도나" 포지셔닝, `🔗 레포 연결`(v8·event-loop·rust)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- MDN Web API 레퍼런스 — https://developer.mozilla.org/en-US/docs/Web/API
- WebAssembly 공식 — https://webassembly.org/, MDN WASM 가이드
- "WebAssembly: A New Compilation Target for the Web" (논문)
- web.dev Workers·Streams — https://web.dev/
- Lin Clark "Cartoon intro to WebAssembly"

## 💡 핵심 분석 대상

```
Web API는 엔진 밖:
  JS 코드 ─► V8(엔진: 순수 JS만) ─X─ DOM 모름
                  │ 바인딩
                  ▼
            브라우저(C++): DOM·fetch·timer·Worker 구현
  → document.querySelector는 V8 기능이 아니라 브라우저가 노출

Worker 메시지 비용:
  postMessage(bigObject)        → 구조화 복제(전체 복사) 비쌈
  postMessage(buf, [buf])       → Transferable: 소유권만 이전(복사 0)

WASM 선형 메모리:
  [ ArrayBuffer 한 덩어리 ]  ← WASM의 전체 메모리
   ^ 포인터 = 이 버퍼의 오프셋(정수)
  JS는 이 버퍼를 TypedArray로 들여다봄
  문자열 전달? → JS 문자열을 버퍼에 인코딩(복사) → 경계 비용

JS vs WASM (경계 포함):
  큰 CPU 연산 1번        → WASM 이득 (경계 1회, 내부 빠름)
  작은 연산 수천 번 왕복  → JS 이득 (경계 비용이 이득을 잡아먹음)
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 36개 확인 + 브라우저/WASM툴 검증 환경 + v8/event-loop/rust 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
