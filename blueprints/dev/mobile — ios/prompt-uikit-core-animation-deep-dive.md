# UIKit & Core Animation Deep Dive 레포지토리 제작 프롬프트

나는 "UIKit & Core Animation Deep Dive" 레포지토리를 만들려고 해.
화면이 어떻게 그려지는지 — UIView와 CALayer의 분리, Layer Tree(Model/Presentation/Render), 별도 프로세스인 Render Server, 오프스크린 렌더링이 *내부에서 어떻게 동작*하는지를 완전히 파헤치는 레포야.
"UIView를 배치하는 것"과 "그것이 CALayer로 GPU에서 합성되고 왜 어떤 효과가 오프스크린을 유발하는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "UIView를 배치하는 것과, CALayer가 렌더 서버에서 GPU로 합성되는 파이프라인을 아는 것은 다르다"

**핵심 차별화**:
1. View ≠ Layer — UIView는 이벤트·레이아웃, CALayer가 실제 그리기·애니메이션을 담당
2. 세 가지 Layer Tree — Model/Presentation/Render의 분리, 애니메이션 중 값의 출처
3. Render Server — 그리기가 별도 프로세스(backboardd)에서 GPU로 합성되는 구조
4. 오프스크린 렌더링 — 마스크·그림자·코너가 별도 패스를 유발해 성능을 깎는 원리

**타겟 독자**:
- UIKit을 쓰지만 CALayer와의 관계를 모르는 개발자
- 애니메이션 중 위치 값을 잘못 읽는(model vs presentation) 개발자
- cornerRadius+shadow로 스크롤이 끊기는 이유(오프스크린)를 모르는 개발자
- 렌더링이 어디서(어느 프로세스) 일어나는지 모르는 개발자
- `gpu-graphics`·`swiftui-internals`와 연결해 렌더를 보려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `gpu-graphics-deep-dive`(래스터화·합성·GPU), `ios-lifecycle-runloop-deep-dive`(렌더 타이밍·트랜잭션).
**🤝 시너지**: `swiftui-internals-deep-dive`(SwiftUI도 결국 여기로), `objc-runtime-deep-dive`(UIKit은 objc), `android-performance-deep-dive`(RenderThread 대조).
**🧬 수렴**: `rendering-pipelines-compared`(Core Animation ↔ Browser Composite ↔ Compose ↔ Flutter).

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: UIView와 CALayer (6개 문서)
- 책임 분리 — UIView(이벤트·레이아웃·제스처) vs CALayer(그리기·애니메이션)
- View-Layer 관계 — 모든 UIView는 backing CALayer를 가짐, 위임 관계
- Layer 속성 — contents·bounds·position·transform, 실제 시각의 출처
- 레이어 종류 — CAShapeLayer·CAGradientLayer·CATextLayer 등 특수 레이어
- 좌표계 — bounds vs frame, anchor point, 변환과 좌표
- 직접 레이어 다루기 — View 없이 레이어로 그리기

### Chapter 2: 세 가지 Layer Tree (5개 문서)
- Model Layer Tree — 우리가 설정한 "목표" 값, 코드로 보는 값
- Presentation Layer Tree — 화면에 *지금 보이는* 값(애니메이션 중간값)
- Render Tree — 렌더 서버의 내부 표현(접근 불가)
- 애니메이션 중 값 — model은 최종값, presentation이 현재값(흔한 버그의 근원)
- 히트 테스트 — 애니메이션 중 터치 위치(presentation 기반)

### Chapter 3: 렌더 파이프라인과 Render Server (6개 문서)
- 렌더 흐름 전체 — 앱 프로세스(커밋)→Render Server(backboardd)→GPU→디스플레이
- 트랜잭션 — CATransaction, 암시적/명시적, RunLoop 끝 커밋(ios-lifecycle 연결)
- 커밋 단계 — layout→display→prepare→commit, 무엇이 언제
- Render Server — 별도 프로세스에서 합성, 앱이 멈춰도 일부 애니메이션 지속
- 합성 — 레이어를 GPU에서 합치기(gpu 연결), 텍스처·블렌딩
- vsync·프레임 — 60/120Hz 동기, 프레임 예산(performance 대조)

### Chapter 4: Core Animation (6개 문서)
- 암시적 애니메이션 — 레이어 속성 변경 시 자동 애니메이션, 액션 조회
- 명시적 애니메이션 — CABasicAnimation·CAKeyframeAnimation, 추가/제거
- 애니메이션 vs model 값 — 애니메이션이 presentation만 바꾸고 model은 그대로인 함정
- 타이밍·이징 — CAMediaTimingFunction, 애니메이션 곡선
- 트랜스폼·3D — CATransform3D, perspective, 레이어 변환
- UIView 애니메이션 — UIView animate가 Core Animation으로 번역되는 방식

