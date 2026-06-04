# gRPC + Protocol Buffers Deep Dive 레포지토리 제작 프롬프트

나는 "gRPC + Protocol Buffers Deep Dive" 레포지토리를 만들려고 해.
서비스 간 REST API를 호출하는 것과, Protocol Buffers가 어떻게 데이터를 바이너리로 직렬화하고 HTTP/2 멀티플렉싱으로 연결을 재사용하며, 서버/클라이언트 스트리밍이 어떻게 동작하는지를 완전히 파헤치는 레포야.

"JSON으로 REST 호출하면 되는 거 아닌가"와 "마이크로서비스 간 통신에서 JSON 직렬화 비용, HTTP/1.1 Connection 오버헤드, API 스키마 계약 없는 개발이 어떤 문제를 만드는지 알고 gRPC를 선택하거나 거부하는 것"의 차이를 만드는 레포야.

## 📋 프로젝트 목표

**컨셉**: "REST로 서비스를 연결하는 것과, 타입 안전한 계약으로 서비스를 연결하고 통신 비용을 줄이는 것은 다르다"

**핵심 차별화**:
1. Protocol Buffers 직렬화 — JSON 대비 크기와 속도가 왜 차이나는지 내부 인코딩 분석
2. HTTP/2 멀티플렉싱 — 하나의 연결로 여러 요청을 처리하는 원리, HTTP/1.1과의 차이
3. 스트리밍 패턴 — 서버/클라이언트/양방향 스트리밍이 적합한 시나리오
4. 서비스 계약 — `.proto` 파일이 API 계약서가 되어 Breaking Change를 방지하는 원리

**타겟 독자**:
- MSA 환경에서 서비스 간 REST API를 쓰지만 스키마 계약이 없어 Breaking Change가 발생하는 개발자
- gRPC를 들어봤지만 "REST 대신 굳이?"라고 생각하는 개발자
- Protobuf를 써봤지만 필드 번호가 왜 중요한지 모르는 개발자
- 실시간 데이터 스트리밍을 Polling으로 구현하는 개발자
- MSA Deep Dive를 공부했지만 gRPC 통신이 내부적으로 어떻게 동작하는지 모르는 개발자

**선행 학습**:
- network-deep-dive (HTTP/2 멀티플렉싱, TLS 핸드쉐이크 이해 필수)
- msa-deep-dive (서비스 간 통신 패턴, gRPC vs REST 선택 기준 맥락)
- spring-webflux-deep-dive (gRPC의 비동기 스트리밍과 Reactive Streams 연결, 시너지 큼)

---

## 🎯 1단계: 전체 구조 설계

다음 7개 챕터로 레포지토리를 설계해줘:

### Chapter 1: gRPC 설계 철학과 아키텍처 (5개 문서)
- REST vs gRPC — 왜 REST가 마이크로서비스에서 한계를 보이는가(느슨한 스키마, JSON 비용, HTTP/1.1 제약), gRPC가 해결하는 문제와 도입 비용
- gRPC 핵심 구성 요소 — `.proto` 파일 → 코드 생성 → Stub → Channel → Server 전체 구조, `protoc` 컴파일러와 플러그인 동작 방식
- HTTP/2 기반 통신 — Frame/Stream/Message 계층 구조, 하나의 TCP 연결에서 여러 RPC를 동시에 처리하는 멀티플렉싱, HTTP/1.1 Head-of-Line Blocking과의 차이
- gRPC 통신 4가지 패턴 — Unary/Server Streaming/Client Streaming/Bidirectional Streaming 각각의 적합한 사용 시나리오
- gRPC 생태계 — gRPC-Web(브라우저 지원), gRPC-Gateway(REST 변환), Envoy Proxy의 gRPC 지원, gRPC Reflection

### Chapter 2: Protocol Buffers 완전 분해 (7개 문서)
- Protobuf 직렬화 원리 — 필드 번호 + Wire Type으로 구성된 Tag-Length-Value 인코딩, Varint 압축(작은 숫자는 더 작게), JSON 대비 크기 비교
- 필드 번호가 API의 본체인 이유 — 필드 이름은 컴파일 후 사라지고 번호만 남는 원리, 필드 번호 재사용이 데이터 손상을 일으키는 시나리오
- 스칼라 타입과 기본값 — int32/int64/string/bool/bytes의 직렬화 방식, 필드가 없을 때 기본값이 전송되지 않는 원리, optional vs required 폐지 이유
- 복합 타입 — message 중첩, repeated 배열 인코딩, map 타입의 내부 구현(repeated message로 변환), oneof로 유니온 타입 표현
- Well-Known Types — `google.protobuf.Timestamp`/`Duration`/`Any`/`Struct`의 사용 목적, JSON과의 변환 규칙
- Protobuf 진화 규칙 — Backward/Forward Compatibility 보장 조건, 필드 삭제 시 `reserved` 키워드 사용, 신규 필드 추가가 안전한 이유
- 직렬화 성능 측정 — JSON vs Protobuf vs MessagePack 직렬화 크기/속도 벤치마크, 네트워크 비용 절감 효과 수치화

