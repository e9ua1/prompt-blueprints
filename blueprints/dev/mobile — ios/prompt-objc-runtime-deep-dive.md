# Objective-C Runtime Deep Dive 레포지토리 제작 프롬프트

나는 "Objective-C Runtime Deep Dive" 레포지토리를 만들려고 해.
Swift가 서 있는 토대 — `objc_msgSend` 메시지 디스패치, isa·메타클래스, 메서드 캐시, 메서드 스위즐링, KVO가 *런타임에서 어떻게 동작*하는지를 완전히 파헤치는 레포야.
"Objective-C/Swift를 쓰는 것"과 "메시지 전송이 런타임에서 어떻게 메서드를 찾고 KVO가 어떻게 마법처럼 동작하는지 아는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "메서드를 호출하는 것과, objc_msgSend가 런타임에 어떻게 메서드를 찾아 디스패치하는지 아는 것은 다르다"

**핵심 차별화**:
1. 동적 메시지 디스패치 — 메서드 호출이 컴파일타임 결정이 아닌 런타임 조회인 원리
2. isa와 메타클래스 — 객체→클래스→메타클래스 체인, 클래스도 객체인 구조
3. 메서드 해결 흐름 — 캐시→메서드 리스트→상위→동적 해결→포워딩의 단계
4. 런타임 마법의 정체 — 스위즐링·KVO·연관 객체가 런타임 조작으로 가능한 이유

**타겟 독자**:
- iOS를 하지만 Swift 아래 Objective-C 런타임을 모르는 개발자
- KVO·스위즐링을 쓰지만 어떻게 동작하는지 모르는 개발자
- `@objc dynamic`이 왜 필요한지 모르는 개발자
- 크래시 로그의 `objc_msgSend`를 못 읽는 개발자
- `swift-deep-dive`의 디스패치가 어디서 오는지 알고 싶은 개발자

## 🔗 레포 연결

**⬆️ 선행**: `computer-architecture-deep-dive`(포인터·메모리), `compiler-deep-dive`(디스패치 일반).
**🤝 시너지**: `swift-deep-dive`(Swift의 objc 상호운용·dynamic), `ios-lifecycle-runloop-deep-dive`(런타임 위 앱), `uikit-core-animation-deep-dive`(UIKit은 objc 기반).
**🧬 수렴**: `swift-deep-dive`의 메시지 디스패치 뿌리. 모든 UIKit 동작의 바닥.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 동적 런타임의 개념 (5개 문서)
- 동적 언어로서의 objc — 컴파일타임이 아닌 런타임에 메서드 결정, C 위의 객체 계층
- objc_msgSend — `[obj method]`가 `objc_msgSend(obj, selector)`로 변환
- 셀렉터(SEL) — 메서드 이름의 런타임 표현, 셀렉터 = 메서드 아님
- IMP — 실제 함수 포인터, SEL→IMP 매핑이 디스패치의 핵심
- 정적 vs 동적 — C++/Swift 정적 디스패치와의 근본 차이, 유연성 대 비용

### Chapter 2: 객체와 클래스 구조 (6개 문서)
- 객체 = isa + 인스턴스 변수 — 모든 객체의 첫 멤버 isa 포인터
- isa 체인 — 인스턴스→클래스→메타클래스→루트 메타클래스
- 메타클래스 — 클래스 메서드를 담는 곳, "클래스도 객체"
- 클래스 구조체 — 메서드 리스트·ivar·프로퍼티·프로토콜
- 상속 체인 — superclass 포인터, 메서드 조회가 위로 올라가는 길
- non-fragile ivar — ivar 레이아웃이 런타임에 결정되는 ABI 안정성

### Chapter 3: 메서드 디스패치 흐름 (6개 문서)
- 디스패치 전체 흐름 — 캐시→클래스 메서드 리스트→상위 클래스→동적 해결→포워딩
- 메서드 캐시 — 최근 호출 SEL→IMP 캐싱, 해시, 캐시 미스 비용
- 메서드 조회 — 클래스 계층을 따라 SEL 검색, 못 찾으면?
- 동적 메서드 해결 — `resolveInstanceMethod:`로 런타임에 메서드 추가
- 메시지 포워딩 — `forwardingTargetForSelector:`·`forwardInvocation:`, 프록시 패턴
- doesNotRecognizeSelector — 최종 실패, unrecognized selector 크래시

### Chapter 4: 런타임 조작 (6개 문서)
- 런타임 API — class_addMethod·method_exchangeImplementations 등 C API
- 메서드 스위즐링 — IMP 교체로 기존 메서드 가로채기, 위험과 규칙
- 연관 객체(Associated Objects) — 카테고리에서 저장 프로퍼티 흉내
- 카테고리·익스텐션 — 런타임에 메서드 추가, 로드 시점, 충돌
- 클래스 동적 생성 — objc_allocateClassPair로 런타임 클래스
- isa swizzling — KVO가 쓰는 기법, 인스턴스의 클래스를 몰래 바꾸기

