# Security Engineering Deep Dive 레포지토리 제작 프롬프트

나는 "Security Engineering Deep Dive" 레포지토리를 만들려고 해.
Spring Security로 인증 필터를 설정하는 것과, SQL Injection이 PreparedStatement를 우회하는 조건, JWT alg:none 공격이 서명 검증을 무력화하는 원리, SSRF로 내부망을 탈출하는 방법을 알고 방어 코드를 설계하는 것은 다르다.

"Spring Security 설정했으니 안전하겠지"와 "공격자 관점에서 내 시스템의 취약점을 먼저 찾고, 각 공격 벡터를 원리부터 이해하여 방어하는 것"의 차이를 만드는 레포야.

## 📋 프로젝트 목표

**컨셉**: "보안 설정을 복붙하는 것과, 공격 원리를 알고 방어를 설계하는 것은 다르다"

**핵심 차별화**:
1. 공격자 관점 — 각 취약점이 실제로 어떻게 악용되는지 원리부터 이해
2. OWASP Top 10 심층 분석 — 표면적 대책이 아닌 근본 원인과 완전한 방어 설계
3. JWT/OAuth2 취약점 — Spring Security 설정의 흔한 실수가 어떤 공격에 노출되는지
4. 실전 방어 코드 — 취약한 코드 → 안전한 코드 Before/After 패턴

**타겟 독자**:
- `@PreAuthorize`, JWT 필터를 설정했지만 무엇을 방어하는지 설명 못하는 개발자
- SQL Injection을 "JPA 쓰면 안전하다"고 알고 있는 개발자 (JPQL도 취약할 수 있음)
- XSS를 "프론트엔드 문제"라고 생각하는 백엔드 개발자
- JWT를 쓰지만 alg 헤더를 검증하지 않는 개발자
- CSRF 토큰을 끄는 이유를 설명 못하는 개발자
- 개인정보/카드 데이터를 평문으로 DB에 저장하는 개발자

**선행 학습**:
- spring-security-deep-dive (Spring Security 인증/인가 체인 이해 필수 — 방어 코드 맥락)
- network-deep-dive (HTTPS/TLS, HTTP 헤더 이해)
- database-internals (SQL 실행 원리 — SQL Injection 완전 이해)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 보안 사고방식과 위협 모델링 (5개 문서)
- 공격자 관점 전환 — 방어자가 아닌 공격자로 생각해야 하는 이유, STRIDE 위협 모델(Spoofing/Tampering/Repudiation/Info Disclosure/DoS/Elevation), 내 시스템에서 가장 위험한 진입점 찾기
- 위협 모델링 방법론 — DFD(Data Flow Diagram)로 공격 표면 식별, PASTA(Process for Attack Simulation and Threat Analysis) 방법론, 우선순위 기반 위험 평가
- OWASP Top 10 2023 개요 — 10가지 취약점의 실제 발생 빈도와 비즈니스 영향, 각 취약점 간 연관성(SSRF → 내부망 SQL Injection 연결)
- Defense in Depth — 단일 방어선의 한계, 계층별 방어 설계(네트워크/애플리케이션/데이터 레이어), Security by Default 원칙
- 보안 개발 생명주기(SDL) — 설계 단계 위협 모델링, 코드 리뷰 보안 체크리스트, 배포 전 침투 테스트, 운영 중 모니터링

### Chapter 2: 인젝션 공격 완전 분해 (7개 문서)
- SQL Injection 원리 — 문자열 연결 쿼리가 어떻게 공격 표면이 되는가, `' OR '1'='1`이 WHERE 절을 무력화하는 원리, UNION 기반 데이터 추출
- Blind SQL Injection — 에러 메시지 없이 시간 기반(`SLEEP()`)으로 데이터를 추출하는 방법, 불리언 기반 추출, Spring/JPA에서 발생하는 조건
- JPA/JPQL에서의 SQL Injection — `@Query("SELECT u FROM User u WHERE u.name = '" + name + "'")` 취약 패턴, Native Query와 SpEL 주입 위험, 완전한 방어 코드
- 명령어 인젝션(Command Injection) — `Runtime.exec(userInput)`의 위험, OS 명령어가 실행되는 원리, ProcessBuilder 안전한 사용법
- LDAP/XML/NoSQL 인젝션 — 인젝션이 SQL에만 국한되지 않는 이유, MongoDB `$where` 연산자 인젝션, XXE(XML External Entity) 공격
- 인젝션 방어 원칙 — PreparedStatement/파라미터 바인딩, 입력값 화이트리스트 검증, 최소 권한 DB 계정, ORM 안전 사용 패턴
- 실전 취약점 재현 — DVWA/WebGoat를 Docker로 구동, SQL Injection 단계별 실습, Spring 방어 코드 Before/After 비교

