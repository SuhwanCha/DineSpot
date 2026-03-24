# 공용 문서 — Proto 스키마 + Docker Compose + 서비스 계약

> 이 문서는 A~E 전원이 공유하는 **계약 문서**입니다.
> 인터페이스(proto, 이벤트, Redis 키)를 변경하려면 반드시 영향받는 담당자 전원 승인이 필요합니다.

---

## 1. 전체 서비스 포트 맵

| 서비스 | 컨테이너명 | gRPC 포트 | 담당 |
|--------|-----------|-----------|------|
| user-service | dinespot-user-service | 50051 | A |
| restaurant-service | dinespot-restaurant-service | 50052 | B |
| reservation-service | dinespot-reservation-service | 50053 | C |
| waiting-service | dinespot-waiting-service | 50054 | D |
| menu-service | dinespot-menu-service | 50055 | B |
| notification-service | dinespot-notification-service | 50056 | D |
| admin-service | dinespot-admin-service | 50057 | E |
| envoy (gRPC-Web GW) | dinespot-envoy | 8080 (public) | 공용 |
| frontend (Next.js) | dinespot-frontend | 3000 | 공용 |
| mysql | dinespot-mysql | 3306 | 공용 |
| redis | dinespot-redis | 6379 | 공용 |
| prometheus | dinespot-prometheus | 9090 | E |
| grafana | dinespot-grafana | 3001 | E |
| mcp-server (dev only) | dinespot-mcp-server | 50060 | 공용 |

---

## 2. 서비스 의존성 그래프

```
mysql ──────────────────────────────────────────────────────────┐
redis ──────────────────────────────────────────┐               │
                                                │               │
user-service ──────────────────────────────────────────────────>│
                        │                       │               │
                        ▼                       │               │
restaurant-service ─────────────────────────────┤               │
    (calls: user-service)                       │               │
                        │                       │               │
                        ▼                       │               │
notification-service ────────────────────────────┤               │
    (calls: --)         │                       │               │
                        ▼                       │               │
reservation-service ─────────────────────────────┤               │
    (calls: user-service, restaurant-service,   │               │
            notification-service)               │               │
                        │                       │               │
                        ▼                       │               │
waiting-service ─────────────────────────────────┤               │
    (calls: notification-service)               │               │
                        │                       │               │
                        ▼                       │               │
menu-service ────────────────────────────────────┤               │
    (calls: restaurant-service,                 │               │
            notification-service)               │               │
                                                │               │
admin-service ───────────────────────────────────┘               │
    (calls: --)                                                 │
                                                                │
envoy ──────────────────── (routes all gRPC-Web) ───────────────┘
frontend ────── (calls envoy) ──────────────────────────────────┘
```

---

## 3. docker-compose.yml (전체)

파일 위치: `docker-compose.yml` (프로젝트 루트)

