# Network Deep Dive 레포지토리 제작 프롬프트

나는 "Network Deep Dive" 레포지토리를 만들려고 해.
TCP가 어떻게 신뢰성을 보장하는지, TLS 핸드쉐이크에서 실제로 무슨 일이 일어나는지, HTTP/2가 HTTP/1.1의 어떤 문제를 어떻게 해결하는지를 완전히 파헤치는 레포를 만들 거야.

단순한 네트워크 개념 정리가 아니야. "패킷을 보내는 것"과 "패킷이 어떤 경로로, 어떤 보장과 함께 전달되는지 아는 것"의 차이를 만드는 레포야.

## 📋 프로젝트 목표

**컨셉**: "HTTP를 쓰는 것과, TCP가 어떻게 연결을 맺고 끊는지 아는 것은 다르다"

**핵심 차별화**:
1. TCP 연결 수립/종료 — 3-Way/4-Way Handshake가 왜 그 횟수인가, TIME_WAIT이 왜 필요한가
2. TLS 완전 분해 — 공개키/대칭키가 핸드쉐이크에서 어떻게 결합되는가
3. HTTP 진화 — HTTP/1.1 → HTTP/2 → HTTP/3(QUIC)에서 어떤 문제를 어떻게 풀었는가
4. 운영 진단 — tcpdump/Wireshark로 실제 패킷을 분석하는 방법

**타겟 독자**:
- HTTP 요청을 보내지만 TCP 3-Way Handshake가 무엇인지 설명 못하는 개발자
- HTTPS를 쓰지만 TLS가 어떻게 암호화를 설정하는지 모르는 개발자
- HTTP/2를 쓰지만 HTTP/1.1과 실질적으로 무엇이 다른지 모르는 개발자
- TIME_WAIT이 왜 발생하는지, 왜 줄이면 안 되는지 모르는 개발자
- 네트워크 장애 발생 시 tcpdump조차 써본 적 없는 개발자
- Spring MVC, MySQL, Docker를 쓰면서 네트워크 레이어를 블랙박스로 두는 개발자

**선행 학습**:
- 없음 (이 레포 자체가 네트워크 기초부터 심화까지 독립 완결)
- spring-mvc-deep-dive (HTTP 요청 처리 흐름과 연결하면 시너지)
- mysql-deep-dive (DB 연결, SSL/TLS 설정과 연결하면 시너지)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: 네트워크 계층과 패킷 흐름 (5개 문서)
- OSI 7계층 vs TCP/IP 4계층 — 계층 모델이 왜 필요한가, 각 계층이 실제로 하는 일
- IP와 라우팅 — 패킷이 출발지에서 목적지까지 찾아가는 방법 (라우팅 테이블, TTL, 단편화)
- Ethernet과 ARP — 같은 네트워크 안에서 MAC 주소로 통신하는 원리
- NAT와 포트 포워딩 — 사설 IP가 공인 IP를 통해 인터넷에 나가는 방법, NAPT 동작 원리
- 네트워크 진단 도구 — tcpdump, wireshark, netstat, ss, ping, traceroute 실전 활용

### Chapter 2: TCP 완전 분해 (7개 문서)
- TCP 연결 수립 — 3-Way Handshake가 왜 3번인가, SYN/SYN-ACK/ACK가 하는 일
- TCP 연결 종료 — 4-Way Handshake와 TIME_WAIT 상태, FIN_WAIT/CLOSE_WAIT 이해
- TCP 신뢰성 보장 — Sequence Number, ACK, 재전송 타이머, 중복 제거
- TCP 흐름 제어 — 수신 버퍼와 윈도우 사이즈, Sliding Window 프로토콜
- TCP 혼잡 제어 — Slow Start, Congestion Avoidance, Fast Retransmit/Recovery
- TCP vs UDP — 각각의 내부 구조 차이, 언제 UDP를 선택하는가 (게임, DNS, QUIC)
- TCP 소켓 상태 머신 — LISTEN/SYN_SENT/ESTABLISHED/TIME_WAIT 전환 조건과 운영 영향