### Chapter 3: 인증과 세션 취약점 (7개 문서)
- JWT 취약점 완전 분해 — `alg:none` 공격(서명 검증 우회), `RS256 → HS256` 알고리즘 혼동 공격, 공개키를 HMAC 비밀키로 사용, `kid` 헤더 인젝션
- JWT 안전한 구현 — 알고리즘 명시적 화이트리스트, 서명 키 분리(서명 키 vs 검증 키), 만료 시간 강제 검증, Spring Security 설정의 취약점
- 세션 고정 공격(Session Fixation) — 로그인 전 세션 ID를 공격자가 설정하여 탈취하는 원리, Spring Security의 `sessionFixation().changeSessionId()` 방어
- CSRF 공격 — Same-Origin Policy를 우회하는 form submission 방식, SameSite 쿠키 속성이 CSRF를 막는 원리, REST API에서 CSRF 토큰을 끄는 조건
- OAuth2 취약점 — `state` 파라미터 없는 CSRF, 오픈 리다이렉트를 통한 Authorization Code 탈취, PKCE 없는 Authorization Code Flow 위험성
- 브루트포스와 계정 보호 — Rate Limiting 없는 로그인 API의 위험, Exponential Backoff, 계정 잠금 정책이 DoS가 되는 역설, Captcha 적용 시점
- 비밀번호 저장 — MD5/SHA-1이 왜 비밀번호에 부적합한가(Rainbow Table), Bcrypt/Argon2의 솔트와 비용 인수(Cost Factor), Spring Security PasswordEncoder 내부

### Chapter 4: 웹 취약점 (XSS, CSRF, Clickjacking) (6개 문서)
- XSS 3가지 유형 — Reflected/Stored/DOM-based XSS의 공격 벡터 차이, `<script>document.cookie</script>`가 세션을 탈취하는 원리
- 백엔드 개발자의 XSS 방어 책임 — API 응답의 `Content-Type: application/json` 강제, `X-Content-Type-Options: nosniff` 헤더, HTML 이스케이핑이 필요한 서버사이드 렌더링
- CSP(Content Security Policy) — 스크립트 실행 도메인 화이트리스트, `script-src 'self'`가 외부 스크립트를 차단하는 원리, CSP 위반 리포트 수집
- Clickjacking — `<iframe>`으로 페이지를 숨겨 클릭을 가로채는 방법, `X-Frame-Options: DENY`, `frame-ancestors 'none'` CSP 지시어
- 보안 HTTP 헤더 완전 가이드 — `Strict-Transport-Security`(HSTS), `X-XSS-Protection`, `Referrer-Policy`, Spring Security의 `headers()` 설정
- Open Redirect — 로그인 후 리다이렉트 URL을 공격자가 조작하는 방법, 화이트리스트 검증 vs 도메인 검증의 우회 방법

### Chapter 5: 접근 제어와 권한 취약점 (6개 문서)
- IDOR(Insecure Direct Object Reference) — `/api/orders/12345`에서 다른 사용자 주문을 조회하는 공격, 소유권 검증 없는 API 설계의 위험
- 수평적 vs 수직적 권한 상승 — 일반 사용자가 다른 사용자 데이터 접근(수평) vs 관리자 권한 획득(수직), Spring Security `@PreAuthorize` 적용의 함정
- Mass Assignment 취약점 — JSON 요청의 예상치 못한 필드가 민감한 속성을 덮어쓰는 원리, `role: "ADMIN"` 주입, `@JsonIgnore`/DTO 분리로 방어
- API Rate Limiting 설계 — 엔드포인트별 차등 한도, 사용자/IP 기반 제한, Redis 슬라이딩 윈도우 Rate Limiter 구현
- JWT 권한 클레임 검증 — `sub`, `roles`, `scope` 클레임 검증 누락이 권한 상승을 허용하는 시나리오, 서버 측 권한 재검증 원칙
- 최소 권한 원칙 적용 — DB 계정 권한 최소화, AWS IAM 역할 범위 제한, Spring Security 메서드 보안 계층 설계

