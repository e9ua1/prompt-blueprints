# 오브젝트 (Object) 학습 노트 레포지토리 제작 프롬프트

나는 조영호 님의 "오브젝트: 코드로 이해하는 객체지향 설계" 책을 학습하면서 만드는 정리 레포지토리를 만들려고 해.
JVM Deep Dive, Modern Java in Action 학습 노트를 완성한 경험을 바탕으로, **책의 단계별 리팩터링 과정을 따라가며 객체지향 설계의 "왜?"를 한국어로 풀어내는** 학습 저장소를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "이론이 먼저가 아니라 코드가 먼저다"

**핵심 차별화**:
1. 책 챕터를 **그대로** 따르되, 각 챕터의 리팩터링 과정을 **단계별 Before → After 코드 비교**로 재구성
2. 책 본문 요약이 아니라 **"왜 이 설계가 필요한가"** 의 깊이 있는 탐구
3. AI와 대화하며 발견한 **추가 통찰** 을 별도 섹션으로 정리
4. 각 챕터의 **핵심 원칙을 실무 코드로 재해석** (Spring/JPA 환경에서 어떻게 적용되는가)
5. **설계 트레이드오프** — "어느 쪽이 정답이 아니라, 어느 쪽을 선택했고 왜인가"

**타겟 독자**:
- 객체지향을 "클래스 만들기"로만 이해하는 개발자
- SOLID 원칙은 외웠지만 실제 코드에 어떻게 적용하는지 모르는 개발자
- "데이터 중심 설계 vs 책임 중심 설계" 차이가 뭔지 헷갈리는 개발자
- 합성과 상속의 차이를 듣긴 했지만 실제로 언제 무엇을 쓰는지 모르는 개발자
- 디자인 패턴은 알지만 패턴 사용이 곧 좋은 설계라고 오해하는 개발자
- "책임 주도 설계(RDD)"를 이론으로만 아는 개발자

**기존 레포와의 관계**:
- **Effective Java**: 자바 모범 사례의 "왜"를 객체지향 설계 관점에서 보강
- **Java Design Patterns**: 패턴이 등장하기 이전, "왜 패턴이 필요한 설계 문제가 생기는가"를 보여줌
- **Modern Java in Action**: 함수형 패러다임이 OOP 설계에 어떤 영향을 미치는가

---

## 🎯 1단계: 전체 구조 (책 그대로 — 변경 금지)

이 레포는 **15개 챕터 + 부록 3개**로 구성된다. 챕터 순서·번호·제목은 책 그대로이며 추가/삭제/순서 변경 없이 이대로 작성한다.

### 📚 본문 (15개 챕터)

| Chapter | 주제 | 핵심 키워드 |
|:--------|------|------------|
| **[01. 객체, 설계](./chapter01/README.md)** | 티켓 판매 애플리케이션 | 의존성, 결합도, 응집도, 캡슐화 |
| **[02. 객체지향 프로그래밍](./chapter02/README.md)** | 영화 예매 시스템 | 협력, 메시지, 다형성 |
| **[03. 역할, 책임, 협력](./chapter03/README.md)** | 객체지향 설계의 핵심 | 책임 주도 설계 |
| **[04. 설계 품질과 트레이드오프](./chapter04/README.md)** | 영화 예매 시스템 개선 | 데이터 중심 vs 책임 중심 설계 |
| **[05. 책임 할당하기](./chapter05/README.md)** | 책임 할당 원칙 | GRASP 패턴 |
| **[06. 메시지와 인터페이스](./chapter06/README.md)** | 인터페이스 설계 | 퍼블릭 인터페이스 설계 |
| **[07. 객체 분해](./chapter07/README.md)** | 추상화 기법 | 프로시저 추상화, 데이터 추상화 |
| **[08. 의존성 관리하기](./chapter08/README.md)** | 의존성 다루기 | 의존성 역전 원칙 |
| **[09. 유연한 설계](./chapter09/README.md)** | 유연성 확보하기 | 개방-폐쇄 원칙 |
| **[10. 상속과 코드 재사용](./chapter10/README.md)** | 재사용 방법 비교 | 합성 vs 상속 |
| **[11. 합성과 유연한 설계](./chapter11/README.md)** | 합성 활용하기 | 믹스인, 인터페이스 |
| **[12. 다형성](./chapter12/README.md)** | 다형성의 종류 | 상속, 오버로딩, 제네릭 |
| **[13. 서브클래싱과 서브타이핑](./chapter13/README.md)** | 상속의 용도 | 리스코프 치환 원칙 |
| **[14. 일관성 있는 협력](./chapter14/README.md)** | 설계 일관성 | 협력 패턴 |
| **[15. 디자인 패턴과 프레임워크](./chapter15/README.md)** | 패턴과 프레임워크 | GoF 패턴, 프레임워크 |