### Chapter 3: gRPC 서비스 설계 (6개 문서)
- .proto 파일 설계 원칙 — 서비스 정의 명명 규칙, 요청/응답 메시지 래핑 패턴(Request/Response Wrapper), 버전 관리 전략(`v1`/`v2` 패키지)
- 에러 처리 — gRPC Status Code 13가지 의미, HTTP 상태 코드와의 매핑, 상세 에러 정보를 `google.rpc.Status`로 전달하는 방법
- 메타데이터(Metadata) — HTTP 헤더에 해당하는 gRPC 메타데이터, 인증 토큰/요청 ID/트레이스 ID 전달, Interceptor에서 메타데이터 추출
- 데드라인과 타임아웃 — Deadline이 호출 체인을 따라 전파되는 원리(Context Propagation), 타임아웃 없는 gRPC 호출의 위험성, Deadline Exceeded 에러 처리
- 로드밸런싱 전략 — gRPC가 HTTP/2 기반이라 L4 로드밸런서가 효과 없는 이유, 클라이언트 사이드 로드밸런싱, Envoy Proxy로 해결하는 방법
- 서비스 계약 관리 — Buf Schema Registry로 중앙화된 `.proto` 관리, Breaking Change 감지(`buf breaking`), Consumer-Driven Contract Testing

### Chapter 4: gRPC 스트리밍 패턴 (5개 문서)
- Server Streaming — 서버가 단일 요청에 여러 응답을 스트리밍하는 원리, 실시간 주식 시세/로그 스트리밍 구현, Backpressure 처리
- Client Streaming — 클라이언트가 여러 메시지를 보내고 단일 응답을 받는 패턴, 파일 업로드/배치 데이터 전송 구현
- Bidirectional Streaming — 양방향 동시 스트리밍, 채팅 서비스/게임 상태 동기화 구현, 스트림 생명주기 관리
- 스트리밍 흐름 제어 — HTTP/2 Flow Control이 수신 측을 보호하는 원리, Window Size 설정, 스트림 버퍼링 전략
- 스트리밍 에러 처리 — 스트림 중간에 에러 발생 시 처리, 재연결 전략, 부분 실패 복구 패턴

### Chapter 5: gRPC 보안과 인증 (5개 문서)
- TLS와 mTLS — gRPC에서 TLS가 필수인 이유, 서버 인증서 검증, mTLS로 클라이언트 신원 검증(서비스 간 Zero Trust)
- JWT 기반 인증 — Interceptor(ClientInterceptor/ServerInterceptor)로 JWT 토큰 주입/검증, 메타데이터로 Authorization 헤더 전달
- API 키와 서비스 간 인증 — 내부 서비스 간 인증에 적합한 방식, 인증 정보를 Context에 전파하는 패턴
- Interceptor 체인 — ClientInterceptor/ServerInterceptor의 역할, 로깅/인증/분산 추적/재시도를 Interceptor로 구현, gRPC 미들웨어 패턴
- 채널 보안 설정 — `ManagedChannelBuilder` 보안 옵션, 개발 환경 plaintext vs 프로덕션 TLS, 인증서 갱신 전략

### Chapter 6: Spring Boot + gRPC 통합 (5개 문서)
- grpc-spring-boot-starter 설정 — 서버/클라이언트 자동 설정, `@GrpcService`로 서비스 구현, `@GrpcClient`로 Stub 주입, 포트 설정
- Spring Security 통합 — gRPC 서버에 Spring Security 적용, JWT 인증 Interceptor 구현, 메서드 레벨 권한 설정
- 예외 처리 통합 — `GrpcExceptionAdvice`로 Spring 예외를 gRPC Status로 변환, 전역 예외 핸들러 구현
- gRPC + Spring WebFlux — Reactive Stub으로 비동기 스트리밍 처리, Mono/Flux로 gRPC 스트림 래핑, Backpressure 연동
- 테스트 전략 — `GrpcServerExtension`으로 단위 테스트, Testcontainers로 통합 테스트, Mock Stub 구현

