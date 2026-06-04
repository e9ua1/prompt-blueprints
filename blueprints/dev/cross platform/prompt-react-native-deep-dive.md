# React Native Deep Dive 레포지토리 제작 프롬프트

나는 "React Native Deep Dive" 레포지토리를 만들려고 해.
JS로 작성한 UI가 어떻게 네이티브 컴포넌트가 되는지 — 구형 Bridge에서 New Architecture(JSI·Fabric·TurboModules)로의 전환과 Hermes 엔진을 완전히 파헤치는 레포야.
"React Native로 앱을 만드는 것"과 "JS와 네이티브가 어떻게 통신하고 왜 Bridge가 New Architecture로 바뀌었는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "RN으로 앱을 만드는 것과, JS와 네이티브가 어떤 경계로 통신하고 그 경계가 어떻게 진화했는지 아는 것은 다르다"

**핵심 차별화**:
1. 두 세계의 통신 — JS 스레드와 네이티브가 분리된 구조, 통신이 곧 병목
2. Bridge → New Architecture — 비동기 직렬화 Bridge의 한계, JSI 동기 호출로의 전환
3. Fabric 렌더러 — C++ Shadow Tree, React Internals(Fiber)가 네이티브 뷰가 되는 경로
4. Hermes — RN 전용 JS 엔진, 바이트코드 사전 컴파일, 시작·메모리 최적화

**타겟 독자**:
- RN을 쓰지만 JS-네이티브 통신을 모르는 개발자
- "Bridge가 느리다"·"New Architecture"를 단어로만 아는 개발자
- Fabric·TurboModules·JSI를 모르는 개발자
- 네이티브 모듈을 작성하지만 경계 비용을 모르는 개발자
- `react-internals`·`v8`·`android/ios` 레포를 RN으로 연결하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `react-internals-deep-dive`(Fiber·재조정), `javascript-deep-dive`(JS 의미), `android-framework-internals`/`uikit-core-animation`(네이티브 뷰).
**🤝 시너지**: `v8-engine-deep-dive`(Hermes vs V8), `flutter-deep-dive`(다른 접근 대조), `event-loop-async-deep-dive`(JS 스레드).
**🧬 수렴**: `rendering-pipelines-compared`(RN→네이티브 뷰 ↔ Flutter 자체 렌더 ↔ 웹).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: RN 아키텍처 개요 (5개 문서)
- RN이란 — "네이티브 뷰를 JS로 제어", 웹뷰가 아닌 진짜 네이티브 컴포넌트
- 스레드 모델 — JS 스레드·메인(UI) 스레드·셰도우 스레드의 분업
- 두 세계 — JS 세계와 네이티브 세계, 둘을 잇는 것이 핵심 과제
- 렌더 흐름 한눈에 — React 트리→네이티브 뷰 생성, 어디서 무엇이
- Old vs New Architecture — 무엇이 왜 바뀌었나 미리보기

### Chapter 2: 구형 Bridge (5개 문서)
- Bridge 구조 — JS와 네이티브 사이 비동기 메시지 큐
- 직렬화 통신 — 모든 호출이 JSON 직렬화→큐→역직렬화, 비동기
- Bridge의 한계 — 직렬화 비용·비동기 강제·배치 지연, 큰 데이터 병목
- 3스레드 흐름 — JS→Shadow(레이아웃)→Main(렌더)의 비동기 단계
- 왜 느린가 — 동기 측정 불가, 잦은 통신 시 병목(스크롤·제스처)

### Chapter 3: JSI — New Architecture의 기반 (6개 문서)
- JSI란 — JavaScript Interface, JS 엔진에 C++로 직접 접근하는 얇은 계층
- 동기 호출 — 직렬화 없이 JS↔C++ 직접 호출, Bridge 제거
- HostObject — JS에서 보이는 C++ 객체, 메서드 직접 노출
- 엔진 독립 — JSI가 특정 엔진(JSC/Hermes/V8)에 안 묶이는 설계
- 메모리·생명주기 — JSI 객체의 소유권, JS GC와 C++의 경계
- Bridge → JSI 비교 — 직렬화 비동기 vs 직접 동기의 차이

### Chapter 4: Fabric 렌더러 (6개 문서)
- Fabric 개요 — 새 렌더 시스템, C++ 코어, React와 네이티브 통합
- C++ Shadow Tree — 레이아웃 트리를 C++로, 크로스 플랫폼 공유
- React Fiber와 연결 — react-internals의 Fiber가 Shadow Tree로(react 연결)
- 렌더 파이프라인 — render→commit→mount, 동기 측정·레이아웃
- Yoga 레이아웃 — Flexbox 레이아웃 엔진(C++), 크로스 플랫폼 일관성
- 동시성·우선순위 — Fabric이 React 18 동시성과 결합