```yaml
version: "3.9"
name: dinespot

# ─────────────────────────────────────────────────────────────
# 공통 재사용 설정
# ─────────────────────────────────────────────────────────────
x-backend-common: &backend-common
  build:
    context: ./backend
    dockerfile: Dockerfile
    target: service
  restart: unless-stopped
  networks: [dinespot-net]

x-backend-env: &backend-env
  DB_DSN: "dinespot:dinespot_pass@tcp(mysql:3306)/dinespot?parseTime=true&loc=Asia%2FSeoul&charset=utf8mb4"
  REDIS_ADDR: "redis:6379"
  LOG_LEVEL: "info"
  LOG_FORMAT: "json"
  OTEL_EXPORTER_OTLP_ENDPOINT: ""   # 로컬에서는 비활성

services:
  # ═══════════════════════════════════════════════════════════
  # Infrastructure
  # ═══════════════════════════════════════════════════════════

  mysql:
    image: mysql:8.0
    container_name: dinespot-mysql
    restart: unless-stopped
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: dinespot_root
      MYSQL_DATABASE: dinespot
      MYSQL_USER: dinespot
      MYSQL_PASSWORD: dinespot_pass
      MYSQL_CHARSET: utf8mb4
      MYSQL_COLLATION: utf8mb4_unicode_ci
    command:
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
      - --default-time-zone=+09:00
      - --innodb-buffer-pool-size=256M
    volumes:
      - mysql_data:/var/lib/mysql
      # 초기 DDL 파일들 (알파벳 순서로 실행됨)
      - ./infra/mysql/init:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1", "-u", "dinespot", "-pdinespot_pass"]
      interval: 10s
      timeout: 5s
      retries: 10
      start_period: 30s
    networks: [dinespot-net]

  redis:
    image: redis:7-alpine
    container_name: dinespot-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    command: >
      redis-server
      --appendonly yes
      --appendfsync everysec
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --save 60 1
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 10
    networks: [dinespot-net]

  # ═══════════════════════════════════════════════════════════
  # Backend gRPC Services
  # ═══════════════════════════════════════════════════════════

  user-service:
    <<: *backend-common
    container_name: dinespot-user-service
    command: ["/app/server", "--service=user", "--port=50051"]
    ports:
      - "50051:50051"
    environment:
      <<: *backend-env
      SERVICE_NAME: user-service
      GRPC_PORT: "50051"
      METRICS_PORT: "9100"
      JWT_SECRET: "${JWT_SECRET:-dev_jwt_secret_CHANGE_IN_PROD}"
      JWT_ACCESS_TTL: "15m"
      JWT_REFRESH_TTL: "168h"   # 7일
      BCRYPT_COST: "12"
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy

  restaurant-service:
    <<: *backend-common
    container_name: dinespot-restaurant-service
    command: ["/app/server", "--service=restaurant", "--port=50052"]
    ports:
      - "50052:50052"
    environment:
      <<: *backend-env
      SERVICE_NAME: restaurant-service
      GRPC_PORT: "50052"
      METRICS_PORT: "9101"
      USER_SERVICE_ADDR: "user-service:50051"
      # S3/GCS 이미지 업로드 (로컬은 로컬 스토리지 사용)
      STORAGE_TYPE: "${STORAGE_TYPE:-local}"
      STORAGE_LOCAL_PATH: "/tmp/dinespot-images"
      S3_BUCKET: "${S3_BUCKET:-}"
      S3_REGION: "${S3_REGION:-ap-northeast-2}"
      GCS_BUCKET: "${GCS_BUCKET:-}"
    volumes:
      - restaurant_images:/tmp/dinespot-images
    depends_on:
      mysql:
        condition: service_healthy
      user-service:
        condition: service_started

  reservation-service:
    <<: *backend-common
    container_name: dinespot-reservation-service
    command: ["/app/server", "--service=reservation", "--port=50053"]
    ports:
      - "50053:50053"
    environment:
      <<: *backend-env
      SERVICE_NAME: reservation-service
      GRPC_PORT: "50053"
      METRICS_PORT: "9102"
      USER_SERVICE_ADDR: "user-service:50051"
      RESTAURANT_SERVICE_ADDR: "restaurant-service:50052"
      NOTIFICATION_SERVICE_ADDR: "notification-service:50056"
      RESERVATION_LOCK_TTL: "30s"
      NOSHOW_CHECK_INTERVAL: "5m"
      NOSHOW_GRACE_PERIOD: "60m"
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
      restaurant-service:
        condition: service_started
      notification-service:
        condition: service_started

  waiting-service:
    <<: *backend-common
    container_name: dinespot-waiting-service
    command: ["/app/server", "--service=waiting", "--port=50054"]
    ports:
      - "50054:50054"
    environment:
      <<: *backend-env
      SERVICE_NAME: waiting-service
      GRPC_PORT: "50054"
      METRICS_PORT: "9103"
      NOTIFICATION_SERVICE_ADDR: "notification-service:50056"
      NOSHOW_TTL: "10m"
      SSE_HEARTBEAT_INTERVAL: "15s"
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
      notification-service:
        condition: service_started

  menu-service:
    <<: *backend-common
    container_name: dinespot-menu-service
    command: ["/app/server", "--service=menu", "--port=50055"]
    ports:
      - "50055:50055"
    environment:
      <<: *backend-env
      SERVICE_NAME: menu-service
      GRPC_PORT: "50055"
      METRICS_PORT: "9104"
      RESTAURANT_SERVICE_ADDR: "restaurant-service:50052"
      NOTIFICATION_SERVICE_ADDR: "notification-service:50056"
      PAYMENT_GATEWAY_URL: "${PAYMENT_GATEWAY_URL:-http://mock-payment:8090}"
      PAYMENT_GATEWAY_SECRET: "${PAYMENT_GATEWAY_SECRET:-mock-secret}"
    depends_on:
      mysql:
        condition: service_healthy
      restaurant-service:
        condition: service_started

  notification-service:
    <<: *backend-common
    container_name: dinespot-notification-service
    command: ["/app/server", "--service=notification", "--port=50056"]
    ports:
      - "50056:50056"
    environment:
      <<: *backend-env
      SERVICE_NAME: notification-service
      GRPC_PORT: "50056"
      METRICS_PORT: "9105"
      KAKAO_API_KEY: "${KAKAO_API_KEY:-mock}"
      KAKAO_SENDER_KEY: "${KAKAO_SENDER_KEY:-mock}"
      SMS_API_KEY: "${SMS_API_KEY:-mock}"
      SMS_SENDER_NUMBER: "${SMS_SENDER_NUMBER:-01000000000}"
      EMAIL_SMTP_HOST: "${EMAIL_SMTP_HOST:-mailhog}"
      EMAIL_SMTP_PORT: "1025"
      RETRY_MAX_COUNT: "3"
      RETRY_BACKOFF_BASE: "2s"
      RETRY_BACKOFF_MAX: "30s"
      DEDUPE_TTL: "24h"
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy

  admin-service:
    <<: *backend-common
    container_name: dinespot-admin-service
    command: ["/app/server", "--service=admin", "--port=50057"]
    ports:
      - "50057:50057"
    environment:
      <<: *backend-env
      SERVICE_NAME: admin-service
      GRPC_PORT: "50057"
      METRICS_PORT: "9106"
      AUDIT_LOG_RETENTION_DAYS: "365"
    depends_on:
      mysql:
        condition: service_healthy

  # ═══════════════════════════════════════════════════════════
  # API Gateway
  # ═══════════════════════════════════════════════════════════

  envoy:
    image: envoyproxy/envoy:v1.28-latest
    container_name: dinespot-envoy
    restart: unless-stopped
    ports:
      - "8080:8080"     # gRPC-Web 퍼블릭 엔드포인트 (FE → Envoy)
      - "9901:9901"     # Envoy Admin UI
    volumes:
      - ./infra/envoy/envoy.yaml:/etc/envoy/envoy.yaml:ro
    depends_on:
      - user-service
      - restaurant-service
      - reservation-service
      - waiting-service
      - menu-service
      - notification-service
      - admin-service
    networks: [dinespot-net]

  # ═══════════════════════════════════════════════════════════
  # Frontend
  # ═══════════════════════════════════════════════════════════

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
      target: runner
    container_name: dinespot-frontend
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: development
      NEXT_PUBLIC_GRPC_WEB_URL: "http://localhost:8080"
      NEXT_PUBLIC_SSE_BASE_URL: "http://localhost:8080"
    depends_on:
      - envoy
    networks: [dinespot-net]

  # ═══════════════════════════════════════════════════════════
  # Observability
  # ═══════════════════════════════════════════════════════════

  prometheus:
    image: prom/prometheus:v2.50.1
    container_name: dinespot-prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.path=/prometheus
      - --storage.tsdb.retention.time=15d
      - --web.enable-lifecycle
    volumes:
      - ./infra/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./infra/prometheus/rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus
    networks: [dinespot-net]

  grafana:
    image: grafana/grafana:10.3.3
    container_name: dinespot-grafana
    restart: unless-stopped
    ports:
      - "3001:3000"     # 3000은 frontend가 사용하므로 3001로 노출
    environment:
      GF_SECURITY_ADMIN_USER: admin
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_AUTH_ANONYMOUS_ENABLED: "true"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./infra/grafana/provisioning:/etc/grafana/provisioning:ro
    depends_on:
      - prometheus
    networks: [dinespot-net]

  # ═══════════════════════════════════════════════════════════
  # Dev-only Tools (--profile dev 로 활성화)
  # ═══════════════════════════════════════════════════════════

  mcp-server:
    build:
      context: ./mcp-server
      dockerfile: Dockerfile
    container_name: dinespot-mcp-server
    ports:
      - "50060:50060"
    environment:
      ENVOY_ADDR: "envoy:8080"
      MCP_PORT: "50060"
    depends_on:
      - envoy
    profiles: ["dev"]
    networks: [dinespot-net]

  # 이메일 테스트용 로컬 SMTP (실제 발송 없이 UI로 확인)
  mailhog:
    image: mailhog/mailhog:latest
    container_name: dinespot-mailhog
    ports:
      - "1025:1025"     # SMTP
      - "8025:8025"     # Web UI
    profiles: ["dev"]
    networks: [dinespot-net]

  # 결제 PG 모킹 서버
  mock-payment:
    image: mockserver/mockserver:latest
    container_name: dinespot-mock-payment
    ports:
      - "8090:8090"
    environment:
      MOCKSERVER_SERVER_PORT: "8090"
    volumes:
      - ./infra/mockserver:/config:ro
    profiles: ["dev"]
    networks: [dinespot-net]

# ─────────────────────────────────────────────────────────────
networks:
  dinespot-net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

volumes:
  mysql_data:
  redis_data:
  prometheus_data:
  grafana_data:
  restaurant_images:
```