### Chapter 7: 운영과 성능 튜닝 (5개 문서)
- gRPC 모니터링 — Micrometer + Prometheus로 gRPC 메트릭 수집(요청 수, 에러율, 응답시간), Grafana 대시보드 구성
- 분산 추적 — OpenTelemetry gRPC Instrumentation으로 서비스 간 Trace 전파, Baggage로 컨텍스트 전달
- 연결 관리 튜닝 — `keepAliveTime`/`keepAliveTimeout`으로 유휴 연결 유지, 최대 연결 수 설정, Channel Pool 구성
- gRPC vs REST 성능 비교 — 동일 비즈니스 로직에서 직렬화 크기/시간, 처리량(TPS), 지연시간(Latency) 수치 비교
- 마이그레이션 전략 — 기존 REST API를 gRPC로 점진적 전환, gRPC-Gateway로 REST/gRPC 동시 지원, 클라이언트 하위 호환성 유지

---

각 챕터는 **5~7개 문서**로 구성해줘. 총 **38개 문서** 목표.

**문서 구조**:
```markdown
## 🎯 핵심 질문
## 🔍 왜 이 개념이 실무에서 중요한가
## 😱 흔한 실수 (Before — REST로만 생각하는 접근)
## ✨ 올바른 접근 (After — gRPC 원리를 알고 선택하는 접근)
## 🔬 내부 동작 원리 (Protobuf 인코딩, HTTP/2 Frame 수준 분석)
## 💻 실전 실험 (grpcurl, Wireshark HTTP/2 캡처, 벤치마크)
## 📊 성능/비용 비교 (JSON vs Protobuf 크기, REST vs gRPC 처리량)
## ⚖️ 트레이드오프
## 📌 핵심 정리
## 🤔 생각해볼 문제 (+ 해설)
```

**스타일 가이드**:
1. REST와 항상 대비 — "REST로 하면 어떻게 되는가"를 먼저 보여주고 gRPC 접근 제시
2. Protobuf 인코딩은 실제 바이트를 16진수로 보여주며 설명
3. 각 설계 결정의 이유 명시 ("왜 필드 이름이 아닌 번호로 직렬화하는가")
4. `grpcurl`로 바로 재현 가능한 실험 포함
5. MSA Deep Dive, network-deep-dive와 연결 — 서비스 간 통신 맥락에서 설명

**실험 환경**:
```yaml
# docker-compose.yml
services:
  grpc-server:
    build:
      context: ./grpc-server
    ports:
      - "9090:9090"   # gRPC
      - "8080:8080"   # gRPC-Gateway (REST)
    environment:
      SPRING_PROFILES_ACTIVE: docker

  grpc-client:
    build:
      context: ./grpc-client
    depends_on:
      - grpc-server
    ports:
      - "8081:8081"

  envoy:
    image: envoyproxy/envoy:v1.28-latest
    volumes:
      - ./envoy.yaml:/etc/envoy/envoy.yaml
    ports:
      - "9901:9901"  # Admin
      - "10000:10000" # gRPC 프록시

  wireshark:
    image: linuxserver/wireshark:latest
    network_mode: host
    cap_add:
      - NET_ADMIN
```

```protobuf
// order_service.proto — 완전한 서비스 정의 예시
syntax = "proto3";
package order.v1;

import "google/protobuf/timestamp.proto";
import "google/rpc/status.proto";

option java_package = "com.example.order.v1";
option java_multiple_files = true;

service OrderService {
  // Unary: 주문 생성
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);

  // Server Streaming: 주문 상태 실시간 구독
  rpc WatchOrderStatus(WatchOrderRequest) returns (stream OrderStatusEvent);

  // Client Streaming: 대량 상품 업로드
  rpc BulkUploadProducts(stream ProductUploadRequest) returns (BulkUploadResponse);

  // Bidirectional: 채팅/실시간 협업
  rpc StreamOrderUpdates(stream OrderUpdateRequest) returns (stream OrderUpdateResponse);
}

message CreateOrderRequest {
  string user_id = 1;
  repeated OrderItem items = 2;
  string shipping_address = 3;
}

message OrderItem {
  string product_id = 1;
  int32 quantity = 2;
  int64 price_cents = 3;
}

message CreateOrderResponse {
  string order_id = 1;
  google.protobuf.Timestamp created_at = 2;
  OrderStatus status = 3;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED = 0;
  ORDER_STATUS_PENDING = 1;
  ORDER_STATUS_CONFIRMED = 2;
  ORDER_STATUS_SHIPPED = 3;
}
```

