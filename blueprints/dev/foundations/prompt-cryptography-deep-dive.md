# Cryptography Deep Dive 레포지토리 제작 프롬프트

나는 "Cryptography Deep Dive" 레포지토리를 만들려고 해.
대칭/비대칭 암호, 해시, 키 교환, 디지털 서명이 *수학적으로 왜 안전한지*와, TLS·JWT·비밀번호 저장이 그 위에 어떻게 세워졌는지를 완전히 파헤치는 레포야.
"라이브러리를 호출하는 것"과 "왜 이 알고리즘을 이 모드로 이 파라미터로 써야 하는지 아는 것"의 차이를 만드는 레포다.

> ⚠️ 이 레포는 **방어자·구현자 관점의 교육 자료**다. 공격 도구가 아니라, 안전하게 쓰기 위한 원리를 다룬다. `security-engineering-deep-dive`가 위협(공격)을 다룬다면, 이 레포는 그 밑의 *수학적 빌딩블록*을 다룬다.

## 📋 프로젝트 목표

**컨셉**: "암호 라이브러리를 호출하는 것과, 그 암호가 왜 안전하고 어떻게 깨지는지 아는 것은 다르다"

**핵심 차별화**:
1. 왜 안전한가 — 단방향성·계산 난해성(소인수분해·이산로그)에 기반한 안전성의 근거
2. 모드와 파라미터가 전부다 — ECB가 위험한 이유, IV·Nonce 재사용이 깨뜨리는 것, 패딩 오라클
3. 키 교환의 마법 — 도청자가 보는 앞에서 공유 비밀을 만드는 DH/ECDH의 원리
4. 실전 프로토콜의 조립 — TLS 핸드쉐이크·JWT·비밀번호 저장이 이 블록들을 어떻게 조합하나

**타겟 독자**:
- `AES`를 쓰지만 어떤 모드(GCM/CBC)를 왜 골라야 하는지 모르는 개발자
- 비밀번호를 해시하지만 bcrypt/Argon2와 SHA-256의 차이를 설명 못하는 개발자
- JWT를 쓰지만 `alg`·서명 검증이 실제로 뭘 보장하는지 모르는 개발자
- HTTPS가 "안전하다"고 알지만 TLS 핸드쉐이크가 뭘 교환하는지 모르는 개발자
- `network-deep-dive`에서 TLS를 봤지만 그 안의 암호 수학은 블랙박스인 개발자

## 🔗 레포 연결

**⬆️ 선행**: 없음(수학 기초만). `computer-architecture-deep-dive`(상수 시간 연산·사이드채널 이해에 도움).
**🤝 시너지**: `network-deep-dive`(TLS 1.3 핸드쉐이크의 암호 내부), `spring-security-deep-dive`(JWT·OAuth2의 서명·검증), `security-engineering-deep-dive`(알고리즘 혼동·`alg:none` 공격의 근원).
**🧬 수렴**: `distributed-systems-theory-deep-dive`(블록체인·합의의 암호 빌딩블록), 모든 Auth 논의의 바닥.

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 설계해줘:

### Chapter 1: 암호학의 토대 (5개 문서)
- 무엇을 보호하는가 — 기밀성/무결성/인증/부인방지, 각각 어떤 도구가 대응되나
- 공격자 모델 — Kerckhoffs 원칙("알고리즘은 공개, 키만 비밀"), 위협 가정의 중요성
- 난수와 엔트로피 — CSPRNG vs PRNG, 약한 난수가 모든 걸 무너뜨리는 사례, `/dev/urandom`
- 계산 난해성 — "안전 = 깨는 데 비현실적 시간", 소인수분해·이산로그 문제
- 인코딩 ≠ 암호화 — Base64·URL 인코딩과 암호화의 혼동이 만드는 사고

### Chapter 2: 대칭키 암호 (6개 문서)
- 블록 암호 기초 — AES 구조 개요, 블록·키 크기, 라운드, 왜 깨기 어려운가
- 운영 모드 — ECB(왜 위험한가, 펭귄 이미지), CBC, CTR, 모드가 보안을 좌우하는 이유
- IV와 Nonce — 초기화 벡터의 역할, 재사용 시 무엇이 새어나가나(CTR/GCM nonce 재사용)
- 인증된 암호화(AEAD) — GCM·ChaCha20-Poly1305, "암호화만으로는 무결성이 없다", MAC-then-encrypt 논쟁
- 패딩과 패딩 오라클 — PKCS#7, 패딩 오라클 공격의 원리와 방어
- 스트림 암호 — ChaCha20, 키스트림, 블록 암호와의 트레이드오프