### 주요 실행 명령어

```bash
# 전체 인프라 + 서비스 시작
docker compose up -d

# dev 도구 포함 (mailhog, mock-payment, mcp-server)
docker compose --profile dev up -d

# 특정 서비스만 (예: user-service와 의존성만)
docker compose up -d mysql redis user-service

# 로그 실시간 확인
docker compose logs -f user-service

# 서비스 재빌드 후 재시작
docker compose up -d --build user-service

# 전체 종료 + 볼륨 삭제 (완전 초기화)
docker compose down -v
```

---

## 4. 프로젝트 디렉토리 전체 구조

```
restaurant-reservation/
│
├── docker-compose.yml                  ← 전체 서비스 정의 (이 문서)
├── .env.example                        ← 환경변수 템플릿
│
├── proto/                              ← 공용 proto 스키마 (이 문서 섹션 5)
│   ├── common/
│   │   └── types.proto
│   ├── user/
│   │   └── user.proto                  ← A 담당
│   ├── restaurant/
│   │   └── restaurant.proto            ← B 담당
│   ├── menu/
│   │   └── menu.proto                  ← B 담당
│   ├── reservation/
│   │   └── reservation.proto           ← C 담당
│   ├── waiting/
│   │   └── waiting.proto               ← D 담당
│   ├── notification/
│   │   └── notification.proto          ← D 담당
│   ├── admin/
│   │   └── admin.proto                 ← E 담당
│   └── buf.yaml                        ← buf 설정
│
├── backend/
│   ├── Dockerfile
│   ├── go.mod
│   ├── go.sum
│   ├── cmd/
│   │   └── server/
│   │       └── main.go                 ← --service 플래그로 서비스 선택
│   ├── internal/
│   │   ├── user/                       ← A 담당
│   │   │   ├── handler.go
│   │   │   ├── service.go
│   │   │   ├── repository.go
│   │   │   └── handler_test.go
│   │   ├── restaurant/                 ← B 담당
│   │   │   ├── handler.go
│   │   │   ├── service.go
│   │   │   ├── repository.go
│   │   │   └── handler_test.go
│   │   ├── menu/                       ← B 담당
│   │   │   ├── handler.go
│   │   │   ├── service.go
│   │   │   ├── repository.go
│   │   │   └── handler_test.go
│   │   ├── reservation/                ← C 담당
│   │   │   ├── handler.go
│   │   │   ├── service.go
│   │   │   ├── repository.go
│   │   │   ├── statemachine.go
│   │   │   └── handler_test.go
│   │   ├── waiting/                    ← D 담당
│   │   │   ├── handler.go
│   │   │   ├── service.go
│   │   │   ├── repository.go
│   │   │   ├── sse.go
│   │   │   └── handler_test.go
│   │   ├── notification/               ← D 담당
│   │   │   ├── handler.go
│   │   │   ├── service.go
│   │   │   ├── repository.go
│   │   │   ├── sender_kakao.go
│   │   │   ├── sender_sms.go
│   │   │   └── handler_test.go
│   │   └── admin/                      ← E 담당
│   │       ├── handler.go
│   │       ├── service.go
│   │       ├── repository.go
│   │       └── handler_test.go
│   └── pkg/
│       ├── middleware/
│       │   ├── auth.go                 ← A 담당 (JWT 인터셉터)
│       │   ├── logging.go              ← E 담당 (공통 로그 인터셉터)
│       │   └── metrics.go              ← E 담당 (공통 메트릭 인터셉터)
│       ├── redis/
│       │   ├── lock.go                 ← C 담당 (분산 락)
│       │   └── queue.go                ← D 담당 (웨이팅 큐)
│       ├── mysql/
│       │   └── db.go                   ← 공용 DB 연결 풀
│       └── errors/
│           └── codes.go                ← 공용 gRPC 에러 코드 매핑
│
├── frontend/
│   ├── Dockerfile
│   ├── package.json
│   └── src/
│       ├── app/
│       │   ├── (customer)/             ← 고객 화면
│       │   ├── (owner)/                ← 점주 대시보드
│       │   └── (admin)/                ← 슈퍼 어드민
│       ├── components/
│       └── generated/                  ← protobuf-es 자동 생성
│
├── infra/
│   ├── mysql/
│   │   └── init/                       ← 초기 DDL SQL 파일
│   │       ├── 01_users.sql            ← A 담당
│   │       ├── 02_restaurants.sql      ← B 담당
│   │       ├── 03_menus.sql            ← B 담당
│   │       ├── 04_reservations.sql     ← C 담당
│   │       ├── 05_waiting.sql          ← D 담당
│   │       ├── 06_notifications.sql    ← D 담당
│   │       └── 07_audit.sql            ← E 담당
│   ├── envoy/
│   │   └── envoy.yaml
│   ├── prometheus/
│   │   ├── prometheus.yml
│   │   └── rules/
│   │       └── dinespot.yml            ← E 담당
│   ├── grafana/
│   │   └── provisioning/
│   │       ├── datasources/
│   │       └── dashboards/             ← E 담당
│   └── mockserver/                     ← 결제 PG 모킹
│
└── mcp-server/
    ├── Dockerfile
    └── main.go
```

---

## 5. Proto 스키마 정의

> 생성 명령어: `buf generate` (프로젝트 루트에서 실행)
> 생성 결과: `backend/internal/gen/` 및 `frontend/src/generated/`

### 5-0. buf.yaml

```yaml
# proto/buf.yaml
version: v2
modules:
  - path: .
lint:
  use:
    - DEFAULT
breaking:
  use:
    - FILE
```

---

### 5-1. common/types.proto

```protobuf
syntax = "proto3";
package common.v1;
option go_package = "github.com/dinespot/backend/internal/gen/common/v1;commonv1";

import "google/protobuf/timestamp.proto";

// 페이지네이션 요청
message PageRequest {
  int32 page_size  = 1;   // 기본값 20, 최대 100
  string page_token = 2;  // 커서 기반 (빈 문자열 = 첫 페이지)
}

// 페이지네이션 응답
message PageResponse {
  string next_page_token = 1;  // 마지막 페이지면 빈 문자열
  int32  total_count     = 2;  // 전체 개수 (비싼 COUNT 쿼리, 선택적)
}

// 날짜 범위 필터
message DateRange {
  google.protobuf.Timestamp from = 1;
  google.protobuf.Timestamp to   = 2;
}

// 금액 (소수점 오차 방지를 위해 정수 단위 사용)
message Money {
  int64  amount   = 1;   // 원 단위 (예: 15000 = 15,000원)
  string currency = 2;   // "KRW"
}
```

---

### 5-2. user/user.proto (A 담당)

