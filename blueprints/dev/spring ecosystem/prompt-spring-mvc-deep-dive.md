# Spring MVC Deep Dive 레포지토리 제작 프롬프트

나는 "Spring MVC Deep Dive" 레포지토리를 만들려고 해.
DispatcherServlet이 HTTP 요청을 어떻게 처리하는지, @RequestMapping이 메서드와 어떻게 매핑되는지 완전히 파헤치는 레포를 만들 거야.

## 📋 프로젝트 목표

**컨셉**: "HTTP 요청이 Controller 메서드에 도달하는 전체 여정"

**핵심 차별화**:
1. DispatcherServlet.doDispatch() 메서드 한 줄씩 분석
2. HandlerMapping → HandlerAdapter → ViewResolver 전체 체인
3. @RequestBody가 객체가 되는 과정 (HttpMessageConverter)
4. Exception이 JSON으로 변환되는 메커니즘

**타겟 독자**:
- 매일 @RestController를 쓰지만 내부 동작을 모르는 개발자
- DispatcherServlet 면접 질문에 막히는 개발자
- ArgumentResolver, ReturnValueHandler를 커스터마이징해야 하는 개발자
- @ExceptionHandler가 어떻게 동작하는지 궁금한 개발자

**선행 학습**:
- Spring Core Deep Dive (AOP, Proxy 이해 필수)
- 독립적 학습도 가능 (MVC는 별도 영역)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: DispatcherServlet Architecture (7개 문서)
- Servlet과 Front Controller 패턴
- DispatcherServlet 초기화 과정 (onRefresh)
- WebApplicationContext vs RootApplicationContext 계층 구조
- HTTP Request 처리 전체 흐름 (doDispatch 분석)
- HandlerMapping 체인 동작
- HandlerAdapter의 역할
- ViewResolver 메커니즘

### Chapter 2: Request Mapping (7개 문서)
- @RequestMapping 처리 과정 (RequestMappingHandlerMapping)
- RequestMappingInfo 생성과 매칭
- Path Pattern 매칭 전략 (AntPathMatcher vs PathPattern)
- HTTP Method 매칭 (GET, POST, PUT, DELETE)
- Content Negotiation (Accept 헤더)
- URI Template Variables 추출
- Matrix Variables와 사용 시점

### Chapter 3: Argument Resolution (7개 문서)
- HandlerMethodArgumentResolver 체인
- @RequestBody와 HttpMessageConverter (JSON → Object)
- @RequestParam vs @PathVariable 처리
- @ModelAttribute 데이터 바인딩 과정
- Validation (@Valid, BindingResult) 동작 원리
- Servlet API 주입 (HttpServletRequest, HttpSession)
- Custom Argument Resolver 작성

### Chapter 4: Response Handling (6개 문서)
- HandlerMethodReturnValueHandler 체인
- @ResponseBody 처리 (Object → JSON)
- ResponseEntity vs @ResponseStatus 선택
- HttpMessageConverter 선택 과정 (canWrite 체크)
- Content Negotiation과 Accept 헤더
- Custom Return Value Handler 작성

### Chapter 5: Exception Handling (6개 문서)
- HandlerExceptionResolver 체인 동작
- @ExceptionHandler 메서드 매칭 과정
- @ControllerAdvice 적용 범위 (basePackages, annotations)
- ResponseEntityExceptionHandler 내부 구조
- Custom Error Response 설계
- Exception 처리 우선순위와 상속

### Chapter 6: Interceptor & Filter (6개 문서)
- Filter vs HandlerInterceptor 차이점
- HandlerInterceptor 체인 실행 순서
- preHandle → postHandle → afterCompletion 호출 시점
- CORS 처리 (CorsFilter, @CrossOrigin)
- Custom Interceptor 작성 패턴
- Async Request와 Interceptor

### Chapter 7: Advanced MVC Topics (6개 문서)
- Async Request Processing (@Async, Callable, DeferredResult)
- Server-Sent Events (SSE) 구현
- File Upload (MultipartResolver, MultipartFile)
- Static Resources Handling (ResourceHandler)
- HTTP Caching (Cache-Control, ETag)
- WebMvcConfigurer 커스터마이징

---

각 챕터는 **6~7개 문서**로 구성해줘. 총 **45개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 메커니즘이 존재하는가
## 😱 흔한 오해 또는 실수 (Before)
## ✨ 올바른 이해와 사용 (After)
## 🔬 내부 동작 원리 (Spring MVC 소스 추적)
## 💻 실험으로 확인하기 (디버깅 포인트, 로그)
## 🌐 HTTP 레벨 분석 (Request/Response 헤더)
## 🤔 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. DispatcherServlet 소스코드 직접 추적
2. 디버깅 포인트 제시 (어디에 브레이크포인트)
3. HTTP Request/Response 헤더 분석
4. curl 명령어로 실험
5. 실무 문제 해결 (CORS, File Upload 등)

**실험 도구**:
- curl -v (HTTP 헤더 확인)
- Chrome DevTools Network 탭
- Spring MVC Test (MockMvc)
- @WebMvcTest
- 디버거 (IntelliJ)

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**: 
   - Spring Core Deep Dive README 참고
   - Spring MVC에 맞게 변형
   - HTTP 레벨 분석 강조
3. **챕터별 문서 작성**: 
   - Chapter 1부터 순서대로
   - 각 문서는 2500~3500 단어 분량

---

## 📚 참고 자료

<Spring Core Deep Dive README를 여기에 붙여넣기>

<JVM Deep Dive README를 여기에 붙여넣기>

<Unit Testing README를 여기에 붙여넣기>

위 README들을 참고해서 비슷한 구조로 Spring MVC Deep Dive 버전을 만들어줘.

**차이점**:
- 주제: HTTP 요청 처리 메커니즘
- 초점: DispatcherServlet 체인 분석
- 실험: curl, MockMvc, 디버거

---

## 💡 핵심 분석 대상

```java
// DispatcherServlet.java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) {
    // 1. HandlerMapping에서 Handler 찾기
    HandlerExecutionChain mappedHandler = getHandler(request);
    
    // 2. HandlerAdapter 찾기
    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
    
    // 3. Interceptor preHandle
    if (!mappedHandler.applyPreHandle(request, response)) {
        return;
    }
    
    // 4. 실제 Handler 실행
    ModelAndView mv = ha.handle(request, response, mappedHandler.getHandler());
    
    // 5. Interceptor postHandle
    mappedHandler.applyPostHandle(request, response, mv);
    
    // 6. View 렌더링
    processDispatchResult(request, response, mappedHandler, mv, dispatchException);
}
```

이 메서드를 한 줄씩 완전히 분해해서 설명!

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (6~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (45개 목표)
- 주요 분석 클래스 (DispatcherServlet, HandlerMapping 등)

**준비됐으면 1단계 구조 설계부터 시작해줘!**
