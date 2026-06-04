# SwiftUI Internals Deep Dive 레포지토리 제작 프롬프트

나는 "SwiftUI Internals Deep Dive" 레포지토리를 만들려고 해.
선언적 뷰가 어떻게 화면 갱신이 되는지 — View Identity, `@State`/`@Binding`의 상태 저장, AttributeGraph 의존성 엔진, diffing과 갱신을 완전히 파헤치는 레포야.
"SwiftUI 뷰를 작성하는 것"과 "View가 왜 다시 그려지고 @State가 어디 저장되며 AttributeGraph가 어떻게 의존성을 추적하는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "SwiftUI 뷰를 짜는 것과, View Identity·상태 저장·AttributeGraph 갱신이 내부에서 어떻게 동작하는지 아는 것은 다르다"

**핵심 차별화**:
1. View는 값 타입 — struct View가 가벼운 설명서이고 실제 렌더 트리는 따로 있는 구조
2. View Identity — 구조적/명시적 정체성이 상태 보존·애니메이션을 결정
3. 상태 저장의 실체 — `@State`가 View struct 밖(프레임워크)에 저장되는 이유와 위치
4. AttributeGraph — 의존성 그래프가 변경된 부분만 정확히 갱신하는 반응성 엔진

**타겟 독자**:
- SwiftUI를 쓰지만 "왜 다시 그려지는지" 모르는 개발자
- `@State`·`@StateObject`·`@Binding` 선택을 외워서 하는 개발자
- View가 값 타입인데 상태가 유지되는 게 이상한 개발자
- 애니메이션·전환이 깨지는 이유(identity)를 모르는 개발자
- `jetpack-compose-internals`·`react-internals`와 비교하려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `swift-deep-dive`(값 타입·property wrapper·result builder), `ios-lifecycle-runloop-deep-dive`(렌더 타이밍).
**🤝 시너지**: `jetpack-compose-internals-deep-dive`(선언적 UI 대조), `react-internals-deep-dive`(diffing 대조), `uikit-core-animation-deep-dive`(아래 렌더).
**🧬 수렴**: `reactivity-state-compared`(AttributeGraph ↔ Compose Snapshot ↔ Signal ↔ MVCC), `rendering-pipelines-compared`.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 선언적 UI와 View (5개 문서)
- 명령형 UIKit → 선언적 SwiftUI — 뷰 갱신 책임의 이동, UI = f(state)
- View는 값 타입 — struct View, body는 가벼운 설명, 인스턴스가 아님
- result builder — `@ViewBuilder`가 body를 뷰 트리로(swift 연결)
- 뷰 트리 vs 렌더 트리 — 선언(struct)과 실제 렌더 노드의 분리
- 첫 렌더 vs 갱신 — body 호출 시점, 무엇이 갱신을 유발

### Chapter 2: View Identity (6개 문서)
- Identity의 중요성 — SwiftUI가 "같은 뷰"를 추적하는 방식, 상태·애니메이션의 기반
- 구조적 정체성 — 뷰 계층 위치로 정체성 부여, if/switch의 영향
- 명시적 정체성 — `.id()`·ForEach의 id, 언제 필요
- identity와 상태 — 정체성이 바뀌면 상태가 리셋되는 이유
- identity와 애니메이션 — 같은 정체성이어야 전환 애니메이션
- AnyView·타입 소거 — 정체성·성능에 주는 영향, 지양 이유

### Chapter 3: 상태 관리 property wrapper (7개 문서)
- property wrapper 복습 — @State 등이 어떻게 동작(swift 연결)
- @State — 값 타입 상태, View struct 밖(프레임워크 저장소)에 보관되는 이유
- @Binding — 상태 참조 전달, 양방향 바인딩의 구현
- @StateObject vs @ObservedObject — 소유·생명주기 차이, 재생성 함정
- @Environment·@EnvironmentObject — 트리 하향 주입, 의존성 전파
- Observation 프레임워크 — `@Observable` 매크로, 세밀 추적(신형)
- 선택 가이드 — 각 wrapper의 정확한 용도·함정

### Chapter 4: AttributeGraph — 반응성 엔진 (6개 문서)
- AttributeGraph 개요 — 의존성 그래프 기반 반응성(비공개 엔진)
- 의존성 추적 — 어떤 뷰가 어떤 상태에 의존하는지 그래프로
- 무효화와 갱신 — 상태 변경→의존 노드 무효화→body 재호출(정확히 그 부분만)
- body 재호출 vs 실제 렌더 — body가 불려도 실제 갱신은 diff 후
- 게으른 평가 — 필요한 것만 계산, 그래프 순서
- Compose·Signal과 동형 — 같은 "의존성 추적 반응성"의 다른 구현(reactivity 연결)