```protobuf
syntax = "proto3";
package user.v1;
option go_package = "github.com/dinespot/backend/internal/gen/user/v1;userv1";

import "google/protobuf/timestamp.proto";
import "common/types.proto";

// ─── Enums ───────────────────────────────────────────────────

enum UserRole {
  USER_ROLE_UNSPECIFIED = 0;
  USER_ROLE_CUSTOMER    = 1;
  USER_ROLE_OWNER       = 2;
  USER_ROLE_SUPER_ADMIN = 3;
}

// ─── Messages ────────────────────────────────────────────────

message User {
  string                    id             = 1;
  string                    email          = 2;
  string                    name           = 3;
  string                    phone          = 4;
  UserRole                  role           = 5;
  int32                     no_show_count  = 6;
  bool                      is_restricted  = 7;
  google.protobuf.Timestamp restricted_until = 8;
  bool                      notification_opt_out = 9;
  google.protobuf.Timestamp created_at     = 10;
  google.protobuf.Timestamp updated_at     = 11;
}

message RegisterRequest {
  string   email    = 1;  // 필수, 이메일 형식
  string   password = 2;  // 필수, 8자 이상
  string   name     = 3;  // 필수
  string   phone    = 4;  // 선택, E.164 형식
  UserRole role     = 5;  // CUSTOMER | OWNER만 허용 (기본: CUSTOMER)
}

message RegisterResponse {
  string                    user_id    = 1;
  google.protobuf.Timestamp created_at = 2;
}

message LoginRequest {
  string email     = 1;
  string password  = 2;
  string device_id = 3;  // 기기 식별자 (선택, 감사로그용)
}

message LoginResponse {
  string access_token       = 1;  // JWT, 15분 유효
  string refresh_token      = 2;  // Opaque token, 7일 유효
  int64  expires_in         = 3;  // access_token 만료 초 (900)
  User   user               = 4;  // 로그인한 유저 정보
}

message RefreshTokenRequest {
  string refresh_token = 1;
}

message RefreshTokenResponse {
  string access_token  = 1;
  string refresh_token = 2;  // RTR(Refresh Token Rotation): 항상 새 토큰 발급
  int64  expires_in    = 3;
}

message LogoutRequest {
  string access_token  = 1;  // 둘 중 하나 이상 필수
  string refresh_token = 2;
}

message LogoutResponse {
  bool revoked = 1;
}

message GetProfileRequest {
  string user_id = 1;  // 빈 문자열이면 인증 컨텍스트에서 추출
}

message GetProfileResponse {
  User user = 1;
}

message UpdateProfileRequest {
  string name                  = 1;  // 선택 (보내지 않으면 변경 없음)
  string phone                 = 2;
  string locale                = 3;  // "ko-KR"
  bool   notification_opt_out  = 4;
}

message UpdateProfileResponse {
  User user = 1;
}

// RBAC 인가 요청 — 다른 서비스의 auth interceptor에서 호출
message AuthorizeRequest {
  string user_id  = 1;
  string role     = 2;  // "CUSTOMER" | "OWNER" | "SUPER_ADMIN"
  string action   = 3;  // "reservation:create" 등
  string resource = 4;  // "restaurant:{id}" | "*"
}

message AuthorizeResponse {
  bool   allow  = 1;
  string reason = 2;  // 거부 시 사유
}

// ─── Service ─────────────────────────────────────────────────

service UserService {
  // 공개 API (인증 불필요)
  rpc Register     (RegisterRequest)     returns (RegisterResponse);
  rpc Login        (LoginRequest)        returns (LoginResponse);
  rpc RefreshToken (RefreshTokenRequest) returns (RefreshTokenResponse);

  // 인증 필요
  rpc Logout        (LogoutRequest)        returns (LogoutResponse);
  rpc GetProfile    (GetProfileRequest)    returns (GetProfileResponse);
  rpc UpdateProfile (UpdateProfileRequest) returns (UpdateProfileResponse);

  // 내부 서비스 간 호출 (gRPC metadata: x-internal-secret)
  rpc Authorize (AuthorizeRequest) returns (AuthorizeResponse);
}
```

---

### 5-3. restaurant/restaurant.proto (B 담당)

```protobuf
syntax = "proto3";
package restaurant.v1;
option go_package = "github.com/dinespot/backend/internal/gen/restaurant/v1;restaurantv1";

import "google/protobuf/timestamp.proto";
import "common/types.proto";

// ─── Enums ───────────────────────────────────────────────────

enum RestaurantStatus {
  RESTAURANT_STATUS_UNSPECIFIED    = 0;
  RESTAURANT_STATUS_PENDING_REVIEW = 1;
  RESTAURANT_STATUS_ACTIVE         = 2;
  RESTAURANT_STATUS_INACTIVE       = 3;
  RESTAURANT_STATUS_REJECTED       = 4;
}

// ─── Messages ────────────────────────────────────────────────

message OperatingHourSlot {
  int32  day_of_week   = 1;   // 0=월, 6=일
  string open_time     = 2;   // "HH:MM" (24시간제, KST)
  string close_time    = 3;   // "HH:MM"
  bool   is_closed     = 4;   // 정기 휴무일
  repeated BreakTime break_times = 5;
}

message BreakTime {
  string start_time = 1;   // "HH:MM"
  string end_time   = 2;   // "HH:MM"
}

message SeatPolicy {
  string policy_id       = 1;
  string restaurant_id   = 2;
  string table_type      = 3;   // "BOOTH", "TABLE", "BAR", "PRIVATE"
  int32  min_party       = 4;
  int32  max_party       = 5;
  int32  table_count     = 6;
  int32  slot_duration_min = 7; // 슬롯 단위 (기본 90분)
}

message Restaurant {
  string           id              = 1;
  string           tenant_id       = 2;   // = owner_id (테넌트 격리 기준)
  string           owner_id        = 3;
  string           name            = 4;
  string           address         = 5;
  string           category        = 6;   // "KOREAN", "JAPANESE" 등
  string           timezone        = 7;   // "Asia/Seoul"
  RestaurantStatus status          = 8;
  string           image_url       = 9;
  string           description     = 10;
  repeated OperatingHourSlot operating_hours = 11;
  repeated SeatPolicy seat_policies = 12;
  google.protobuf.Timestamp created_at = 13;
  google.protobuf.Timestamp updated_at = 14;
}

message CreateRestaurantRequest {
  string tenant_id    = 1;
  string name         = 2;
  string address      = 3;
  string category     = 4;
  string timezone     = 5;
  string description  = 6;
}

message CreateRestaurantResponse {
  string restaurant_id = 1;
}

message UpdateOperatingHoursRequest {
  string restaurant_id              = 1;
  repeated OperatingHourSlot hours  = 2;  // 전체 7일치 한 번에 교체
}

message UpdateOperatingHoursResponse {
  int64 version = 1;  // 낙관적 락용 버전
}

message UpsertSeatPolicyRequest {
  string restaurant_id     = 1;
  string table_type        = 2;
  int32  min_party         = 3;
  int32  max_party         = 4;
  int32  table_count       = 5;
  int32  slot_duration_min = 6;
}

message UpsertSeatPolicyResponse {
  string policy_id = 1;
}

message SetHolidayRequest {
  string restaurant_id = 1;
  string holiday_date  = 2;   // "YYYY-MM-DD"
  string reason        = 3;
}

message SetHolidayResponse {
  string holiday_id = 1;
}

// C(예약)가 호출하는 핵심 API
message GetReservationReferenceDataRequest {
  string restaurant_id = 1;
  string target_date   = 2;   // "YYYY-MM-DD"
}

message GetReservationReferenceDataResponse {
  string restaurant_id                   = 1;
  string target_date                     = 2;
  bool   holiday_flag                    = 3;   // true → 예약 생성 불가
  repeated OperatingHourSlot hours       = 4;   // 해당 요일의 영업시간
  repeated SeatPolicy seat_policies      = 5;
  string timezone                        = 6;
}

message SearchRestaurantsRequest {
  string area       = 1;
  string category   = 2;
  string date       = 3;   // "YYYY-MM-DD"
  string time       = 4;   // "HH:MM"
  int32  party_size = 5;
  common.v1.PageRequest page = 6;
}

message SearchRestaurantsResponse {
  repeated Restaurant restaurants = 1;
  common.v1.PageResponse page     = 2;
}

// ─── Service ─────────────────────────────────────────────────

service RestaurantService {
  rpc CreateRestaurant               (CreateRestaurantRequest)               returns (CreateRestaurantResponse);
  rpc GetRestaurant                  (GetRestaurantRequest)                  returns (GetRestaurantResponse);
  rpc UpdateOperatingHours           (UpdateOperatingHoursRequest)           returns (UpdateOperatingHoursResponse);
  rpc UpsertSeatPolicy               (UpsertSeatPolicyRequest)               returns (UpsertSeatPolicyResponse);
  rpc SetHoliday                     (SetHolidayRequest)                     returns (SetHolidayResponse);
  rpc SearchRestaurants              (SearchRestaurantsRequest)              returns (SearchRestaurantsResponse);

  // C에서 호출 (내부 API)
  rpc GetReservationReferenceData    (GetReservationReferenceDataRequest)    returns (GetReservationReferenceDataResponse);
}

message GetRestaurantRequest  { string restaurant_id = 1; }
message GetRestaurantResponse { Restaurant restaurant = 1; }
```