### Chapter 5: KVO와 KVC (5개 문서)
- KVC — 키 경로로 프로퍼티 접근, 셀렉터 기반 동적 접근
- KVO 원리 — 관찰 시작 시 isa swizzling으로 동적 서브클래스 생성
- 동적 서브클래스 — setter를 오버라이드해 변경 통지를 끼워넣는 방식
- KVO 함정 — 수동 통지·스레드·제거 누락, Swift에서의 KVO
- 통지 메커니즘 — willChange/didChange, 의존 키

### Chapter 6: Swift와의 상호운용 (5개 문서)
- Swift↔objc 브리징 — `@objc`·`@objc dynamic`, 언제 필요한가
- dynamic의 의미 — Swift가 objc 메시지 디스패치를 쓰도록 강제(KVO·스위즐링 위해)
- 브리징 비용 — NSObject 상속·브리징 변환의 오버헤드
- Swift에서 런타임 — Mirror vs objc 런타임, 제약
- 상호운용 함정 — 셀렉터·옵셔널·이름 매핑

### Chapter 7: 디버깅과 실전 (4개 문서)
- 크래시 읽기 — `objc_msgSend` 크래시·unrecognized selector 분석
- 런타임 탐색 — class_copyMethodList 등으로 런타임 내부 덤프
- 도구 — `image lookup`·lldb로 메서드·IMP 조사
- 종합 — 스위즐링 기반 로깅/분석 도구를 안전하게 구현

→ **총 35개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 런타임 구조체·디스패치 흐름, `💻 실전 실험`은 런타임 API·lldb·런타임 덤프. `📊`는 디스패치·캐시 히트의 성능.

## 🎨 스타일 가이드

1. **"메시지를 보낸다"로 사고** — 메서드 호출이 아닌 메시지 전송 멘탈 모델
2. **런타임을 덤프** — class_copyMethodList 등으로 *직접 본다*
3. **Swift와 연결** — `dynamic`·KVO가 왜 objc 런타임을 필요로 하는지
4. **마법을 해부** — KVO·스위즐링이 isa swizzling으로 가능함을 보여준다
5. isa 체인·디스패치 흐름은 다이어그램으로

## 🔬 검증 환경

> macOS + Xcode 권장(objc 런타임). lldb로 런타임 관찰.

```bash
# 환경: macOS, Xcode (또는 GNUstep으로 일부 실습)

# 런타임 덤프 (코드로)
#  unsigned int count;
#  Method *methods = class_copyMethodList([NSString class], &count);
#  → 메서드·IMP 직접 열람

# objc_msgSend 관찰
clang -S -emit-llvm demo.m              # [obj m] → objc_msgSend 호출 확인
otool -tv a.out | grep msgSend          # 어셈블리에서 msgSend

# lldb로 런타임 조사
(lldb) image lookup -rn 'objc_msgSend'
(lldb) po object_getClass(obj)          # isa 확인
(lldb) expr (void)class_addMethod(...)  # 런타임 조작

# KVO isa swizzling 확인:
#   관찰 전: object_getClass(obj) → Foo
#   관찰 후: object_getClass(obj) → NSKVONotifying_Foo (동적 서브클래스!)

# 스위즐링 실험: method_exchangeImplementations로 메서드 가로채기
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🍎 iOS 톤, "메서드를 호출한다 vs 메시지가 어떻게 디스패치되나" 포지셔닝, `🔗 레포 연결`(swift·uikit)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- objc 런타임 문서(Apple) — https://developer.apple.com/documentation/objectivec/objective-c_runtime
- objc 런타임 소스 — https://github.com/apple-oss-distributions/objc4
- "Objective-C Runtime Programming Guide" (Apple)
- Mike Ash "Friday Q&A" (런타임 심층)
- *Effective Objective-C 2.0* (Matt Galloway)

## 💡 핵심 분석 대상

```
메시지 전송:
  [obj doSomething]
  ↓ 컴파일러
  objc_msgSend(obj, @selector(doSomething))
  ↓ 런타임
  1. obj->isa 로 클래스 찾기
  2. 클래스 메서드 캐시에서 SEL 조회 (빠른 경로)
  3. 미스 → 클래스 메서드 리스트 검색
  4. 없음 → superclass 따라 위로
  5. 끝까지 없음 → 동적 해결 → 포워딩 → 크래시

isa 체인 (클래스도 객체):
  instance --isa--> Class --isa--> Metaclass --isa--> RootMetaclass
            (인스턴스 메서드)  (클래스 메서드)
  Class --superclass--> SuperClass (상속)

KVO = isa swizzling:
  addObserver 호출 시:
    obj의 클래스 Foo → 런타임이 NSKVONotifying_Foo 동적 생성
    obj의 isa를 몰래 NSKVONotifying_Foo로 변경
    이 서브클래스가 setter 오버라이드 → willChange/didChange 끼움
  → "마법"의 정체 = 런타임 isa 조작

Swift dynamic:
  class Foo: NSObject {
    @objc dynamic var x = 0   // objc 메시지 디스패치 강제
  }
  → KVO·스위즐링 가능 (없으면 Swift 정적 디스패치라 불가)
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 35개 확인 + lldb/런타임덤프 검증 환경 + swift/uikit 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