### 📘 부록 (3개)

| Appendix | 주제 | 핵심 키워드 |
|:---------|------|------------|
| **[A. 계약에 의한 설계](./appendixA/README.md)** | 협력의 명시적 문서화 | 사전조건, 사후조건, 불변식, 공변성 |
| **[B. 타입 계층의 구현](./appendixB/README.md)** | 다양한 타입 구현 방법 | 인터페이스, 추상 클래스, 덕 타이핑, 믹스인 |
| **[C. 동적인 협력, 정적인 코드](./appendixC/README.md)** | 모델과 코드의 관계 | 동적 모델, 정적 모델, 도메인 모델, TYPE OBJECT |

**총 18개 문서 (챕터 15 + 부록 3) — 확정**

---

## 📐 챕터 문서 구조 (모든 챕터 동일 적용)

```markdown
## 📌 핵심 개념
챕터의 주요 학습 내용을 한국어로 요약. 단순 책 요약이 아닌 "왜 이 개념이 등장했는가"의 맥락 포함.

## 🎯 학습 목표
이 챕터를 통해 얻을 수 있는 것 (3~5개의 구체적 항목).

## 📖 핵심 예제 분석
책의 메인 예제 (티켓 판매, 영화 예매 등)를 코드 단계별로 분석.
- **Step 1: 초기 설계** — 어떤 문제가 있는가
- **Step 2: 1차 리팩터링** — 무엇을 바꾸었고 왜인가
- **Step 3: 최종 설계** — 어떤 원칙을 따르게 되었는가

## 💻 코드 분석 (Before / After)
리팩터링 전/후 코드를 좌우로 비교하며 변경 사유를 한 줄씩 짚기.
모든 예제는 Java 21 + IntelliJ에서 실행 가능.

## 🤔 깊이 파기 (AI와 대화하며 발견한 통찰)
챕터를 읽으며 든 의문과 그에 대한 분석.
- "이 원칙이 깨지면 어떤 코드 냄새가 나는가?"
- "Spring 환경에서는 어떻게 적용되는가?"
- "함수형 프로그래밍 관점에서 보면 어떻게 다른가?"

## 🔥 실무 적용 사례
책의 도메인(영화 예매)을 넘어, 실제 백엔드 개발에서 마주치는 상황으로 확장.
JPA Entity, Spring Service Layer 등 익숙한 맥락으로 재해석.

## ⚖️ 설계 트레이드오프
이 챕터의 원칙이 항상 옳은 것은 아니다. 어떤 상황에서 다르게 선택할 수 있는가.

## ✨ 핵심 정리 (한 화면 요약)
실전에서 적용할 수 있는 원칙을 5~7줄로 압축.

## 🤔 생각해볼 문제 (+ 해설)
이 챕터를 진정으로 이해했는지 검증하는 응용 문제 2~3개와 그 해설.
```

---

## 🎨 스타일 가이드

1. **책 요약이 아닌 "왜"의 탐구** — 책 본문을 그대로 옮기지 말고, "이 문장이 의미하는 바는 무엇인가"를 풀기
2. **모든 코드는 Java 21 기준 + record / Sealed Class 활용** — 책이 자바 8 시점이지만 모던 자바로 재해석
3. **Before/After 코드 비교를 모든 챕터에 강제** — 책의 리팩터링 단계를 그대로 보여주되, 각 단계의 결정 사유를 명시
4. **한국어 톤은 차분하고 분석적으로** — "~인 것이다", "~가 된다" 같은 단정형 사용
5. **Spring/JPA 맥락으로 재해석** — 책의 도메인이 추상적이므로 익숙한 백엔드 코드로 옮겨 보여주기
6. **"AI와 대화하며 발견한 통찰" 섹션은 본문과 별개로** — 책 너머의 사고를 보여주는 곳

---

## 🛠️ 실험 환경