---

### 5-4. menu/menu.proto (B 담당)

```protobuf
syntax = "proto3";
package menu.v1;
option go_package = "github.com/dinespot/backend/internal/gen/menu/v1;menuv1";

import "google/protobuf/timestamp.proto";
import "common/types.proto";

enum MenuItemStatus {
  MENU_ITEM_STATUS_UNSPECIFIED = 0;
  MENU_ITEM_STATUS_ACTIVE      = 1;
  MENU_ITEM_STATUS_INACTIVE    = 2;
}

enum OrderStatus {
  ORDER_STATUS_UNSPECIFIED      = 0;
  ORDER_STATUS_PENDING          = 1;
  ORDER_STATUS_CONFIRMED        = 2;
  ORDER_STATUS_PAYMENT_PENDING  = 3;
  ORDER_STATUS_CANCELLED        = 4;
}

message MenuItem {
  string         id            = 1;
  string         restaurant_id = 2;
  string         category      = 3;
  string         name          = 4;
  int64          price         = 5;   // 원 단위
  string         description   = 6;
  string         image_url     = 7;
  int32          stock         = 8;   // -1 = 무제한
  bool           is_sold_out   = 9;
  MenuItemStatus status        = 10;
  int32          sort_order    = 11;
  google.protobuf.Timestamp created_at = 12;
}

message OrderItem {
  string menu_item_id = 1;
  int32  quantity     = 2;
  int64  unit_price   = 3;
}

message Order {
  string            id             = 1;
  string            reservation_id = 2;
  string            customer_id    = 3;
  repeated OrderItem items         = 4;
  int64             total_amount   = 5;
  OrderStatus       status         = 6;
  google.protobuf.Timestamp created_at = 7;
}

message UpsertMenuItemRequest {
  string         restaurant_id = 1;
  string         item_id       = 2;   // 빈 문자열이면 신규 생성
  string         category      = 3;
  string         name          = 4;
  int64          price         = 5;
  string         description   = 6;
  int32          stock         = 7;
  MenuItemStatus status        = 8;
  int32          sort_order    = 9;
}

message UpsertMenuItemResponse {
  string menu_item_id = 1;
}

message ListMenuItemsRequest {
  string restaurant_id        = 1;
  bool   include_inactive     = 2;  // 점주 관리 화면에서 true
}

message ListMenuItemsResponse {
  repeated MenuItem items = 1;
}

message PlacePreOrderRequest {
  string             reservation_id  = 1;
  repeated OrderItem items           = 2;
  string             idempotency_key = 3;
}

message PlacePreOrderResponse {
  string      order_id      = 1;
  OrderStatus status        = 2;
  int64       total_amount  = 3;
  string      payment_url   = 4;  // 결제 진행 URL (PAYMENT_PENDING일 때)
}

service MenuService {
  rpc UpsertMenuItem    (UpsertMenuItemRequest)    returns (UpsertMenuItemResponse);
  rpc ListMenuItems     (ListMenuItemsRequest)     returns (ListMenuItemsResponse);
  rpc PlacePreOrder     (PlacePreOrderRequest)     returns (PlacePreOrderResponse);
  rpc GetOrder          (GetOrderRequest)          returns (GetOrderResponse);
}

message GetOrderRequest  { string order_id = 1; }
message GetOrderResponse { Order order = 1; }
```

---

### 5-5. reservation/reservation.proto (C 담당)