### Chapter 6: SSRF, 민감 데이터 노출, 설정 오류 (5개 문서)
- SSRF(Server-Side Request Forgery) — 서버가 공격자가 지정한 내부 URL을 요청하게 만드는 방법, 클라우드 환경 메타데이터 서비스(169.254.169.254) 탈취, AWS IMDSv2로 방어
- 민감 데이터 노출 — 로그에 개인정보/카드번호가 남는 패턴, Spring 예외 응답에 스택 트레이스 포함, `application.properties` 자격증명 Git 커밋
- 암호화 설계 — 대칭키(AES-256-GCM)와 비대칭키(RSA) 사용 시나리오 구분, AES-ECB가 패턴을 노출하는 이유, 암호화 키 관리(AWS KMS, Vault)
- 보안 설정 오류 — Spring Actuator 프로덕션 노출, CORS `allowedOrigins("*")` + `allowCredentials(true)` 조합의 위험, H2 Console 프로덕션 활성화
- 의존성 취약점 관리 — CVE(Common Vulnerabilities and Exposures) 모니터링, `./gradlew dependencyCheckAnalyze`로 취약 라이브러리 탐지, 자동 업데이트 전략

### Chapter 7: 보안 테스트와 운영 보안 (5개 문서)
- SAST(정적 분석) 자동화 — SpotBugs Security Plugin, SonarQube 보안 규칙, CI Pipeline에 보안 게이트 추가
- DAST(동적 분석) — OWASP ZAP으로 실행 중인 Spring 앱 자동 스캔, 주요 취약점 자동 탐지, 결과 해석
- 침투 테스트 방법론 — Reconnaissance → Scanning → Exploitation → Post-Exploitation, 버그 바운티 보고서 작성법
- 보안 로깅과 모니터링 — SIEM(Security Information and Event Management), 의심스러운 패턴 감지(비정상 로그인 시도, 대량 데이터 조회), Spring AOP로 보안 이벤트 로깅
- 인시던트 대응 — 침해 사고 발생 시 초기 대응 절차, 포렌식 로그 보존, 사용자 공지 의무(개인정보보호법), 사후 분석

---

각 챕터는 **5~7개 문서**로 구성해줘. 총 **41개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 취약점이 실무에서 위험한가 (실제 침해 사고 사례)
## 😱 취약한 코드 (Before — 원리를 모를 때의 구현)
## ✨ 방어 코드 (After — 공격 원리를 알고 설계한 구현)
## 🔬 공격 원리 분석 (공격자 관점의 내부 동작)
## 💻 실전 실험 (취약점 재현 → 공격 → 방어 검증)
## 📊 공격 성공 조건 vs 방어 성공 조건 비교
## ⚖️ 트레이드오프 (보안 강화 vs 사용성/성능)
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 모든 취약점은 실제 공격 코드/페이로드와 취약한 서버 코드를 함께 제시
2. 공격 → 방어 순서 (왜 위험한지 먼저 보여주고 방어 설계)
3. Spring Security와 직접 연결 — 설정의 어느 부분이 어떤 공격을 막는지 명시
4. 실제 CVE 번호와 영향받은 서비스 사례 포함 (교육적 맥락)
5. 취약한 Before 코드와 안전한 After 코드 반드시 비교

**실험 환경**:
```yaml
# docker-compose.yml
services:
  vulnerable-app:
    build:
      context: ./vulnerable-spring-app
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: vulnerable  # 취약한 설정

  secure-app:
    build:
      context: ./secure-spring-app
    ports:
      - "8081:8081"
    environment:
      SPRING_PROFILES_ACTIVE: secure      # 방어 설정

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: security_test

  webgoat:
    image: webgoat/webgoat:latest
    ports:
      - "8888:8888"   # 취약점 학습 플랫폼

  owasp-zap:
    image: ghcr.io/zaproxy/zaproxy:stable
    command: zap-webswing.sh
    ports:
      - "8090:8090"   # ZAP GUI
      - "8091:8091"   # ZAP API
```