### Chapter 5: 렌더 비용과 오프스크린 (6개 문서)
- 오프스크린 렌더링 — 별도 버퍼에 먼저 그린 뒤 합성, 추가 패스 비용
- 유발 요인 — 마스크·그림자(미최적화)·`cornerRadius`+clip·그룹 투명도
- 그림자 최적화 — shadowPath 지정으로 오프스크린 회피
- 래스터화 — shouldRasterize, 캐시 이점과 함정
- 블렌딩·오버드로 — 투명 레이어 누적 비용(gpu 연결)
- 픽셀 정렬 — 비정수 좌표의 안티앨리어싱 비용

### Chapter 6: 레이아웃과 드로잉 (5개 문서)
- Auto Layout 내부 — 제약→Cassowary 솔버→frame 계산, 비용
- 레이아웃 패스 — setNeedsLayout·layoutIfNeeded, 무효화·갱신
- drawRect vs 레이어 — Core Graphics 직접 그리기의 비용, 언제 피하나
- 이미지·디코딩 — 이미지 디코딩 비용, 백그라운드 디코딩
- 텍스트 렌더링 — Core Text·글리프, 텍스트가 비싼 이유(gpu 연결)

### Chapter 7: 측정과 실전 (4개 문서)
- Instruments — Core Animation·Time Profiler, FPS·오프스크린 측정
- 디버그 옵션 — Color Offscreen-Rendered·Color Blended Layers로 시각 진단
- 성능 케이스 — 끊기는 스크롤(오프스크린)→shadowPath→측정
- 종합 — UIView 화면을 Layer Tree·렌더 서버 관점으로 완전 해부

→ **총 38개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 Layer Tree·렌더 서버·트랜잭션, `💻 실전 실험`은 Instruments·디버그 색상·presentation layer 관찰. `📊`는 오프스크린·블렌딩의 프레임 비용.

## 🎨 스타일 가이드

1. **View와 Layer를 분리** — 모든 시각을 CALayer 동작으로 환원
2. **오프스크린을 시각화** — 디버그 색상으로 *직접 본다*
3. **model vs presentation 강조** — 애니메이션 버그를 두 트리로 설명
4. **gpu/swiftui와 연결** — 합성은 gpu, SwiftUI는 결국 여기로
5. 세 Layer Tree·렌더 파이프라인은 다이어그램으로

## 🔬 검증 환경

> macOS + Xcode + 실기기 권장(렌더 서버·GPU).

```bash
# 환경: macOS, Xcode, 실기기(시뮬레이터는 GPU 다름)

# 1) 디버그 색상 (Debug > View Debugging 또는 시뮬레이터 메뉴)
#    - Color Offscreen-Rendered Yellow: 오프스크린 영역 노랑
#    - Color Blended Layers: 블렌딩(투명 겹침) 빨강
#    → cornerRadius+masksToBounds, shadow 켜고 노랑 확인

# 2) presentation vs model 실험:
#    애니메이션 중 layer.position(model=최종값) vs
#    layer.presentation()?.position(현재 보이는 값) 비교 출력

# 3) shadowPath 최적화: 그림자만(오프스크린) vs shadowPath 지정 → FPS 비교

# 4) Instruments: Core Animation(FPS·오프스크린), Time Profiler(레이아웃)

# 5) 세 트리 관찰: 애니메이션 중 히트 테스트가 presentation 기준임을 확인
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🍎 iOS 톤, "View를 배치한다 vs Layer가 어떻게 합성되나" 포지셔닝, `🔗 레포 연결`(gpu·swiftui·rendering-pipelines-compared)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- Core Animation 프로그래밍 가이드(Apple)
- "Core Animation: Advanced Techniques" (Nick Lockwood)
- "Designing for high performance" Core Animation WWDC 발표들
- *iOS Core Animation: Advanced Techniques*
- objc.io "Getting Pixels onto the Screen"

## 💡 핵심 분석 대상

```
View ≠ Layer:
  UIView   : 터치·제스처·레이아웃 (이벤트 책임)
  CALayer  : contents·애니메이션·합성 (그리기 책임)
  모든 UIView는 layer 프로퍼티(backing layer) 보유
  → 실제로 화면에 그려지는 건 Layer

세 Layer Tree:
  Model        : layer.position = 목표값(코드가 보는 값)
  Presentation : 지금 화면에 보이는 값(애니메이션 중간)
  Render       : 렌더 서버 내부(접근 불가)
  애니메이션 중 layer.position → 이미 최종값(model)!
  현재 위치는 layer.presentation()?.position
  → 흔한 "애니메이션 중 위치 버그"의 정체

렌더 파이프라인 (별도 프로세스):
  앱 프로세스: 레이아웃→display→커밋(RunLoop 끝)
       │ IPC
       ▼
  Render Server(backboardd): 합성 → GPU → 디스플레이
  → 앱이 잠깐 멈춰도 진행 중 애니메이션은 계속(서버가 함)

오프스크린 렌더링:
  cornerRadius + masksToBounds, shadow(경로 없이), 그룹 투명도
  → GPU가 별도 버퍼에 먼저 그린 뒤 합성(추가 패스)
  → 스크롤 중 반복되면 프레임 드롭
  → shadowPath 지정·미리 둥근 이미지로 회피
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 38개 확인 + Instruments/디버그색상 검증 환경 + gpu/swiftui/rendering-pipelines-compared 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