```protobuf
syntax = "proto3";
package reservation.v1;
option go_package = "github.com/dinespot/backend/internal/gen/reservation/v1;reservationv1";

import "google/protobuf/timestamp.proto";
import "common/types.proto";

enum ReservationStatus {
  RESERVATION_STATUS_UNSPECIFIED = 0;
  RESERVATION_STATUS_PENDING     = 1;
  RESERVATION_STATUS_CONFIRMED   = 2;
  RESERVATION_STATUS_REJECTED    = 3;
  RESERVATION_STATUS_CANCELLED   = 4;
  RESERVATION_STATUS_COMPLETED   = 5;
  RESERVATION_STATUS_NO_SHOW     = 6;
}

message Reservation {
  string            id               = 1;
  string            customer_id      = 2;
  string            restaurant_id    = 3;
  int32             party_size       = 4;
  string            reserved_at      = 5;  // "YYYY-MM-DDTHH:MM:SS+09:00"
  ReservationStatus status           = 6;
  string            rejection_reason = 7;
  string            cancel_reason    = 8;
  string            idempotency_key  = 9;
  google.protobuf.Timestamp created_at = 10;
  google.protobuf.Timestamp updated_at = 11;
}

message TimeSlot {
  string reserved_at     = 1;   // "HH:MM"
  bool   available       = 2;
  int32  remaining_seats = 3;
}

message CreateReservationRequest {
  string customer_id      = 1;
  string restaurant_id    = 2;
  string reserved_at      = 3;  // "YYYY-MM-DDTHH:MM:SS+09:00"
  int32  party_size       = 4;
  string idempotency_key  = 5;  // UUID v4, 필수
}

message CreateReservationResponse {
  string            reservation_id = 1;
  ReservationStatus status         = 2;
}

message GetAvailabilityRequest {
  string restaurant_id = 1;
  string date          = 2;   // "YYYY-MM-DD"
  int32  party_size    = 3;
}

message GetAvailabilityResponse {
  repeated TimeSlot slots = 1;
}

message UpdateReservationRequest {
  string reservation_id  = 1;
  string new_reserved_at = 2;   // 변경하지 않으면 빈 문자열
  int32  new_party_size  = 3;   // 변경하지 않으면 0
}

message UpdateReservationResponse {
  ReservationStatus status     = 1;
  string            updated_at = 2;
}

message CancelReservationRequest {
  string reservation_id = 1;
  string reason         = 2;
}

message CancelReservationResponse {
  ReservationStatus status = 1;
}

message ApplyNoShowRequest {
  string reservation_id = 1;
  string operator_id    = 2;
}

message ApplyNoShowResponse {
  ReservationStatus status = 1;
}

message ConfirmReservationRequest {
  string reservation_id = 1;
  string operator_id    = 2;
}

message ConfirmReservationResponse {
  ReservationStatus status = 1;
}

message RejectReservationRequest {
  string reservation_id = 1;
  string reason         = 2;
  string operator_id    = 3;
}

message RejectReservationResponse {
  ReservationStatus status = 1;
}

message ListReservationsRequest {
  string customer_id    = 1;   // customer_id 또는 restaurant_id 중 하나
  string restaurant_id  = 2;
  ReservationStatus status = 3;
  common.v1.PageRequest page = 4;
}

message ListReservationsResponse {
  repeated Reservation reservations = 1;
  common.v1.PageResponse page       = 2;
}

service ReservationService {
  rpc CreateReservation   (CreateReservationRequest)   returns (CreateReservationResponse);
  rpc GetAvailability     (GetAvailabilityRequest)     returns (GetAvailabilityResponse);
  rpc UpdateReservation   (UpdateReservationRequest)   returns (UpdateReservationResponse);
  rpc CancelReservation   (CancelReservationRequest)   returns (CancelReservationResponse);
  rpc ConfirmReservation  (ConfirmReservationRequest)  returns (ConfirmReservationResponse);
  rpc RejectReservation   (RejectReservationRequest)   returns (RejectReservationResponse);
  rpc ApplyNoShow         (ApplyNoShowRequest)         returns (ApplyNoShowResponse);
  rpc ListReservations    (ListReservationsRequest)    returns (ListReservationsResponse);
}
```

---

### 5-6. waiting/waiting.proto (D 담당)

```protobuf
syntax = "proto3";
package waiting.v1;
option go_package = "github.com/dinespot/backend/internal/gen/waiting/v1;waitingv1";

import "google/protobuf/timestamp.proto";

enum WaitingStatus {
  WAITING_STATUS_UNSPECIFIED = 0;
  WAITING_STATUS_WAITING     = 1;
  WAITING_STATUS_READY       = 2;
  WAITING_STATUS_SEATED      = 3;
  WAITING_STATUS_NO_SHOW     = 4;
  WAITING_STATUS_CANCELLED   = 5;
}

message WaitingEntry {
  string        id            = 1;
  string        restaurant_id = 2;
  string        customer_id   = 3;
  int32         queue_no      = 4;
  int32         party_size    = 5;
  WaitingStatus status        = 6;
  int32         ahead_count   = 7;   // 앞 대기 팀 수
  int32         eta_min       = 8;   // 예상 대기 시간 (분)
  google.protobuf.Timestamp registered_at = 9;
  google.protobuf.Timestamp notified_at   = 10;
}

// SSE로 브로드캐스트하는 큐 상태 이벤트
message QueueUpdateEvent {
  string        restaurant_id = 1;
  int32         total_waiting = 2;
  int32         current_no    = 3;   // 현재 호출된 순번
  WaitingStatus status        = 4;
  string        event_id      = 5;   // UUID v4, 중복 수신 제거용
  string        occurred_at   = 6;   // ISO 8601
}

message JoinWaitingRequest {
  string restaurant_id    = 1;
  string customer_id      = 2;
  int32  party_size       = 3;
  string idempotency_key  = 4;
}

message JoinWaitingResponse {
  string        waiting_id  = 1;
  int32         queue_no    = 2;
  int32         ahead_count = 3;
  int32         eta_min     = 4;
}

message CallNextRequest {
  string restaurant_id = 1;
  string operator_id   = 2;
}

message CallNextResponse {
  string called_waiting_id = 1;
  int32  queue_no          = 2;
  int32  ttl_min           = 3;   // 응답 제한 시간 (기본 10분)
}

message CancelWaitingRequest {
  string waiting_id = 1;
  string reason     = 2;
}

message CancelWaitingResponse {
  WaitingStatus status = 1;
}

message ConfirmSeatedRequest {
  string waiting_id   = 1;
  string customer_id  = 2;
}

message ConfirmSeatedResponse {
  WaitingStatus status = 1;
}

message GetWaitingStatusRequest {
  string waiting_id = 1;
}

message GetWaitingStatusResponse {
  WaitingEntry entry = 1;
}

// SSE 스트리밍: 클라이언트가 구독, 서버가 큐 변경 시 이벤트 푸시
message StreamQueueRequest {
  string restaurant_id = 1;
}

service WaitingService {
  rpc JoinWaiting       (JoinWaitingRequest)       returns (JoinWaitingResponse);
  rpc CallNext          (CallNextRequest)           returns (CallNextResponse);
  rpc CancelWaiting     (CancelWaitingRequest)      returns (CancelWaitingResponse);
  rpc ConfirmSeated     (ConfirmSeatedRequest)      returns (ConfirmSeatedResponse);
  rpc GetWaitingStatus  (GetWaitingStatusRequest)   returns (GetWaitingStatusResponse);

  // 서버 → 클라이언트 실시간 스트리밍 (gRPC Server Streaming)
  // HTTP/SSE 래퍼는 Envoy가 처리
  rpc StreamQueueUpdates (StreamQueueRequest) returns (stream QueueUpdateEvent);
}
```

---

### 5-7. notification/notification.proto (D 담당)