```java
// 취약한 코드 vs 방어 코드 예시 (JWT alg:none 공격)

// ✗ 취약한 구현: 알고리즘 검증 없음
public Claims parseToken(String token) {
    return Jwts.parser()
        .setSigningKey(secretKey)
        .parseClaimsJws(token)  // alg:none이면 서명 검증 건너뜀
        .getBody();
}

// ✓ 방어 구현: 알고리즘 명시 + 화이트리스트
public Claims parseToken(String token) {
    return Jwts.parserBuilder()
        .requireAlgorithm("HS256")  // 허용 알고리즘 명시
        .setSigningKey(Keys.hmacShaKeyFor(secretKey.getBytes()))
        .build()
        .parseClaimsJws(token)
        .getBody();
}
```

```bash
# 보안 테스트 명령어 세트

# SQL Injection 테스트
sqlmap -u "http://localhost:8080/api/users?id=1" --dbs

# JWT 토큰 디코딩 (취약점 분석)
echo "eyJhbGciOiJub25lIn0.eyJzdWIiOiJhZG1pbiJ9." | \
  python3 -c "import sys,base64; print(base64.b64decode(sys.stdin.readline().split('.')[1]+'=='))"

# OWASP ZAP 자동 스캔
docker exec owasp-zap zap-cli --zap-url http://localhost:8090 \
  active-scan http://localhost:8080

# Trivy 취약점 스캔
trivy image myapp:latest --severity HIGH,CRITICAL

# 의존성 취약점 확인 (Gradle)
./gradlew dependencyCheckAnalyze
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**:
   - spring-security-deep-dive README 스타일 참고
   - "보안 설정을 복붙하는 것과 공격 원리를 알고 방어를 설계하는 것의 차이"라는 포지셔닝 강조
   - spring-security-deep-dive, network-deep-dive, database-internals와의 연결 지점 명시
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료

- OWASP Top 10 2023 — https://owasp.org/Top10/
- OWASP Testing Guide — https://owasp.org/www-project-web-security-testing-guide/
- PortSwigger Web Security Academy — https://portswigger.net/web-security (실습 무료)
- JWT 취약점 — https://auth0.com/blog/critical-vulnerabilities-in-json-web-token-libraries/
- The Web Application Hacker's Handbook (Stuttard, Pinto)
- NIST Cybersecurity Framework — https://www.nist.gov/cyberframework
- HackTricks — https://book.hacktricks.xyz/ (공격 기법 레퍼런스)

---

## 💡 핵심 분석 대상

```
SQL Injection 완전 흐름:

취약한 코드:
String query = "SELECT * FROM users WHERE id = " + userId;
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(query);

공격 페이로드: userId = "1 OR 1=1"
실행 쿼리: SELECT * FROM users WHERE id = 1 OR 1=1
결과: 전체 사용자 데이터 반환

UNION 기반 데이터 추출:
userId = "1 UNION SELECT table_name,2,3 FROM information_schema.tables--"
→ DB 스키마 구조 노출

방어:
PreparedStatement pstmt = conn.prepareStatement(
    "SELECT * FROM users WHERE id = ?"
);
pstmt.setInt(1, userId);  // 파라미터 바인딩 → SQL 구조 변경 불가

JWT alg:none 공격:
  정상 JWT: header.payload.signature
  공격 JWT: {"alg":"none"}.{"sub":"admin"}.  (서명 없음)
  → 취약한 서버: 알고리즘 "none" 허용 → 서명 검증 건너뜀
  → 공격자가 임의의 클레임으로 토큰 위조 가능

SSRF → 클라우드 메타데이터 탈취:
  공격자 요청: POST /api/fetch
  { "url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2-role" }
  → 서버가 AWS 메타데이터 서비스 요청
  → AWS 임시 자격증명 탈취
  → AWS S3/RDS 전체 접근 가능

  방어:
  1. URL 화이트리스트 (허용된 도메인만)
  2. IMDSv2 강제 (PUT 방식 토큰 필요)
  3. 내부 IP 범위 차단 (169.254.x.x, 10.x.x.x)
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (41개 목표)
- Docker Compose 실험 환경 (취약한 앱 + 방어 앱 + WebGoat + OWASP ZAP)
- spring-security-deep-dive, network-deep-dive, database-internals와 연결되는 지점 명시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