### Chapter 3: HTTP 완전 분해 (6개 문서)
- HTTP/1.1 내부 구조 — 요청/응답 메시지 포맷, Keep-Alive, 파이프라이닝의 한계
- HTTP/2 — 바이너리 프레이밍, 멀티플렉싱이 HOL Blocking을 해결하는 방식, 헤더 압축(HPACK)
- HTTP/3과 QUIC — UDP 위에서 신뢰성을 구현하는 방법, 0-RTT 핸드쉐이크 원리
- HTTP 캐싱 완전 분해 — Cache-Control 지시어, ETag, 조건부 요청, 캐시 무효화 전략
- HTTP 메서드와 상태 코드 — 멱등성/안전성의 의미, 올바른 상태 코드 선택 기준
- WebSocket — HTTP Upgrade 핸드쉐이크, 프레임 구조, Long Polling과의 차이

### Chapter 4: TLS/HTTPS 완전 분해 (6개 문서)
- 암호화 기초 — 대칭키/비대칭키/해시 함수가 각각 어디에 쓰이는가
- TLS 1.2 핸드쉐이크 — ClientHello부터 Finished까지 패킷별 상세 분석
- TLS 1.3 개선점 — 1-RTT 핸드쉐이크, Forward Secrecy, 0-RTT의 재전송 공격 위험
- 인증서와 PKI — X.509 구조, CA 체인 검증, Self-Signed vs 공인 인증서
- HTTPS 성능 최적화 — Session Resumption(Session ID vs Session Ticket), OCSP Stapling
- mTLS — 서버+클라이언트 양방향 인증, 마이크로서비스 간 Zero Trust 보안

### Chapter 5: DNS 완전 분해 (4개 문서)
- DNS 조회 흐름 — 재귀 vs 반복 쿼리, Resolver/Root NS/TLD NS/Authoritative NS 역할
- DNS 레코드 완전 가이드 — A, AAAA, CNAME, MX, TXT, SRV, PTR 레코드별 용도
- DNS 캐싱과 전파 — TTL이 캐시 갱신에 미치는 영향, 배포 시 DNS 전파 지연 대응
- DNS 보안과 고급 — DNSSEC 서명 검증, DNS over HTTPS/TLS, DNS 기반 로드밸런싱

### Chapter 6: 로드 밸런싱과 프록시 아키텍처 (5개 문서)
- L4 vs L7 로드 밸런서 — 어느 계층에서 분산하는가, 각각의 구현 방식과 한계
- 리버스 프록시 — Nginx 내부 동작, upstream 연결 관리, 버퍼링 원리
- 연결 유지 전략 — Sticky Session vs Stateless 설계, Session Affinity 구현 방법
- Rate Limiting 알고리즘 — Fixed Window, Sliding Window, Token Bucket, Leaky Bucket 비교
- 서킷 브레이커 패턴 — 네트워크 장애 전파 방지, Half-Open 상태 전환 조건

### Chapter 7: 컨테이너와 실전 네트워크 (4개 문서)
- Docker 네트워킹 — bridge/host/overlay 네트워크 모드, veth 인터페이스, iptables 규칙
- Kubernetes 네트워킹 — Pod IP 할당, kube-proxy, Service ClusterIP/NodePort/LoadBalancer
- 운영에서 만나는 네트워크 문제 패턴 — Connection Reset, Timeout vs Connection Refused, 포트 고갈
- 네트워크 성능 측정 — RTT, 처리량, 지연시간 측정 방법, iperf3, ss로 소켓 상태 분석

---

각 챕터는 **4~7개 문서**로 구성해줘. 총 **37개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 실무에서 중요한가
## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)
## ✨ 올바른 접근 (After — 원리를 알고 난 설계/운영)
## 🔬 내부 동작 원리 (패킷 레벨, 프로토콜 내부 분석)
## 💻 실전 실험 (tcpdump 캡처, Wireshark 분석, 재현 시나리오)
## 📊 성능/비용 비교 (HTTP/1.1 vs HTTP/2, TLS 버전별 RTT 등)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. 추상적 설명 금지 — 모든 원리는 "실제 패킷이 어떻게 생겼는가"로 연결
2. tcpdump/Wireshark 캡처 결과를 직접 분석하는 실험 포함
3. 각 프로토콜의 설계 이유 ("왜 이렇게 만들었는가") 명시
4. Spring 애플리케이션 관점 연결 — KeepAlive 설정, 커넥션 풀, SSL 설정이 TCP/TLS와 어떻게 연결되는지
5. 운영 장애 시나리오와 진단 명령어 세트