```protobuf
syntax = "proto3";
package notification.v1;
option go_package = "github.com/dinespot/backend/internal/gen/notification/v1;notificationv1";

import "google/protobuf/timestamp.proto";

enum NotificationChannel {
  NOTIFICATION_CHANNEL_UNSPECIFIED = 0;
  NOTIFICATION_CHANNEL_KAKAO       = 1;
  NOTIFICATION_CHANNEL_SMS         = 2;
  NOTIFICATION_CHANNEL_EMAIL       = 3;
}

enum NotificationStatus {
  NOTIFICATION_STATUS_UNSPECIFIED = 0;
  NOTIFICATION_STATUS_SENT        = 1;
  NOTIFICATION_STATUS_FAILED      = 2;
  NOTIFICATION_STATUS_SKIPPED     = 3;   // dedupe로 스킵
}

// 알림 템플릿 ID (하드코딩 목록)
// RESERVATION_REQUESTED     - 예약 신청 (점주에게)
// RESERVATION_CONFIRMED     - 예약 확정 (고객에게)
// RESERVATION_REJECTED      - 예약 거절 (고객에게)
// RESERVATION_CANCELLED     - 예약 취소 (상대방에게)
// RESERVATION_REMINDER      - 방문 30분 전 리마인더 (고객에게)
// WAITING_REGISTERED        - 웨이팅 등록 완료 (고객에게)
// WAITING_CALLED            - 입장 가능 알림 (고객에게)
// WAITING_NO_SHOW           - 미응답 노쇼 처리 (고객에게)
// ORDER_CONFIRMED           - 사전 주문 확정 (고객에게)
// ORDER_PAYMENT_FAILED      - 결제 실패, 재결제 링크 (고객에게)

message SendNotificationRequest {
  NotificationChannel channel     = 1;
  string              template_id = 2;
  string              recipient   = 3;  // 전화번호 또는 이메일
  map<string, string> variables   = 4;  // 템플릿 변수 치환
  string              dedupe_key  = 5;  // 중복 방지 키 (24시간 TTL)
  // dedupe_key 규칙: "{template_id}:{target_id}:{date}"
  // 예: "RESERVATION_CONFIRMED:rsv_abc123:2026-03-24"
}

message SendNotificationResponse {
  string             message_id = 1;
  NotificationStatus status     = 2;
  string             sent_at    = 3;
}

message GetNotificationLogRequest {
  string dedupe_key = 1;
}

message GetNotificationLogResponse {
  string             message_id = 1;
  NotificationStatus status     = 2;
  string             error_msg  = 3;
  google.protobuf.Timestamp sent_at = 4;
}

service NotificationService {
  rpc SendNotification    (SendNotificationRequest)    returns (SendNotificationResponse);
  rpc GetNotificationLog  (GetNotificationLogRequest)  returns (GetNotificationLogResponse);
}
```

---

### 5-8. admin/admin.proto (E 담당)

```protobuf
syntax = "proto3";
package admin.v1;
option go_package = "github.com/dinespot/backend/internal/gen/admin/v1;adminv1";

import "google/protobuf/timestamp.proto";
import "common/types.proto";

enum ApprovalDecision {
  APPROVAL_DECISION_UNSPECIFIED = 0;
  APPROVAL_DECISION_APPROVE     = 1;
  APPROVAL_DECISION_REJECT      = 2;
}

enum AuditResult {
  AUDIT_RESULT_UNSPECIFIED = 0;
  AUDIT_RESULT_SUCCESS     = 1;
  AUDIT_RESULT_FAILURE     = 2;
}

message AuditLog {
  string      id         = 1;
  string      actor_id   = 2;
  string      actor_role = 3;
  string      action     = 4;   // "restaurant.approve", "user.restrict" 등
  string      resource   = 5;   // "restaurant:abc123"
  string      request_id = 6;
  AuditResult result     = 7;
  string      detail     = 8;   // JSON 직렬화된 변경 내용
  google.protobuf.Timestamp created_at = 9;
}

message KPIData {
  string  tenant_id         = 1;
  string  period_from       = 2;
  string  period_to         = 3;
  int64   total_reservations = 4;
  float   reservation_rate   = 5;   // 예약 성사율
  float   cancel_rate        = 6;
  float   noshow_rate        = 7;
  float   wait_avg_min       = 8;   // 평균 웨이팅 시간 (분)
  int64   total_orders       = 9;
  int64   total_revenue      = 10;  // 원 단위
}

message ApproveRestaurantRequest {
  string           admin_id      = 1;
  string           restaurant_id = 2;
  ApprovalDecision decision      = 3;
  string           reason        = 4;
}

message ApproveRestaurantResponse {
  string approval_status = 1;
  string approved_at     = 2;
}

message WriteAuditLogRequest {
  string      actor_id   = 1;
  string      actor_role = 2;
  string      action     = 3;
  string      resource   = 4;
  string      request_id = 5;
  AuditResult result     = 6;
  string      detail     = 7;
}

message WriteAuditLogResponse {
  string log_id = 1;
}

message QueryAuditLogsRequest {
  string actor_id    = 1;
  string action      = 2;
  common.v1.DateRange date_range = 3;
  common.v1.PageRequest page     = 4;
}

message QueryAuditLogsResponse {
  repeated AuditLog logs          = 1;
  common.v1.PageResponse page     = 2;
}

message GetKPIRequest {
  string tenant_id    = 1;   // 빈 문자열이면 전체
  string from         = 2;   // "YYYY-MM-DD"
  string to           = 3;   // "YYYY-MM-DD"
  string granularity  = 4;   // "DAY" | "WEEK" | "MONTH"
}

message GetKPIResponse {
  repeated KPIData data = 1;
}

service AdminService {
  rpc ApproveRestaurant (ApproveRestaurantRequest) returns (ApproveRestaurantResponse);
  rpc WriteAuditLog     (WriteAuditLogRequest)     returns (WriteAuditLogResponse);
  rpc QueryAuditLogs    (QueryAuditLogsRequest)    returns (QueryAuditLogsResponse);
  rpc GetKPI            (GetKPIRequest)            returns (GetKPIResponse);
}
```

---

## 6. Redis 키 네이밍 컨벤션 (전 서비스 공용)

> 형식: `{namespace}:{service}:{entity}:{id}:{field}`
> 구분자: `:` (콜론)
> 모든 키는 TTL을 반드시 설정할 것

