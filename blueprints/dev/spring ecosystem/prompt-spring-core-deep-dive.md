# Spring Core Deep Dive 레포지토리 제작 프롬프트

나는 "Spring Core Deep Dive" 레포지토리를 만들려고 해.
JVM Deep Dive, Unit Testing 프로젝트를 완성한 경험을 바탕으로, Spring의 IoC/DI/AOP 원리를 바이트코드 레벨까지 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "스프링 컨테이너가 Bean을 관리하는 모든 비밀"

**핵심 차별화**:
1. "어떻게 쓰나"가 아닌 "어떻게 동작하나" (원리 중심)
2. 바이트코드 레벨 분석 (JDK Proxy vs CGLIB)
3. Before/After 코드 비교 (잘못된 사용 → 올바른 사용)
4. Spring 소스코드 직접 추적

**타겟 독자**:
- Spring을 사용하지만 내부 동작을 모르는 개발자
- "@Transactional이 왜 private에서 안 되는지" 궁금한 개발자
- 순환 참조 에러를 만났지만 왜 생기는지 모르는 개발자
- Spring 면접 질문에 깊이 있게 답하고 싶은 개발자

**기존 레포와의 관계**:
- JVM Deep Dive: Spring이 JVM 위에서 동작하는 원리 연결
- Unit Testing: Spring Bean 테스트 전략 연결
- Design Patterns: Spring이 사용하는 패턴들 연결

---

## 🎯 1단계: 전체 구조 설계

다음 8개 챕터로 레포지토리를 설계해줘:

### Chapter 1: IoC Container Architecture (6개 문서)
- BeanFactory vs ApplicationContext 구조 차이
- BeanDefinition과 Bean 메타데이터
- BeanPostProcessor 체인 동작 원리
- ApplicationContext 계층 구조와 Parent-Child 관계
- Environment & PropertySource 우선순위
- Resource Abstraction 패턴

### Chapter 2: Dependency Injection Internals (7개 문서)
- 생성자 주입 vs 필드 주입 vs 세터 주입 (바이트코드 비교)
- 순환 참조 해결 메커니즘 (3단계 캐시)
- @Autowired vs @Inject vs @Resource 차이
- @Qualifier & @Primary 우선순위
- Optional Dependency 처리
- Constructor Binding의 장점
- Lazy Initialization 동작 원리

### Chapter 3: Bean Lifecycle Deep Dive (7개 문서)
- Bean 생성 과정 (Instantiation → Population → Initialization)
- @PostConstruct vs InitializingBean 실행 순서
- @PreDestroy vs DisposableBean 정리 과정
- Aware 인터페이스 체인 (BeanNameAware, ApplicationContextAware 등)
- Scope 종류와 프록시 (Singleton, Prototype, Request, Session)
- FactoryBean vs ObjectProvider 사용 시점
- Circular Dependencies 내부 해결 과정

### Chapter 4: AOP Implementation (8개 문서)
- JDK Dynamic Proxy vs CGLIB 바이트코드 비교
- ProxyFactoryBean 내부 구조
- @AspectJ 어노테이션 처리 과정
- Pointcut Expression 파싱과 매칭
- Advice 체인 실행 순서
- @Transactional이 프록시인 이유
- 왜 private 메서드는 AOP가 안 되는가
- Proxy 성능 비교 (JDK vs CGLIB)

### Chapter 5: Component Scanning (6개 문서)
- @ComponentScan 동작 원리
- ClassPathScanningCandidateComponentProvider 내부
- TypeFilter (Include/Exclude) 적용
- @Conditional 평가 과정
- Bean Definition 등록 과정
- Index를 통한 스캔 최적화

### Chapter 6: Configuration Classes (6개 문서)
- @Configuration vs @Component 차이
- CGLIB 프록시가 적용되는 이유
- @Bean 메서드 호출 가로채기
- Lite Mode vs Full Mode
- @Import의 3가지 방식
- ImportSelector & ImportBeanDefinitionRegistrar

### Chapter 7: Event & Listener (5개 문서)
- ApplicationEvent 발행/구독 메커니즘
- @EventListener vs ApplicationListener
- 비동기 이벤트 처리 (@Async)
- 이벤트 실행 순서 보장
- Transaction 바운드 이벤트

### Chapter 8: SpEL & Type Conversion (5개 문서)
- Spring Expression Language 파싱 과정
- @Value("${...}") vs @Value("#{...}") 차이
- PropertyEditor vs Converter 변환
- ConversionService 등록과 우선순위
- Custom Type Converter 작성

---

각 챕터는 **5~8개 문서**로 구성해줘. 총 **50개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 메커니즘이 존재하는가
## 😱 흔한 오해 또는 잘못된 사용 (Before)
## ✨ 올바른 이해와 사용 (After)
## 🔬 내부 동작 원리 (소스코드 추적)
## 💻 실험으로 확인하기 (실행 가능한 코드)
## 🤔 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. Spring Framework 소스코드 직접 추적
2. 바이트코드 레벨 분석 (javap 활용)
3. "왜 이렇게 설계됐는가?" 질문 중심
4. JVM Deep Dive처럼 깊이 파고들기
5. 실무 문제 해결 (순환 참조, 프록시 제약 등)

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**: 
   - JVM Deep Dive / Unit Testing README 참고
   - Spring Core에 맞게 변형
   - Dev Book Lab 소개 포함
   - 학습 경로 (초급 2주 / 중급 6주 / 고급 3개월)
3. **챕터별 문서 작성**: 
   - Chapter 1부터 순서대로
   - 한 챕터씩 완성 후 다음으로
   - 각 문서는 2500~3500 단어 분량

---

## 📚 참고 자료

<JVM Deep Dive README를 여기에 붙여넣기>

<Unit Testing README를 여기에 붙여넣기>

위 두 README를 참고해서 비슷한 구조와 느낌으로 Spring Core 버전을 만들어줘.

**차이점**:
- 주제: JVM 내부 → Spring Container 내부
- 스타일: 바이트코드 분석 + Spring 소스코드 추적
- 분량: 약 50개 문서

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~8개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (50개 목표)
- 주요 분석 대상 (예: BeanFactory 소스, CGLIB 바이트코드 등)

설계가 완료되면 내가 검토하고, 승인하면 2단계(디렉토리 생성)로 넘어가자.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