### Chapter 5: 레이아웃과 렌더 (5개 문서)
- 레이아웃 시스템 — 부모-자식 크기 협상(제안→선택), Compose와 대조
- 커스텀 레이아웃 — Layout 프로토콜, 정렬·우선순위
- diffing — 이전/현재 뷰 비교로 최소 갱신 결정
- UIKit 브리징 — UIViewRepresentable, 두 시스템의 경계(uikit 연결)
- 렌더 파이프라인 — SwiftUI→Core Animation→렌더 서버(uikit-core-animation 연결)

### Chapter 6: 성능과 함정 (5개 문서)
- 과한 body 재호출 — 무엇이 불필요한 재계산을 유발
- 상태 범위 — 상태를 적절한 위치에(상태 호이스팅), 갱신 범위 축소
- Equatable·EquatableView — 불필요 갱신 방지
- 무거운 body — 계산을 body 밖으로, 지연
- 리스트 성능 — List·LazyVStack, 대량 항목·identity

### Chapter 7: 측정과 실전 (4개 문서)
- 디버깅 — `Self._printChanges()`로 재호출 원인, Instruments SwiftUI
- 갱신 추적 — 무엇이·왜 갱신됐는지 특정
- AttributeGraph 관찰 — 그래프 덤프(비공식), 의존성 확인
- 종합 — 과한 갱신 화면을 _printChanges로 진단→identity/상태 정리→측정

→ **총 40개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 identity·상태 저장·AttributeGraph, `💻 실전 실험`은 `_printChanges`·Instruments·identity 실험. `📊`는 갱신 횟수·재호출 비교.

## 🎨 스타일 가이드

1. **"왜 다시 그려지나"에 답한다** — 모든 갱신을 identity·의존성으로 환원
2. **Compose/React와 3자 대조** — 위치 기억(Compose)·재조정(React)·의존성 그래프(SwiftUI)
3. **identity 실험** — `.id()`·구조 변경으로 상태 리셋을 *직접 본다*
4. **MVCC/Snapshot 동형 예고** — AttributeGraph가 reactivity-state-compared로
5. 뷰 트리·AttributeGraph·identity는 다이어그램으로

## 🔬 검증 환경

> macOS + Xcode. `_printChanges`와 Instruments로 관찰.

```bash
# 환경: macOS, Xcode

# 1) 갱신 원인 추적 (핵심)
#    var body: some View {
#      let _ = Self._printChanges()   // 무엇이 이 뷰를 갱신시켰나 콘솔 출력
#      ...
#    }

# 2) identity 실험:
#    if cond { ViewA() } else { ViewA() }  → 다른 구조적 정체성 → 상태 리셋
#    .id(value) 바꾸면 → 새 정체성 → 상태 리셋·재생성 관찰

# 3) @State 저장 위치 확인:
#    View struct는 매번 새로 생성되는데 @State 값은 유지됨
#    → 프레임워크가 별도 저장(AttributeGraph 노드) 입증

# 4) @StateObject vs @ObservedObject:
#    부모 갱신 시 @ObservedObject는 재생성, @StateObject는 유지 → 관찰

# 5) Instruments: SwiftUI 템플릿으로 view body·갱신 추적
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🍎 iOS 톤, "뷰를 짠다 vs 왜 갱신되나" 포지셔닝, `🔗 레포 연결`(swift·compose·react·reactivity-state-compared)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- "Demystify SwiftUI" (WWDC21) — identity·lifetime·dependencies 핵심
- "Demystify SwiftUI performance" (WWDC23)
- SwiftUI 공식 문서 — https://developer.apple.com/documentation/swiftui
- "Observation" 프레임워크 문서·WWDC23
- objc.io "Thinking in SwiftUI"

## 💡 핵심 분석 대상

```
View는 값 타입 (설명서일 뿐):
  struct ContentView: View {
    @State var count = 0
    var body: some View { Text("\(count)") }
  }
  ContentView()는 매 갱신마다 새로 생성(가벼움)
  하지만 count(@State)는 유지됨
  → @State는 struct 밖, 프레임워크(AttributeGraph)에 저장
  → 그래서 값 타입인데 상태가 유지됨

View Identity:
  같은 정체성 → 상태 유지 + 전환 애니메이션
  정체성 변경(.id 변경/구조 분기) → 상태 리셋 + 재생성
  if cond { A() } else { A() }  // 두 A는 다른 정체성!

AttributeGraph (반응성):
  body에서 count 읽음 → "이 뷰가 count에 의존" 그래프 기록
  count 변경 → 의존 노드만 무효화 → 그 body만 재호출
  → React 재조정·Compose Slot Table과 같은 목표, 다른 구현

3자 대조 (선언적 UI 반응성):
  React   : VDOM diff + Fiber 재조정
  Compose : Slot Table 위치 기억 + Snapshot
  SwiftUI : AttributeGraph 의존성 그래프
  → 모두 "변경된 부분만 갱신" (reactivity-state-compared)
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 40개 확인 + _printChanges/Instruments 검증 환경 + swift/compose/react/reactivity-state-compared 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