| 키 패턴 | 담당 | TTL | 설명 |
|---------|------|-----|------|
| `auth:blacklist:{jti}` | A | access_token 잔여 만료시간 | JWT 블랙리스트 |
| `auth:refresh:{user_id}:{device_id}` | A | 7일 | refresh token 저장 |
| `auth:login_fail:{email}` | A | 1시간 | 로그인 실패 횟수 (rate limit) |
| `lock:slot:{restaurant_id}:{reserved_at}` | C | 30초 | 예약 슬롯 분산 락 |
| `idempotent:rsv:{idempotency_key}` | C | 24시간 | 예약 멱등키 → reservation_id |
| `waiting:seq:{restaurant_id}` | D | 영구 (당일 자정 초기화) | 웨이팅 순번 카운터 |
| `waiting:queue:{restaurant_id}` | D | 영구 | Sorted Set (score=queue_no) |
| `waiting:noshow_timer:{waiting_id}` | D | 10분 | 노쇼 TTL 키 (만료 시 이벤트) |
| `notif:dedupe:{dedupe_key}` | D | 24시간 | 알림 중복 방지 |
| `conn:queue:{instance_id}` | C | 30초 | Connection Queue 대기열 |
| `cache:ref_data:{restaurant_id}:{date}` | B | 5분 | 예약 참조 데이터 캐시 |

---

## 7. 이벤트 스키마 (Redis Streams 기반)

Redis Streams 키: `events:{event_name}` (예: `events:reservation.canceled.v1`)

### 7-1. reservation.canceled.v1 (C → D)

```json
{
  "event_id":       "uuid-v4",
  "event_name":     "reservation.canceled.v1",
  "occurred_at":    "2026-03-24T18:30:00+09:00",
  "reservation_id": "rsv_abc123",
  "restaurant_id":  "rest_xyz",
  "customer_id":    "usr_def",
  "reserved_at":    "2026-03-24T19:00:00+09:00",
  "party_size":     2,
  "cancel_reason":  "customer_request"
}
```

### 7-2. waiting.joined.v1 (D → E)

```json
{
  "event_id":       "uuid-v4",
  "event_name":     "waiting.joined.v1",
  "occurred_at":    "2026-03-24T18:35:00+09:00",
  "waiting_id":     "wait_abc",
  "restaurant_id":  "rest_xyz",
  "customer_id":    "usr_ghi",
  "queue_no":       5,
  "party_size":     2
}
```

### 7-3. waiting.called.v1 (D → D 내부 + FE SSE)

```json
{
  "event_id":       "uuid-v4",
  "event_name":     "waiting.called.v1",
  "occurred_at":    "2026-03-24T18:40:00+09:00",
  "waiting_id":     "wait_abc",
  "restaurant_id":  "rest_xyz",
  "queue_no":       5,
  "ttl_min":        10
}
```

### 7-4. waiting.canceled.v1 (D → E)

```json
{
  "event_id":       "uuid-v4",
  "event_name":     "waiting.canceled.v1",
  "occurred_at":    "2026-03-24T18:45:00+09:00",
  "waiting_id":     "wait_abc",
  "restaurant_id":  "rest_xyz",
  "customer_id":    "usr_ghi",
  "queue_no":       5,
  "reason":         "customer_request"
}
```

### 7-5. restaurant.status_changed.v1 (B → E)

```json
{
  "event_id":       "uuid-v4",
  "event_name":     "restaurant.status_changed.v1",
  "occurred_at":    "2026-03-24T10:00:00+09:00",
  "restaurant_id":  "rest_xyz",
  "tenant_id":      "tenant_abc",
  "from_status":    "PENDING_REVIEW",
  "to_status":      "ACTIVE"
}
```

---

## 8. gRPC Metadata 표준 (전 서비스 공통)

모든 서비스 간 호출 시 아래 메타데이터를 반드시 전달해야 합니다.

| 키 | 설명 | 설정 주체 |
|----|------|---------|
| `x-request-id` | 요청 추적 ID (UUID v4) | API Gateway 또는 첫 진입 서비스 |
| `x-user-id` | 인증된 유저 ID | A의 auth interceptor |
| `x-tenant-id` | 테넌트 ID (= owner_id) | A의 auth interceptor |
| `x-role` | 유저 역할 | A의 auth interceptor |
| `x-forwarded-for` | 클라이언트 IP | Envoy |
| `x-internal-secret` | 서비스 간 내부 호출 검증 | 환경변수로 주입 |

---

## 9. gRPC 에러 코드 표준 매핑

```go
// backend/pkg/errors/codes.go
package errors

import "google.golang.org/grpc/codes"

var (
  // 인증/권한
  ErrUnauthenticated  = codes.Unauthenticated   // 토큰 없음/만료
  ErrPermissionDenied = codes.PermissionDenied  // 권한 없음

  // 입력 오류
  ErrInvalidArgument  = codes.InvalidArgument   // 유효성 오류
  ErrAlreadyExists    = codes.AlreadyExists      // 중복

  // 상태 오류
  ErrNotFound         = codes.NotFound           // 리소스 없음
  ErrConflict         = codes.Aborted            // 슬롯 충돌
  ErrInvalidState     = codes.FailedPrecondition // 상태 전이 불가
  ErrFailedPrecond    = codes.FailedPrecondition // 전제 조건 미충족

  // 서버 오류
  ErrInternal         = codes.Internal           // 서버 내부 오류
  ErrUnavailable      = codes.Unavailable        // 외부 서비스 장애
)
```

---

## 10. 공통 로그 JSON 구조 (전 서비스 의무)

모든 서비스는 아래 필드를 반드시 포함한 JSON 로그를 stdout으로 출력합니다.

```json
{
  "timestamp":   "2026-03-24T18:30:00.123+09:00",
  "level":       "info",
  "service":     "reservation-service",
  "trace_id":    "4bf92f3577b34da6a3ce929d0e0e4736",
  "request_id":  "01HX7YZABC123DEF456",
  "tenant_id":   "tenant_abc",
  "user_id":     "usr_def",
  "method":      "CreateReservation",
  "duration_ms": 45,
  "status_code": "OK",
  "message":     "reservation created",
  "error":       ""
}
```

**level 기준:**
- `debug`: 개발 환경 상세 정보
- `info`: 정상 처리 이벤트 (API 요청/응답)
- `warn`: 재시도 가능한 오류, SLO 주의
- `error`: 처리 실패, 즉시 확인 필요
- `fatal`: 서비스 중단 수준

---

## 11. Prometheus 메트릭 표준 (전 서비스 의무)

각 서비스는 `:9100`~`:9106` 포트에서 `/metrics` 엔드포인트를 노출합니다.

| 메트릭명 | 타입 | labels | 설명 |
|---------|------|--------|------|
| `api_request_total` | Counter | service, method, status_code | API 호출 횟수 |
| `api_request_duration_ms` | Histogram | service, method | API 응답 시간 |
| `api_error_total` | Counter | service, method, error_code | 에러 발생 횟수 |
| `db_query_duration_ms` | Histogram | service, query_type | DB 쿼리 시간 |
| `redis_operation_total` | Counter | service, operation, status | Redis 연산 횟수 |
| `queue_depth` | Gauge | restaurant_id | 웨이팅 큐 깊이 |
| `notification_sent_total` | Counter | channel, template_id, status | 알림 발송 횟수 |
| `reservation_conflict_total` | Counter | restaurant_id | 예약 슬롯 충돌 횟수 |