**실험 환경**:
```yaml
# docker-compose.yml
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./certs:/etc/nginx/certs

  app:
    image: eclipse-temurin:21-jre
    depends_on:
      - nginx

  wireshark:
    image: linuxserver/wireshark
    cap_add:
      - NET_ADMIN
    network_mode: host
    ports:
      - "3000:3000"
```

```bash
# 실험용 공통 명령어
# TCP 핸드쉐이크 캡처
tcpdump -i any -w capture.pcap 'tcp port 80'

# TLS 핸드쉐이크 분석
openssl s_client -connect example.com:443 -tls1_3 -msg

# HTTP/2 확인
curl -v --http2 https://example.com 2>&1 | grep -E "HTTP|TLS"

# TCP 소켓 상태
ss -tanp | grep ESTABLISHED
netstat -an | grep TIME_WAIT | wc -l
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**:
   - database-internals/mysql-deep-dive README 스타일 참고
   - "패킷 레벨에서 네트워크를 이해한다"는 차별화 강조
   - Spring/MySQL/Docker와의 연결 지점 명시
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료 섹션

README 작성 시 다음을 Reference로 포함해줘:
- Computer Networks: A Systems Approach (Peterson & Davie)
- TCP/IP Illustrated, Volume 1 (W. Richard Stevens)
- High Performance Browser Networking (Ilya Grigorik) — https://hpbn.co
- RFC 793 (TCP), RFC 7540 (HTTP/2), RFC 9000 (QUIC/HTTP/3)
- Cloudflare Blog — https://blog.cloudflare.com

---

## 💡 핵심 분석 대상

```
HTTP 요청 전체 흐름:

애플리케이션 (Spring RestTemplate / WebClient)
  │
  ▼ DNS 조회 (Resolver → Root NS → TLD NS → Authoritative NS)
  ▼
IP 주소 확인
  │
  ▼ TCP 연결 수립 (3-Way Handshake)
  │  SYN → SYN-ACK → ACK (1.5 RTT)
  ▼
TLS 핸드쉐이크 (HTTPS인 경우)
  │  TLS 1.3: 1 RTT (ClientHello → ServerHello+Certificate+Finished → Finished)
  │  TLS 1.2: 2 RTT
  ▼
HTTP 요청 전송
  │  HTTP/1.1: 직렬 요청 (HOL Blocking)
  │  HTTP/2: 멀티플렉싱 (병렬 스트림)
  │  HTTP/3: QUIC (UDP 기반, 0-RTT 가능)
  ▼
응답 수신
  │
  ▼ TCP 연결 종료 또는 Keep-Alive 유지
    FIN → FIN-ACK → FIN → ACK → TIME_WAIT (2MSL)

장애 진단 흐름:
  증상: 특정 API 응답 없음
  ├── ping → ICMP 도달 여부
  ├── telnet host 443 → TCP 포트 오픈 여부
  ├── curl -v → TLS 핸드쉐이크 성공 여부
  ├── tcpdump → 패킷 수준 RST/FIN 확인
  └── ss -tanp → 소켓 상태 (CLOSE_WAIT 누적 등)

TIME_WAIT 현상:
  서버 재시작 후 → 같은 포트로 재연결 실패
  원인: 이전 연결의 TIME_WAIT 상태 (2MSL = 120초)
  잘못된 해결: SO_REUSEADDR 무분별 사용
  올바른 이해: TIME_WAIT은 지연 패킷으로부터 새 연결을 보호
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (4~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (37개 목표)
- tcpdump/Wireshark 실험 환경
- Spring/MySQL/Docker와 연결되는 지점 명시 (어느 문서에서 어느 레포와 연결하는지)

**준비됐으면 1단계 구조 설계부터 시작해줘!**
