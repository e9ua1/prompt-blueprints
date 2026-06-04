# Real-time & Client Networking Deep Dive 레포지토리 제작 프롬프트

나는 "Real-time & Client Networking Deep Dive" 레포지토리를 만들려고 해.
실시간 통신의 옵션들 — 폴링·SSE·WebSocket·WebRTC가 *내부에서 어떻게 동작*하고, 연결을 어떻게 신뢰성 있게 유지하며, 실시간 동기화를 어떻게 다루는지를 완전히 파헤치는 레포야.
"Socket.io를 붙이는 것"과 "각 실시간 기술이 어느 계층에서 어떻게 동작하고 무엇을 트레이드오프하는지 알고 선택하는 것"의 차이를 만드는 레포다.

## 📋 프로젝트 목표

**컨셉**: "실시간 라이브러리를 붙이는 것과, WebSocket·SSE·WebRTC가 내부에서 어떻게 동작하는지 아는 것은 다르다"

**핵심 차별화**:
1. 실시간 스펙트럼 — 폴링→롱폴링→SSE→WebSocket→WebRTC, 각자의 적합 시나리오
2. 프로토콜 내부 — WS 핸드셰이크·프레임, WebRTC의 ICE/STUN/TURN/SRTP
3. 연결 신뢰성 — 재연결·하트비트·백프레셔·메시지 순서 보장
4. 실시간 동기화 — 동시 편집·상태 일관성(OT/CRDT, distributed 연결)

**타겟 독자**:
- Socket.io를 쓰지만 WebSocket 프로토콜을 모르는 개발자
- SSE와 WebSocket 중 뭘 쓸지 기준이 없는 개발자
- WebRTC를 "화상통화"로만 알고 P2P 연결 수립을 모르는 개발자
- 실시간 연결이 끊길 때 재연결을 제대로 못 짜는 개발자
- `network-deep-dive`를 마치고 클라이언트 실시간을 파려는 개발자

## 🔗 레포 연결

**⬆️ 선행**: `network-deep-dive`(TCP·HTTP·TLS), `event-loop-async-deep-dive`(비동기 메시지 처리).
**🤝 시너지**: `web-apis-wasm-deep-dive`(스트림·Worker), `distributed-systems-theory-deep-dive`(동기화·일관성·CRDT), `kafka-deep-dive`(백엔드 스트리밍 대조).
**🧬 수렴**: `local-first-sync-deep-dive`(CRDT 동기화 — 모바일과 공유), `concurrency-models-compared`.

---

## 🎯 1단계: 전체 구조 설계

### Chapter 1: 실시간 통신 스펙트럼 (5개 문서)
- "실시간"의 종류 — 서버 푸시 vs 양방향 vs P2P, 지연·방향성 축
- 폴링과 롱폴링 — 주기적 요청·서버 보류, HTTP만으로 실시간 흉내, 비용
- 기술 선택 매트릭스 — SSE vs WS vs WebRTC를 방향·페이로드·인프라로 분류
- 계층 이해 — 각 기술이 HTTP/TCP/UDP의 어디에 서는가(network 연결)
- 실시간의 어려움 — 연결 유지·순서·중복·장애가 만드는 문제

### Chapter 2: WebSocket (5개 문서)
- 핸드셰이크 — HTTP Upgrade로 시작, 101 전환, 프로토콜 협상
- 프레임 구조 — opcode·마스킹·페이로드, 텍스트/바이너리, 제어 프레임
- 연결 생명주기 — open/message/close/error, ping/pong 하트비트
- 메시지 패턴 — 발행/구독·룸·브로드캐스트 구현
- 확장과 한계 — 압축(permessage-deflate), 프록시·로드밸런서 이슈, 스케일아웃

### Chapter 3: SSE와 롱폴링 (5개 문서)
- SSE 동작 — text/event-stream, 단방향 서버→클라 푸시, 자동 재연결
- SSE 포맷 — event/data/id/retry 필드, Last-Event-ID로 재개
- SSE vs WebSocket — 단방향·HTTP 친화 vs 양방향, 언제 SSE로 충분한가
- HTTP/2와 SSE — 멀티플렉싱으로 연결 수 제한 완화
- 롱폴링 폴백 — WS 불가 환경 대응, 트레이드오프

### Chapter 4: WebRTC (5개 문서)
- WebRTC 개요 — 브라우저 간 P2P, 미디어 + 데이터 채널
- 시그널링 — SDP 교환(별도 채널 필요), offer/answer
- NAT 통과 — ICE 후보 수집, STUN(공인 IP 발견)·TURN(릴레이) 폴백
- 데이터 채널 — SCTP 기반, 신뢰/비신뢰·순서/비순서 모드 선택
- 보안·미디어 — DTLS/SRTP 암호화, 미디어 파이프라인 개요

### Chapter 5: 연결 신뢰성 (5개 문서)
- 재연결 전략 — 지수 백오프·지터, 끊김 감지, 상태 복원
- 하트비트 — ping/pong으로 죽은 연결 감지(좀비 연결 문제)
- 메시지 신뢰성 — ack·재전송·중복 제거·순서 보장(at-least/exactly-once)
- 백프레셔 — 빠른 생산·느린 소비, 버퍼 관리, 흐름 제어
- 오프라인·복원 — 끊김 동안 큐잉, 재연결 시 동기화

