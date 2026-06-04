# Spring Security Deep Dive 레포지토리 제작 프롬프트

나는 "Spring Security Deep Dive" 레포지토리를 만들려고 해.
FilterChainProxy가 어떻게 15개 Filter를 거쳐 인증/인가하는지, JWT 토큰이 SecurityContext에 저장되는 과정을 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "요청이 Filter Chain을 거쳐 인증되는 전체 여정"

**핵심 차별화**:
1. Security Filter Chain 15개 완전 분해
2. Authentication vs Authorization 명확한 구분
3. JWT 인증 구현의 모든 과정
4. OAuth2/OpenID Connect 완전 정복

**타겟 독자**:
- Spring Security를 쓰지만 "마법" 같다고 느끼는 개발자
- Filter Chain이 무엇인지 모르는 개발자
- JWT 구현은 했지만 원리를 모르는 개발자
- OAuth2 Authorization Code Flow를 설명 못하는 개발자

**선행 학습**:
- Spring Core Deep Dive (Filter, AOP 이해 필수)
- Spring MVC Deep Dive (DispatcherServlet 이해 권장)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: Security Architecture (7개 문서)
- DelegatingFilterProxy와 FilterChainProxy 관계
- SecurityFilterChain 구성과 우선순위
- Security Filter 15개 완전 정복 (순서와 역할)
- SecurityContext & SecurityContextHolder (ThreadLocal)
- Authentication 객체 구조 (Principal, Credentials, Authorities)
- GrantedAuthority vs Role 차이
- SecurityContextPersistenceFilter 동작

### Chapter 2: Authentication Process (7개 문서)
- AuthenticationManager vs ProviderManager 차이
- AuthenticationProvider 체인 동작
- UserDetailsService 구현과 커스터마이징
- PasswordEncoder 종류와 선택 (BCrypt, Argon2, SCrypt)
- UsernamePasswordAuthenticationFilter 분석
- Remember-Me 인증 메커니즘
- Custom Authentication Provider 작성

### Chapter 3: Authorization & Method Security (6개 문서)
- @PreAuthorize vs @Secured vs @RolesAllowed
- Method Security 동작 원리 (AOP Proxy)
- FilterSecurityInterceptor 내부 구조
- AccessDecisionManager와 Voter 체인
- SpEL을 활용한 복잡한 권한 검사
- Custom Authorization Logic

### Chapter 4: Session Management (6개 문서)
- Session Fixation 공격과 방어
- Concurrent Session Control (동시 로그인 제한)
- Session Timeout 처리
- SessionRegistry 활용
- Stateless Session (JWT 환경)
- CSRF Protection 메커니즘

### Chapter 5: JWT Authentication (7개 문서)
- JWT 구조 완전 분석 (Header, Payload, Signature)
- Custom JWT Authentication Filter 작성
- JWT Token 발급 과정 (JwtTokenProvider)
- JWT Token 검증과 SecurityContext 저장
- Refresh Token 전략 (RTR - Refresh Token Rotation)
- Claims 추출과 사용
- JWT Token 만료 및 갱신 처리

### Chapter 6: OAuth2 & OpenID Connect (7개 문서)
- OAuth2 4가지 Grant Type (Authorization Code, Implicit, Password, Client Credentials)
- Authorization Code Flow 완전 분석
- OAuth2LoginAuthenticationFilter 동작
- ClientRegistration과 InMemoryClientRegistrationRepository
- OAuth2AuthorizedClient 관리
- Custom OAuth2UserService 작성
- JWT Bearer Token Resource Server

### Chapter 7: Advanced Security Topics (5개 문서)
- CORS Configuration (CorsFilter vs @CrossOrigin)
- Security Headers (CSP, XSS Protection, HSTS, X-Frame-Options)
- Security Events & Listeners (AuthenticationSuccessEvent)
- Method Security with SpEL 고급 활용
- Multi-tenancy Security 전략

---

각 챕터는 **5~7개 문서**로 구성해줘. 총 **45개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 보안 메커니즘이 필요한가
## 😱 흔한 보안 실수 (Before - 취약한 코드)
## ✨ 올바른 보안 구현 (After - 안전한 코드)
## 🔬 내부 동작 원리 (Spring Security 소스 추적)
## 💻 실험으로 확인하기 (Postman, curl)
## 🔒 보안 체크리스트
## 🤔 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. Security Filter 소스코드 직접 추적
2. 취약한 코드 → 안전한 코드 비교
3. Postman/curl로 인증/인가 테스트
4. JWT 토큰 디코딩 (jwt.io)
5. OWASP Top 10 연계

**실험 도구**:
- Postman (Authorization 탭)
- curl -H "Authorization: Bearer <token>"
- jwt.io (JWT 디코딩)
- Chrome DevTools (Network, Application 탭)
- @WithMockUser (Spring Security Test)

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**: 
   - Spring Core Deep Dive README 참고
   - Spring Security에 맞게 변형
   - 보안 중요성 강조
3. **챕터별 문서 작성**: 
   - Chapter 1부터 순서대로
   - 각 문서는 2500~3500 단어 분량

---

## 📚 참고 자료

<Spring Core Deep Dive README를 여기에 붙여넣기>

<Spring MVC Deep Dive README를 여기에 붙여넣기>

<JVM Deep Dive README를 여기에 붙여넣기>

위 README들을 참고해서 비슷한 구조로 Spring Security Deep Dive 버전을 만들어줘.

**차이점**:
- 주제: 인증/인가 메커니즘
- 초점: Filter Chain + JWT + OAuth2
- 보안: 취약점과 방어 전략

---

## 💡 핵심 분석 대상

```java
// FilterChainProxy가 거치는 주요 Filter들
1. SecurityContextPersistenceFilter      // SecurityContext 로드/저장
2. LogoutFilter                          // 로그아웃 처리
3. UsernamePasswordAuthenticationFilter  // Form 로그인
4. JwtAuthenticationFilter               // JWT 검증 (Custom)
5. BasicAuthenticationFilter             // Basic Auth
6. RequestCacheAwareFilter               // 이전 요청 복원
7. SecurityContextHolderAwareRequestFilter
8. AnonymousAuthenticationFilter         // 익명 사용자
9. SessionManagementFilter               // Session 관리
10. ExceptionTranslationFilter           // 인증/인가 예외 처리
11. FilterSecurityInterceptor            // 권한 검사

// JWT 인증 흐름
Request → JwtAuthenticationFilter
       → JwtTokenProvider.validateToken()
       → JwtTokenProvider.getAuthentication()
       → SecurityContextHolder.setContext()
       → Controller
```

이 흐름을 완전히 분해해서 설명!

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (45개 목표)
- 주요 분석 Filter와 클래스 명시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