```bash
# 실험 명령어 세트

# grpcurl로 gRPC 서비스 호출
grpcurl -plaintext localhost:9090 list
grpcurl -plaintext localhost:9090 order.v1.OrderService/CreateOrder \
  -d '{"user_id": "user-1", "items": [{"product_id": "prod-1", "quantity": 2}]}'

# Protobuf 직렬화 크기 비교
echo '{"userId":"user-1","items":[{"productId":"prod-1","quantity":2}]}' | wc -c
# vs Protobuf 인코딩 바이트 수

# gRPC 서버 리플렉션으로 스키마 조회
grpcurl -plaintext localhost:9090 describe order.v1.OrderService

# 스트리밍 호출
grpcurl -plaintext localhost:9090 order.v1.OrderService/WatchOrderStatus \
  -d '{"order_id": "order-123"}'

# Wireshark HTTP/2 Frame 캡처
# Filter: http2 and tcp.port==9090
```

---

## 🎯 2단계: 작업 순서

전체 구조를 설계한 후:

1. **디렉토리 생성**: bash 명령어로 모든 챕터 폴더 생성
2. **README.md 작성**:
   - msa-deep-dive README 스타일 참고
   - "REST로 서비스를 연결하는 것과 타입 안전한 계약으로 연결하는 것의 차이"라는 포지셔닝 강조
   - network-deep-dive, msa-deep-dive, spring-webflux-deep-dive와의 연결 지점 명시
3. **챕터별 문서 작성**:
   - Chapter 1부터 순서대로
   - 각 문서는 2000~3000 단어 분량

---

## 📚 참고 자료

- gRPC 공식 문서 — https://grpc.io/docs/
- Protocol Buffers 공식 문서 — https://protobuf.dev/
- gRPC: Up and Running (Kasun Indrasiri, Danesh Kuruppu)
- HTTP/2 스펙 — https://www.rfc-editor.org/rfc/rfc9113
- Buf Schema Registry — https://buf.build/
- grpc-spring-boot-starter — https://github.com/grpc-ecosystem/grpc-spring

---

## 💡 핵심 분석 대상

```
Protobuf 직렬화 내부:

message Person {
  string name = 1;   // 필드 번호 1
  int32 age = 2;     // 필드 번호 2
}

Person { name: "kim", age: 25 }

직렬화 결과 (hex):
0a 03 6b 69 6d 10 19
│  │  └─── "kim" UTF-8 바이트
│  └────── 길이 = 3
└───────── Tag = (필드번호 1 << 3) | WireType 2(LEN) = 0x0a
           Tag = (필드번호 2 << 3) | WireType 0(VARINT) = 0x10
           0x19 = Varint(25)

JSON 비교:
{"name":"kim","age":25}  → 22 바이트
Protobuf:                → 7 바이트 (68% 절감)

HTTP/2 멀티플렉싱 vs HTTP/1.1:

HTTP/1.1:
  Connection 1: [요청A] → 응답 대기 → [응답A]
  Connection 2: [요청B] → 응답 대기 → [응답B]
  Connection 3: [요청C] → 응답 대기 → [응답C]
  → 동시 처리 위해 연결 여러 개 필요

HTTP/2 (gRPC):
  Connection 1:
    Stream 1: [요청A] ──────────────── [응답A]
    Stream 3:     [요청B] ──── [응답B]
    Stream 5:         [요청C] ────────── [응답C]
  → 하나의 연결에서 병렬 처리

Breaking Change 방지:

  ✗ 위험한 변경 (기존 클라이언트 깨짐):
    필드 번호 재사용: 기존 string name = 1 → int32 id = 1
    필드 번호 삭제: 번호 2를 reserved 없이 삭제 후 재사용

  ✓ 안전한 변경:
    새 필드 추가: string email = 3 (새 번호)
    필드 삭제: reserved 1, 2; reserved "name", "age";
    → 구 클라이언트: 모르는 필드 무시 (Forward Compatible)
    → 신 클라이언트: 없는 필드 기본값 사용 (Backward Compatible)
```

---

## 🎯 진행 방식

**1단계부터 시작해줘. 전체 구조를 먼저 상세히 설계해줘.**

구조 설계 시 다음을 포함해줘:
- 각 챕터별 문서 제목 (5~7개씩)
- 각 문서가 다루는 핵심 내용 (3~4줄)
- 전체 문서 개수 확인 (38개 목표)
- Docker Compose 실험 환경 (gRPC 서버 + Envoy + Wireshark)
- network-deep-dive, msa-deep-dive, spring-webflux-deep-dive와 연결되는 지점 명시

**준비됐으면 1단계 구조 설계부터 시작해줘!**