### Chapter 6: 실시간 동기화 (5개 문서)
- 동기화 문제 — 동시 편집·상태 충돌, 단일 진실 원천의 부재
- OT(Operational Transformation) — 연산 변환으로 충돌 해결, 구현 난이도
- CRDT — 충돌 없는 수렴(distributed 레포 연결), 협업 편집
- Presence·인지 — 누가 접속·어디 보는지, 커서 공유
- 일관성 모델 — 최종 일관성 수용, 낙관적 업데이트·롤백

### Chapter 7: 운영과 측정 (5개 문서)
- 스케일 — 다중 서버 간 메시지 전파(Redis Pub/Sub·백엔드 브로커)
- 모니터링 — 연결 수·지연·드롭, WS 프레임 추적
- 디버깅 — DevTools WS 프레임·chrome://webrtc-internals
- 보안 — 인증(연결 시 토큰)·rate limit·메시지 검증
- 종합 — 실시간 협업 데모(WS + 동기화)를 끝까지 구현·측정

→ **총 35개 문서** 목표.

## 📄 문서 구조

각 문서 10섹션 (표준 v2). `🔬 내부 동작 원리`는 프레임·핸드셰이크·ICE 흐름, `💻 실전 실험`은 WS/SSE/WebRTC 데모 + DevTools 프레임 캡처. `📊`는 기술별 지연·오버헤드 비교.

## 🎨 스타일 가이드

1. **계층으로 분류** — 각 기술이 어느 프로토콜 위에 서는지 명확히
2. **프레임을 까본다** — WS 프레임·SSE 스트림·SDP를 *직접 본다*
3. **선택 기준 제공** — 방향성·페이로드·인프라로 기술 결정
4. **distributed/network와 연결** — 동기화는 distributed, 전송은 network 레포로
5. 핸드셰이크·ICE·동기화는 시퀀스 다이어그램으로

## 🔬 검증 환경

> docker(작은 시그널링/WS 서버) + 브라우저 DevTools.

```yaml
# docker-compose.yml — 실시간 실습 (간단 서버)
services:
  ws-server:
    build: ./ws-server      # node ws 또는 SSE 서버
    ports: ["8080:8080"]
  signaling:
    build: ./signaling      # WebRTC 시그널링
    ports: ["8081:8081"]
  redis:
    image: redis:7          # 다중 서버 스케일아웃 데모
    ports: ["6379:6379"]
```

```bash
# 검증 방법
# 1) DevTools Network > WS: 프레임(보냄/받음) 실시간 확인, 마스킹·opcode
# 2) SSE: Network에서 EventStream 탭, 이벤트 스트림 관찰
# 3) WebRTC: chrome://webrtc-internals 에서 ICE 후보·연결 상태·통계
# 4) 재연결 실험: 서버 죽였다 살려 백오프 재연결 동작 관찰
# 5) 동기화: 두 탭에서 동시 편집 → 충돌 해결(CRDT/OT) 수렴 확인
# 6) 백프레셔: 빠른 송신·느린 수신 → 버퍼 증가 관찰
```

## 🎯 2단계: 작업 순서

1. **디렉토리 생성**: 7개 챕터 + `demos/`(WS·SSE·WebRTC·협업)
2. **README.md**: 🌐 Frontend 톤, "라이브러리를 붙인다 vs 프로토콜이 어떻게 도나" 포지셔닝, `🔗 레포 연결`(network·distributed·local-first)
3. **챕터별 문서**: Chapter 1부터, 문서당 2000~3000 단어

## 📚 참고 자료

- MDN WebSocket·SSE·WebRTC — https://developer.mozilla.org/en-US/docs/Web/API/WebSockets_API
- RFC 6455 (WebSocket), WHATWG SSE
- "WebRTC for the Curious" — https://webrtcforthecurious.com/
- *High Performance Browser Networking* (Ilya Grigorik) — 실시간 장
- Yjs(CRDT 협업) 문서 — https://docs.yjs.dev/

## 💡 핵심 분석 대상

```
실시간 스펙트럼:
  폴링      : 주기적 GET (지연·낭비)
  롱폴링    : 서버가 응답 보류 → 이벤트 시 응답 (HTTP 재사용)
  SSE       : 단방향 서버→클라 스트림 (HTTP, 자동 재연결)
  WebSocket : 양방향 풀듀플렉스 (HTTP Upgrade 후 TCP)
  WebRTC    : P2P (UDP, NAT 통과, 미디어/데이터)

WS 핸드셰이크:
  Client: GET /ws  Upgrade: websocket  Sec-WebSocket-Key: ...
  Server: 101 Switching Protocols  Sec-WebSocket-Accept: ...
  → 이후 같은 TCP 연결에서 프레임 양방향

WebRTC NAT 통과:
  A와 B 둘 다 NAT 뒤 → 직접 연결 불가
  STUN: "내 공인 IP가 뭐지?" 발견 → 후보 교환(시그널링)
  직접 연결 시도 → 실패 시 TURN(릴레이 서버 경유)

재연결 (지수 백오프):
  끊김 → 1s 후 시도 → 실패 → 2s → 4s → 8s (+지터)
  하트비트(ping/pong)로 좀비 연결 조기 감지

동기화 (CRDT):
  두 사용자 동시 편집 → 각자 로컬 적용(낙관적)
  → 연산 교환 → CRDT 병합 규칙으로 항상 같은 결과 수렴
```

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**
챕터별 문서 제목 + 핵심 내용(3~4줄) + 총 35개 확인 + WS서버/DevTools/webrtc-internals 검증 환경 + network/distributed/local-first 연결 지점 명시.

**준비됐으면 1단계 구조 설계부터 시작해줘!**