### Chapter 3: 해시와 메시지 인증 (5개 문서)
- 암호학적 해시 — SHA-256/SHA-3, 단방향성·충돌 저항성, 일반 해시와의 차이
- 충돌과 공격 — 생일 문제, MD5/SHA-1이 깨진 이유, 길이 확장 공격(length extension)
- HMAC — 해시로 무결성+인증, "왜 그냥 hash(key‖msg)가 아닌가"(길이 확장 방어)
- 비밀번호 저장 — bcrypt/scrypt/Argon2, salt·work factor, 왜 SHA-256은 비밀번호에 부적합한가
- 키 유도(KDF) — PBKDF2·HKDF, 비밀번호/공유비밀에서 키를 안전하게 뽑기

### Chapter 4: 공개키 암호 (6개 문서)
- 비대칭의 직관 — 공개키/개인키, "암호화는 누구나, 복호화는 한 명", 대칭과의 역할 분담
- RSA — 모듈러 지수승, 키 생성(소수 두 개), 왜 소인수분해가 어려워서 안전한가, 패딩(OAEP)
- 타원곡선(ECC) — 같은 안전성에 더 짧은 키, 곡선 위의 점 덧셈, ECDLP
- Diffie-Hellman 키 교환 — 도청자 앞에서 공유 비밀 만들기, 모듈러 거듭제곱의 마법
- ECDH와 전방향 비밀성(PFS) — 임시 키(ephemeral), 과거 트래픽이 미래 키 유출에도 안전한 이유
- 하이브리드 암호 — 공개키로 대칭키를 교환하고 대칭키로 데이터 암호화(왜 섞어 쓰나)

### Chapter 5: 디지털 서명과 PKI (5개 문서)
- 디지털 서명 — 해시 후 개인키로 서명, 무결성+인증+부인방지, RSA/ECDSA/EdDSA
- 인증서와 신뢰 사슬 — X.509, CA, 루트→중간→리프, 브라우저가 신뢰를 검증하는 방식
- 인증서 검증 실패 시나리오 — 만료·도메인 불일치·해지(CRL/OCSP), 왜 검증을 끄면 안 되나
- 서명 알고리즘 혼동 — ECDSA nonce 재사용(PS3 사례), 알고리즘 다운그레이드
- 신뢰의 다른 모델 — Web of Trust, Certificate Transparency, 자가서명의 한계

### Chapter 6: 실전 프로토콜 — TLS와 토큰 (6개 문서)
- TLS 1.3 핸드쉐이크 완전 분해 — ClientHello→키 교환(ECDHE)→인증→Finished, 1-RTT
- TLS 1.2 vs 1.3 — 제거된 위험한 옵션들(RSA 키 전송·정적 DH), 왜 1.3이 더 안전하고 빠른가
- 세션 재개와 0-RTT — 재개의 편의와 0-RTT 재전송 공격의 위험
- JWT 완전 분해 — 헤더·페이로드·서명, HS256 vs RS256, `alg:none`·키 혼동 공격이 통하는 이유
- 토큰 보안 설계 — 만료·회전·해지, JWT를 세션처럼 쓸 때의 함정, refresh 토큰
- mTLS — 양방향 인증, 서비스 간 Zero Trust(gRPC/Service Mesh와 연결)

### Chapter 7: 구현의 함정과 사이드채널 (5개 문서)
- 타이밍 공격 — 비교 시간이 비밀을 누설, 상수 시간 비교(constant-time)의 필요성
- 사이드채널 개요 — 전력·캐시·타이밍, "수학은 안전한데 구현이 샌다"
- 직접 구현하지 말 것 — "Don't roll your own crypto", 검증된 라이브러리를 쓰는 이유
- 키 관리 — 키 저장(KMS/HSM)·회전·폐기, 하드코딩된 키 사고
- 마이그레이션과 민첩성(crypto-agility) — 알고리즘 교체 설계, 양자내성 암호(PQC) 대비

→ 각 챕터 5~6개 문서. **총 36개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 수학·바이트 레벨, `💻 실전 실험`은 openssl/코드로 직접 암호화·서명·검증 재현. `📊`는 알고리즘별 키 크기·속도·안전성 비교.

## 🎨 스타일 가이드