### Chapter 5: TurboModules와 네이티브 (5개 문서)
- TurboModules — JSI 기반 네이티브 모듈, 지연 로딩·동기 호출
- 구형 NativeModules와 비교 — Bridge 기반 vs JSI 기반
- Codegen — 타입 안전한 JS↔네이티브 인터페이스 자동 생성
- 네이티브 모듈 작성 — Android(Kotlin)·iOS(Swift) 모듈, 경계 설계
- 경계 비용 — 여전히 존재하는 비용, 무엇을 네이티브로 보낼까

### Chapter 6: Hermes 엔진 (5개 문서)
- Hermes란 — RN용 경량 JS 엔진, 모바일 최적화
- 바이트코드 사전 컴파일 — 빌드 타임에 바이트코드 생성, 파싱 시작 비용 제거
- V8과 비교 — JIT 전략·메모리·시작 시간 트레이드오프(v8 연결)
- 메모리·GC — 모바일 제약에 맞춘 GC
- 디버깅 — Hermes 디버거, 바이트코드 관찰

### Chapter 7: 성능과 실전 (4개 문서)
- 성능 측정 — Flipper·프로파일러, JS 스레드·네이티브 추적
- 흔한 병목 — 과한 Bridge/JSI 통신, 큰 리스트, 이미지
- New Architecture 마이그레이션 — 전환 고려, 호환
- 종합 — 같은 화면을 Old/New Architecture로 측정 비교

→ **총 38개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 Bridge/JSI/Fabric 흐름, `💻 실전 실험`은 Flipper·프로파일러·네이티브 모듈. `📊`는 Bridge vs JSI·Hermes vs JSC 비교.

## 🎨 스타일 가이드

1. **두 세계의 경계로 환원** — 모든 동작·비용을 JS↔네이티브 통신으로
2. **Old/New 대조** — Bridge(직렬화 비동기) vs JSI(직접 동기)를 항상
3. **react/v8/네이티브 레포로 착지** — Fiber→react, 엔진→v8, 뷰→android/ios
4. **Flutter와 대조 예고** — "네이티브 뷰 제어"(RN) vs "자체 렌더"(Flutter)
5. 스레드 모델·렌더 파이프라인은 다이어그램으로

## 🔬 검증 환경

> Node + RN CLI + 시뮬레이터/기기.

```bash
npx react-native@latest init RNLab   # New Architecture 기본
# Hermes 기본 활성

# 검증 방법
# 1) New Architecture on/off 비교(가능한 버전에서)
# 2) Flipper/React DevTools: JS 스레드·네이티브 뷰 계층·성능
# 3) Hermes 바이트코드:
#    hermesc -emit-binary -out app.hbc app.js   # 바이트코드 생성 관찰
# 4) 네이티브 모듈 작성: 간단한 TurboModule(Codegen) Android/iOS
# 5) 통신 비용: 큰 데이터를 JS↔네이티브로 반복 전달 → Bridge vs JSI 시간
# 6) Yoga 레이아웃: Flexbox 동작이 웹/네이티브 일관되는지 확인
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🔀 Cross-Platform 톤, "RN으로 만든다 vs JS↔네이티브가 어떻게 통신하나" 포지셔닝, `🔗 레포 연결`(react-internals·v8·flutter)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- RN New Architecture 문서 — https://reactnative.dev/architecture/landing-page
- "RN Architecture overview" — JSI·Fabric·TurboModules
- Hermes 문서 — https://hermesengine.dev/
- Yoga 레이아웃 — https://www.yogalayout.dev/
- RN 소스 — https://github.com/facebook/react-native

## 💡 핵심 분석 대상

```
두 세계 + 경계:
  [ JS 스레드 ]              [ 네이티브 (Main/UI) ]
   React 트리                 UIView / android.View
       │                          │
       └──── 경계(Bridge/JSI) ────┘

Old Bridge (직렬화 비동기):
  JS: createView(...) → JSON 직렬화 → 큐 → 네이티브 역직렬화 → 뷰 생성
  - 모든 통신 비동기·직렬화 → 동기 측정 불가, 잦으면 병목

New Architecture (JSI 직접):
  JS ──JSI(C++ 직접 호출)──► 네이티브 (직렬화 없음, 동기 가능)
  Fabric: React Fiber → C++ Shadow Tree → 네이티브 뷰 마운트
  TurboModules: 네이티브 모듈도 JSI로 직접·지연 로딩

Hermes:
  V8: 런타임에 JS 파싱→바이트코드→JIT (시작 비용)
  Hermes: 빌드 타임에 바이트코드(.hbc) 사전 생성
          → 앱 시작 시 파싱 없이 바로 실행(모바일 시작 ↓)

RN vs Flutter:
  RN     : JS가 "네이티브 뷰"를 제어(UIView/android.View)
  Flutter: 자체 엔진이 모든 픽셀을 직접 그림(네이티브 뷰 안 씀)
  → rendering-pipelines-compared
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 38개 확인 + Flipper/Hermes/네이티브모듈 검증 환경 + react-internals/v8/flutter 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