- Java 21 + IntelliJ IDEA
- Gradle 8.x
- 책의 원본 코드 저장소(`eternity-oop/object`) 참고
- JUnit 5 + AssertJ (코드 검증)

---

## 🎯 2단계: 작업 순서

전체 구조는 위에 이미 확정되어 있으므로:

1. **디렉토리 생성**: bash 명령어로 18개 폴더 생성
   ```
   chapter01/ ~ chapter15/
   appendixA/  appendixB/  appendixC/
   images/  (커버 이미지 등)
   ```
2. **루트 README.md 작성**:
   - 책 소개 + Dev Book Lab 컨셉 명시
   - 15챕터 + 3부록 표 형식 목차
   - 학습 방법 (Read → AI Analysis → Deep Dive → Practice → Document)
   - 각 챕터 문서 구조 안내 (9섹션 통일)
3. **챕터별 문서 작성**:
   - Chapter 01부터 순서대로
   - 한 챕터 완성 후 다음으로 (책 흐름이 누적되므로 순서 준수 필수)
   - 각 문서는 3000~4500 단어 분량 (책의 단계별 리팩터링이 길어서 다른 시리즈보다 김)
   - 부록 3개는 본문 15장 모두 작성 후 마지막에

---

## 📚 참고 자료

- 원서: 조영호, 『오브젝트: 코드로 이해하는 객체지향 설계』, 위키북스
- 원서 코드: [eternity-oop/object](https://github.com/eternity-oop/object)
- 관련 서적: 『객체지향의 사실과 오해』 (조영호 저) — 1~3장의 개념적 배경

<JVM Deep Dive README를 여기에 붙여넣기>
<Effective Java README를 여기에 붙여넣기>

위 README들의 비주얼 톤 (배지, 표, 학습 방법 시각화)을 유지.

---

## 💡 핵심 분석 대상

### Chapter 01 — 티켓 판매 애플리케이션

```
Theater → Audience.getBag() → Bag.hold(ticket)
   ↓                                ↑
   └────────── 강한 결합 ──────────┘
```

→ Theater가 Audience와 Bag의 내부 구조를 알아야만 동작하는 코드 → 캡슐화 위반 → 자율적 객체로 전환

### Chapter 04 — 데이터 중심 vs 책임 중심 설계

```java
// ❌ 데이터 중심 — Movie가 데이터 덩어리
class Movie {
    private Money fee;
    private DiscountPolicy discountPolicy;
    public Money getFee() { return fee; }
    public DiscountPolicy getDiscountPolicy() { return discountPolicy; }
}

// 어딘가에서 외부 코드가 책임을 가져감
Money calculateFee(Movie movie, Screening screening) {
    Money discount = movie.getDiscountPolicy().calculateDiscount(screening);
    return movie.getFee().minus(discount);
}

// ✅ 책임 중심 — Movie 자신이 가격 계산을 책임
class Movie {
    public Money calculateMovieFee(Screening screening) {
        return fee.minus(discountPolicy.calculateDiscountAmount(screening));
    }
}
```

→ "묻지 말고 시켜라(Tell, Don't Ask)" 가 코드 변경 비용을 어떻게 줄이는지 추적

### Chapter 10 — 합성 vs 상속

```java
// ❌ 상속 — Phone이 NightlyDiscountPhone에 강하게 결합
class NightlyDiscountPhone extends Phone {
    @Override
    public Money calculateFee() {
        // 부모의 구현을 알아야 한다
    }
}

// ✅ 합성 — 정책을 객체로 분리
class Phone {
    private RatePolicy ratePolicy;
    public Money calculateFee() {
        return ratePolicy.calculateFee(this);
    }
}
```

→ 상속이 만드는 컴파일 시점 결합 vs 합성이 만드는 런타임 유연성을 코드로 보여주기

---

## 🎯 진행 방식

**구조는 이미 확정되어 있으므로 1단계는 생략하고 바로 2단계(디렉토리 생성)부터 시작.**

진행 순서:
1. 디렉토리 18개 생성 (chapter01~15 + appendixA/B/C)
2. 루트 README.md 작성
3. Chapter 01부터 순서대로 작성
4. 각 챕터마다 9섹션 구조 강제 준수
5. Before/After 코드 비교를 모든 챕터에 포함
6. 본문 15장 완료 후 부록 3개

**시작해줘! 디렉토리 생성 후 루트 README, 그다음 Chapter 01부터.**