1. **"왜 안전한가"를 항상 먼저** — 알고리즘 사용법보다 안전성의 근거(난해성 문제)부터
2. **깨지는 사례로 가르친다** — IV 재사용·ECB·alg:none 등 *실패가 보여주는 원리* 중심
3. **openssl로 재현** — 추상 설명을 CLI 명령으로 직접 만들고 까보기
4. **방어자 관점 고정** — 공격 재현은 개념 증명 수준, 항상 "그래서 어떻게 막나"로 마무리
5. **실전 연결** — 각 빌딩블록이 TLS/JWT/비밀번호 저장 어디에 쓰이는지 명시

## 🔬 검증 환경

> 멀티 노드 불필요. **openssl CLI + 스크립트 언어 + Wireshark(TLS 캡처)** 가 검증 도구.

```dockerfile
# Dockerfile — 암호 실습 환경
FROM ubuntu:24.04
RUN apt-get update && apt-get install -y \
    openssl \
    python3 python3-pip \
    tshark
RUN pip3 install --break-system-packages cryptography pyjwt
```

```bash
# 핵심 재현 명령어

# 대칭키 — AES-GCM 암호화/복호화
openssl rand -hex 32   # 키
openssl enc -aes-256-gcm ...   # 모드별 차이 관찰

# 해시 / 비밀번호
echo -n "password" | openssl dgst -sha256
python3 -c "import bcrypt; print(bcrypt.hashpw(b'pw', bcrypt.gensalt()))"  # work factor 체감

# 키쌍 / 서명
openssl genpkey -algorithm RSA -out priv.pem
openssl ec -in ... # ECC
openssl dgst -sha256 -sign priv.pem msg.txt  # 서명 → 검증

# TLS 핸드쉐이크 들여다보기
openssl s_client -connect example.com:443 -tls1_3   # 협상된 cipher·인증서
# Wireshark filter: tls.handshake  (ClientHello/ServerHello 프레임 관찰)

# JWT alg 혼동 데모 (방어 학습용)
python3 jwt_alg_confusion_demo.py   # 왜 alg를 검증해야 하는지
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 폴더
2. **README.md**: 🪨 Foundations 톤 + ⚠️ 방어자 관점 명시, "라이브러리 호출 vs 왜 안전한지 안다" 포지셔닝, `network`/`spring-security`/`security-engineering` 연결
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- *Serious Cryptography* (Jean-Philippe Aumasson)
- *Cryptography Engineering* (Ferguson, Schneier, Kohno)
- *Real-World Cryptography* (David Wong)
- RFC 8446 (TLS 1.3), RFC 7519 (JWT)
- Cryptopals Challenges — https://cryptopals.com/ (구현 실패로 배우기)
- *A Graduate Course in Applied Cryptography* (Boneh & Shoup, 무료)

## 💡 핵심 분석 대상

```
대칭 vs 비대칭 (역할 분담):
  대칭(AES)   : 빠름, 키 공유가 문제      → 대량 데이터 암호화
  비대칭(RSA/EC): 느림, 키 공유 해결       → 키 교환·서명
  실전(TLS)   : 비대칭으로 대칭키 교환 → 대칭키로 데이터 (하이브리드)

Diffie-Hellman (도청자 앞에서 비밀 만들기):
  공개: g, p
  Alice: a 비밀 → A = g^a mod p (공개)
  Bob:   b 비밀 → B = g^b mod p (공개)
  공유키: Alice는 B^a, Bob은 A^b → 둘 다 g^(ab) mod p
  도청자: g,p,A,B 다 봤지만 g^(ab)는 못 구함 (이산로그 난해)

ECB가 위험한 이유:
  같은 평문 블록 → 항상 같은 암호문 블록
  → 이미지 암호화 시 패턴이 그대로 보임(유명한 펭귄)
  해결: CBC(이전 블록과 XOR) / CTR / GCM

JWT alg:none 공격 (왜 alg를 검증해야 하나):
  {"alg":"HS256"} 로 발급된 토큰을
  공격자가 {"alg":"none"} 로 바꾸고 서명 제거
  → 검증기가 alg를 신뢰하면 "서명 없음=통과" → 위조 성공
  방어: 서버가 허용 alg를 고정(토큰의 alg를 믿지 않음)

비밀번호 저장:
  SHA-256(pw)         ✗ 너무 빠름 → GPU로 초당 수십억 시도
  bcrypt/Argon2(pw)   ✓ 의도적으로 느림(work factor) + salt → 무차별 대입 방어
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목(5~6개) + 핵심 내용(3~4줄) + 총 36개 확인 + openssl/Wireshark 검증 환경 + network/spring-security/security-engineering 연결 지점 명시. **방어자 관점**을 일관되게 유지.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
