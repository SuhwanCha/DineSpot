# E 담당 문서 — 관리자/운영/관측성

> 이 문서만 보고 개발할 수 있도록 작성되었습니다.
> Proto 스키마 전체는 `00-공용-proto-스키마.md` 섹션 5-8을 참고하세요.

---

## 0. 담당 서비스 & 디렉토리

### 담당 컨테이너

| 컨테이너명 | gRPC 포트 | 메트릭 포트 | 설명 |
|-----------|-----------|------------|------|
| `dinespot-admin-service` | 50057 | 9106 | 관리자 승인/감사로그/KPI |
| `dinespot-prometheus` | 9090 | - | 메트릭 수집 |
| `dinespot-grafana` | 3001 | - | 대시보드 시각화 |

### 담당 디렉토리

```
backend/
├── internal/
│   └── admin/                       ← admin-service 핵심 구현
│       ├── handler.go               # gRPC 핸들러 (AdminServiceServer 구현)
│       ├── service.go               # 비즈니스 로직 (승인/KPI 집계)
│       ├── repository.go            # DB 접근 계층
│       ├── kpi_aggregator.go        # KPI 계산 쿼리
│       └── handler_test.go
└── pkg/
    └── middleware/
        ├── logging.go               ← 공통 로그 인터셉터 (A~D가 import)
        └── metrics.go               ← 공통 메트릭 인터셉터 (A~D가 import)

infra/
├── mysql/
│   └── init/
│       └── 07_audit.sql             ← 감사로그 DDL
├── prometheus/
│   ├── prometheus.yml               ← 수집 타겟 설정
│   └── rules/
│       └── dinespot.yml             ← 알람 규칙
└── grafana/
    └── provisioning/
        ├── datasources/
        │   └── prometheus.yml       ← Prometheus 데이터소스 자동 등록
        └── dashboards/
            ├── dashboards.yml       ← 대시보드 자동 로딩 설정
            ├── overview.json        ← 전체 서비스 현황 대시보드
            ├── reservations.json    ← 예약 KPI 대시보드
            └── notifications.json   ← 알림 현황 대시보드
```

### docker-compose 서비스 스니펫

```yaml
admin-service:
  build:
    context: ./backend
    dockerfile: Dockerfile
    target: service
  container_name: dinespot-admin-service
  command: ["/app/server", "--service=admin", "--port=50057"]
  ports:
    - "50057:50057"
    - "9106:9106"
  environment:
    SERVICE_NAME: admin-service
    DB_DSN: "dinespot:dinespot_pass@tcp(mysql:3306)/dinespot?parseTime=true&loc=Asia%2FSeoul"
    REDIS_ADDR: "redis:6379"
    AUDIT_LOG_RETENTION_DAYS: "365"
    LOG_LEVEL: "info"
  depends_on:
    mysql:
      condition: service_healthy
  networks: [dinespot-net]

prometheus:
  image: prom/prometheus:v2.50.1
  container_name: dinespot-prometheus
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
  ports:
    - "3001:3000"
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
```

### 로컬 개발 명령어

```bash
# admin + 관측성 스택 시작
docker compose up -d mysql redis admin-service prometheus grafana

# 전체 서비스 시작 (메트릭 수집 포함)
docker compose up -d

# Grafana 접속: http://localhost:3001 (admin/admin)
# Prometheus 접속: http://localhost:9090

# 내 서비스 로그 확인
docker compose logs -f admin-service

# 재빌드
docker compose up -d --build admin-service

# 테스트 실행
cd backend && go test ./internal/admin/... -v
cd backend && go test ./pkg/middleware/... -v

# Prometheus에서 즉시 규칙 리로드 (알람 규칙 변경 후)
curl -X POST http://localhost:9090/-/reload

# 감사로그 조회 테스트
grpcurl -plaintext \
  -H "authorization: Bearer {admin_token}" \
  -d '{"date_range":{"from":"2026-03-24T00:00:00Z","to":"2026-03-24T23:59:59Z"}}' \
  localhost:50057 admin.v1.AdminService/QueryAuditLogs
```

---

## 1. 역할 목표

- 운영자는 서비스 이상을 **5분 내** 감지하고 조치 시작할 수 있어야 한다.
- 감사로그/대시보드/알람이 제품 운영의 단일 진실 공급원(SSOT) 역할을 해야 한다.
- **W1에 공통 로그/메트릭 인터셉터를 먼저 A~D에 배포**해야 W3 이후 대시보드 데이터가 수집됨. 이것이 가장 중요한 첫 번째 작업.

---

## 2. 기능 요구사항 (FR)

| ID | 요구사항 | 우선순위 |
|----|---------|---------|
| FR-E-01 | 레스토랑 등록 신청 승인/거절 워크플로우 | P0 |
| FR-E-02 | 감사로그 저장/조회 (actor, action, resource, result) | P0 |
| FR-E-03 | KPI 대시보드 (예약/취소/웨이팅/알림 지표) | P1 |
| FR-E-04 | 알람 규칙 정의 + 온콜 연동 | P1 |
| FR-E-05 | 장애 대응 Runbook 작성 + 관리 | P1 |
| FR-E-06 | 공통 로그 인터셉터 배포 (A~D 전 서비스) | P0 |
| FR-E-07 | 공통 메트릭 인터셉터 배포 (A~D 전 서비스) | P0 |

---

## 3. API/함수 구현 명세

| ID | 함수 | Input | Output | 실패 조건 |
|----|------|-------|--------|----------|
| E-FN-01 | `ApproveRestaurant(ctx, req)` | `admin_id, restaurant_id, decision, reason` | `approval_status, approved_at` | 권한 없음 → `PERMISSION_DENIED` |
| E-FN-02 | `WriteAuditLog(ctx, event)` | `actor_id, actor_role, action, resource, request_id, result, detail` | `log_id` | 필드 누락 → `INVALID_ARGUMENT` |
| E-FN-03 | `QueryAuditLogs(ctx, filter)` | `actor_id, action, date_range, page` | `logs[], next_cursor` | 조회 범위 30일 초과 → `FAILED_PRECONDITION` |
| E-FN-04 | `GetKPI(ctx, req)` | `tenant_id, from, to, granularity` | `KPIData[]` | 데이터 없음 → 빈 배열 |

---

## 4. DB 스키마

### 파일 위치: `infra/mysql/init/07_audit.sql`

```sql
-- 감사 로그 (불변 로그, UPDATE/DELETE 불가 정책)
CREATE TABLE IF NOT EXISTS audit_logs (
  id          CHAR(36)     NOT NULL,
  actor_id    CHAR(36)     NOT NULL,
  actor_role  VARCHAR(50)  NOT NULL,
  action      VARCHAR(100) NOT NULL,   -- 'restaurant.approve', 'reservation.cancel' 등
  resource    VARCHAR(200) NOT NULL,   -- 'restaurant:abc123', 'reservation:def456'
  request_id  VARCHAR(128) DEFAULT NULL,
  result      ENUM('SUCCESS','FAILURE') NOT NULL,
  detail      JSON         DEFAULT NULL,  -- 변경 전후 값 등
  created_at  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_audit_actor_id (actor_id),
  KEY idx_audit_action (action),
  KEY idx_audit_created_at (created_at),
  KEY idx_audit_resource (resource(100))
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- audit_logs는 DELETE/UPDATE 금지 (트리거로 강제)
-- CREATE TRIGGER prevent_audit_update BEFORE UPDATE ON audit_logs
--   FOR EACH ROW SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT='audit_logs is immutable';
```

---

## 5. 공통 로그 인터셉터 (W1 배포 필수)

파일 위치: `backend/pkg/middleware/logging.go`

이 파일은 **E 담당자가 작성**하고, A~D 서비스 모두 grpc.Server 시작 시 등록합니다.

```go
// backend/pkg/middleware/logging.go
package middleware

import (
    "context"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/metadata"
    "google.golang.org/grpc/status"
)

// 공통 로그 JSON 구조체 (모든 서비스가 이 형식으로 출력)
type LogEntry struct {
    Timestamp  string `json:"timestamp"`
    Level      string `json:"level"`
    Service    string `json:"service"`
    TraceID    string `json:"trace_id"`
    RequestID  string `json:"request_id"`
    TenantID   string `json:"tenant_id"`
    UserID     string `json:"user_id"`
    Method     string `json:"method"`
    DurationMs int64  `json:"duration_ms"`
    StatusCode string `json:"status_code"`
    Message    string `json:"message"`
    Error      string `json:"error,omitempty"`
}

// UnaryLoggingInterceptor — 모든 서비스에 등록
// 사용법: grpc.ChainUnaryInterceptor(..., middleware.UnaryLoggingInterceptor(serviceName, logger))
func UnaryLoggingInterceptor(serviceName string, logger Logger) grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
        start := time.Now()

        // request_id, trace_id 추출 (없으면 생성)
        md, _ := metadata.FromIncomingContext(ctx)
        requestID := extractMD(md, "x-request-id")
        traceID := extractMD(md, "x-trace-id")
        tenantID := extractMD(md, "x-tenant-id")
        userID := extractMD(md, "x-user-id")

        if requestID == "" {
            requestID = generateRequestID()
            // 다운스트림에도 전파
            ctx = metadata.AppendToOutgoingContext(ctx, "x-request-id", requestID)
        }

        // 핸들러 실행
        resp, err := handler(ctx, req)

        // 로그 출력
        duration := time.Since(start).Milliseconds()
        statusCode := "OK"
        errMsg := ""
        if err != nil {
            st, _ := status.FromError(err)
            statusCode = st.Code().String()
            errMsg = st.Message()
        }

        entry := LogEntry{
            Timestamp:  time.Now().Format("2006-01-02T15:04:05.000+09:00"),
            Level:      logLevel(statusCode),
            Service:    serviceName,
            TraceID:    traceID,
            RequestID:  requestID,
            TenantID:   tenantID,
            UserID:     userID,
            Method:     info.FullMethod,
            DurationMs: duration,
            StatusCode: statusCode,
            Message:    "gRPC request processed",
            Error:      errMsg,
        }
        logger.JSON(entry)

        return resp, err
    }
}

func logLevel(statusCode string) string {
    switch statusCode {
    case "OK":
        return "info"
    case "INTERNAL", "UNKNOWN":
        return "error"
    default:
        return "warn"
    }
}
```

---

## 6. 공통 메트릭 인터셉터 (W1 배포 필수)

파일 위치: `backend/pkg/middleware/metrics.go`

```go
// backend/pkg/middleware/metrics.go
package middleware

import (
    "context"
    "time"

    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "google.golang.org/grpc"
    "google.golang.org/grpc/status"
)

var (
    requestTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "api_request_total",
            Help: "Total number of gRPC requests",
        },
        []string{"service", "method", "status_code"},
    )

    requestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "api_request_duration_ms",
            Help:    "gRPC request duration in milliseconds",
            Buckets: []float64{10, 25, 50, 100, 200, 500, 1000, 5000},
        },
        []string{"service", "method"},
    )

    errorTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "api_error_total",
            Help: "Total number of gRPC errors",
        },
        []string{"service", "method", "error_code"},
    )
)

// UnaryMetricsInterceptor — 모든 서비스에 등록
func UnaryMetricsInterceptor(serviceName string) grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
        start := time.Now()
        resp, err := handler(ctx, req)
        duration := float64(time.Since(start).Milliseconds())

        method := lastSegment(info.FullMethod)   // "/user.v1.UserService/Login" → "Login"
        statusCode := "OK"
        if err != nil {
            st, _ := status.FromError(err)
            statusCode = st.Code().String()
            errorTotal.WithLabelValues(serviceName, method, statusCode).Inc()
        }

        requestTotal.WithLabelValues(serviceName, method, statusCode).Inc()
        requestDuration.WithLabelValues(serviceName, method).Observe(duration)

        return resp, err
    }
}
```

---

## 7. 감사로그 수집 방식

각 서비스에서 중요 액션 발생 시 admin-service의 `WriteAuditLog()`를 **비동기**로 호출합니다.

```
[각 서비스]                    [admin-service]          [audit_logs DB]
     │                              │                         │
     │  중요 액션 완료              │                         │
     │  (예: restaurant approve)    │                         │
     │  비동기 goroutine 실행 ──────▶ WriteAuditLog()         │
     │  (메인 응답에 영향 없음)      │──────────────────────────▶
     │                              │  INSERT audit_logs      │
```

```go
// 각 서비스에서 중요 액션 후 호출하는 헬퍼 (공통 유틸)
// backend/pkg/audit/client.go

func WriteAuditLogAsync(ctx context.Context, adminClient adminv1.AdminServiceClient, req *adminv1.WriteAuditLogRequest) {
    go func() {
        // 별도 context 사용 (원본 ctx 취소 영향 안 받음)
        auditCtx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
        defer cancel()
        if _, err := adminClient.WriteAuditLog(auditCtx, req); err != nil {
            // 감사로그 실패는 메인 로직에 영향 없음, 단 로그 출력
            log.Error("audit log write failed", "error", err)
        }
    }()
}
```

**감사로그를 남겨야 하는 액션 목록:**

| 서비스 | action | resource 형식 |
|--------|--------|--------------|
| restaurant-service | `restaurant.create` | `restaurant:{id}` |
| restaurant-service | `restaurant.status_change` | `restaurant:{id}` |
| reservation-service | `reservation.confirm` | `reservation:{id}` |
| reservation-service | `reservation.reject` | `reservation:{id}` |
| reservation-service | `reservation.cancel` | `reservation:{id}` |
| reservation-service | `reservation.noshow` | `reservation:{id}` |
| admin-service | `restaurant.approve` | `restaurant:{id}` |
| admin-service | `user.restrict` | `user:{id}` |
| user-service | `user.login` | `user:{id}` |
| user-service | `user.logout` | `user:{id}` |

---

## 8. Prometheus 설정

### `infra/prometheus/prometheus.yml`

```yaml
global:
  scrape_interval:     15s
  evaluation_interval: 15s
  external_labels:
    environment: 'local'

rule_files:
  - /etc/prometheus/rules/*.yml

scrape_configs:
  - job_name: 'user-service'
    static_configs:
      - targets: ['user-service:9100']
    metrics_path: '/metrics'

  - job_name: 'restaurant-service'
    static_configs:
      - targets: ['restaurant-service:9101']

  - job_name: 'reservation-service'
    static_configs:
      - targets: ['reservation-service:9102']

  - job_name: 'waiting-service'
    static_configs:
      - targets: ['waiting-service:9103']

  - job_name: 'menu-service'
    static_configs:
      - targets: ['menu-service:9104']

  - job_name: 'notification-service'
    static_configs:
      - targets: ['notification-service:9105']

  - job_name: 'admin-service'
    static_configs:
      - targets: ['admin-service:9106']

  - job_name: 'mysql'
    static_configs:
      - targets: ['mysql:9104']   # mysql_exporter (선택)

  - job_name: 'redis'
    static_configs:
      - targets: ['redis:9121']   # redis_exporter (선택)
```

---

## 9. 알람 규칙

### `infra/prometheus/rules/dinespot.yml`

```yaml
groups:
  - name: dinespot.api
    rules:
      # API 에러율 > 2% (5분 윈도우)
      - alert: HighApiErrorRate
        expr: |
          sum(rate(api_error_total[5m])) by (service)
          / sum(rate(api_request_total[5m])) by (service)
          > 0.02
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.service }} 에러율 {{ $value | humanizePercentage }} 초과"
          description: "5분 윈도우 기준 에러율이 2%를 초과했습니다. 즉시 확인하세요."

      # 예약 API p99 > 200ms (10분 윈도우)
      - alert: ReservationApiSlowResponse
        expr: |
          histogram_quantile(0.99,
            sum(rate(api_request_duration_ms_bucket{service="reservation-service"}[10m])) by (le)
          ) > 200
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "예약 API p99 응답시간 {{ $value }}ms 초과"

  - name: dinespot.notification
    rules:
      # 알림 실패율 > 1% (10분 윈도우)
      - alert: HighNotificationFailureRate
        expr: |
          sum(rate(notification_sent_total{status="FAILED"}[10m]))
          / sum(rate(notification_sent_total[10m]))
          > 0.01
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "알림 실패율 {{ $value | humanizePercentage }} 초과"

  - name: dinespot.infra
    rules:
      # MySQL 연결 불가 (30초 이상 응답 없음)
      - alert: MysqlDown
        expr: up{job="mysql"} == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "MySQL 서비스 다운"

      # Redis 연결 불가
      - alert: RedisDown
        expr: up{job="redis"} == 0
        for: 30s
        labels:
          severity: critical
        annotations:
          summary: "Redis 서비스 다운"

      # 웨이팅 큐 과부하 (특정 매장 50명 이상)
      - alert: WaitingQueueDepthHigh
        expr: queue_depth > 50
        for: 1m
        labels:
          severity: info
        annotations:
          summary: "매장 {{ $labels.restaurant_id }} 웨이팅 {{ $value }}명"
```

---

## 10. Grafana 설정

### `infra/grafana/provisioning/datasources/prometheus.yml`

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      timeInterval: "15s"
```

### `infra/grafana/provisioning/dashboards/dashboards.yml`

```yaml
apiVersion: 1
providers:
  - name: DineSpot
    orgId: 1
    folder: DineSpot
    type: file
    options:
      path: /etc/grafana/provisioning/dashboards
      foldersFromFilesStructure: true
```

### 대시보드 구성 (overview.json)

Grafana JSON 대시보드 대신 **패널 목록**으로 정의합니다. 실제 JSON은 Grafana UI에서 생성 후 export.

**패널 목록 (overview.json):**

| 패널 이름 | 쿼리 | 시각화 타입 |
|---------|------|-----------|
| 서비스별 초당 요청수 | `sum(rate(api_request_total[1m])) by (service)` | Time Series |
| 서비스별 에러율 | `sum(rate(api_error_total[5m])) by (service) / sum(rate(api_request_total[5m])) by (service)` | Gauge (임계값 2% 빨강) |
| 예약 API p99 응답시간 | `histogram_quantile(0.99, sum(rate(api_request_duration_ms_bucket{service="reservation-service"}[5m])) by (le))` | Stat |
| 알림 채널별 발송 현황 | `sum(increase(notification_sent_total[1h])) by (channel, status)` | Bar Chart |
| 웨이팅 큐 현황 | `queue_depth` by restaurant_id | Table |
| 예약 충돌 차단 건수 | `sum(increase(reservation_conflict_total[1h]))` | Stat |

---

## 11. KPI 집계 쿼리 (`GetKPI` 구현)

파일 위치: `backend/internal/admin/kpi_aggregator.go`

```go
// 예약 성사율 (CONFIRMED / 전체 예약 신청)
const queryReservationRate = `
    SELECT
        DATE_FORMAT(created_at, ?) AS period,   -- '%Y-%m-%d' or '%Y-%u'
        COUNT(*) AS total,
        SUM(CASE WHEN status = 'CONFIRMED' THEN 1 ELSE 0 END) AS confirmed,
        SUM(CASE WHEN status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled,
        SUM(CASE WHEN status = 'NO_SHOW' THEN 1 ELSE 0 END) AS noshow
    FROM reservations
    WHERE restaurant_id IN (
        SELECT id FROM restaurants WHERE tenant_id = ?
    )
    AND created_at BETWEEN ? AND ?
    GROUP BY period
    ORDER BY period
`

// 평균 웨이팅 시간 (SEATED 기준: seated_at - registered_at)
const queryWaitingAvg = `
    SELECT
        DATE_FORMAT(registered_at, ?) AS period,
        AVG(TIMESTAMPDIFF(MINUTE, registered_at, seated_at)) AS avg_wait_min
    FROM waiting_entries
    WHERE restaurant_id IN (
        SELECT id FROM restaurants WHERE tenant_id = ?
    )
    AND status = 'SEATED'
    AND registered_at BETWEEN ? AND ?
    GROUP BY period
    ORDER BY period
`

// 알림 실패율 (notification_logs)
const queryNotificationFailRate = `
    SELECT
        DATE_FORMAT(created_at, ?) AS period,
        COUNT(*) AS total,
        SUM(CASE WHEN status = 'FAILED' THEN 1 ELSE 0 END) AS failed
    FROM notification_logs
    WHERE created_at BETWEEN ? AND ?
    GROUP BY period
    ORDER BY period
`
```

---

## 12. 공통 로그 JSON 구조 (A~D 전 서비스 의무 준수)

> 이 형식을 준수하지 않는 PR은 merge 금지 (CI에서 lint 체크 예정)

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
- `debug`: 개발 환경 상세 정보 (운영 환경에서는 비활성)
- `info`: 정상 처리 이벤트
- `warn`: gRPC status `INVALID_ARGUMENT`, `NOT_FOUND` 등 클라이언트 오류
- `error`: `INTERNAL`, `UNAVAILABLE` 등 서버 오류
- `fatal`: 서비스 중단 수준 (프로세스 종료 전)

---

## 13. Runbook 템플릿 (장애 시나리오 5개)

### Runbook 파일 위치: `infra/runbook/`

#### RB-01: MySQL 다운

```markdown
## 증상
- 모든 서비스에서 DB 연결 오류
- Grafana: MysqlDown 알람 발생

## 즉각 확인
1. docker compose ps mysql  # 컨테이너 상태 확인
2. docker compose logs mysql --tail=50  # 오류 메시지 확인

## 복구 절차
1. 컨테이너 재시작: docker compose restart mysql
2. 30초 대기 후 healthcheck 통과 확인
3. 서비스 재시작: docker compose restart user-service reservation-service ...
4. smoke test: grpcurl localhost:50051 user.v1.UserService/Login

## 사후 조치
- MySQL 로그 분석 (OOM, disk full 등)
- 재발 방지 조치 작성
```

#### RB-02: 예약 API 응답 지연 (p99 > 200ms)

```markdown
## 증상
- Grafana: ReservationApiSlowResponse 알람
- 고객 예약 버튼 응답 느림

## 즉각 확인
1. Prometheus: api_request_duration_ms by method 확인
2. 슬로우 쿼리 확인:
   docker compose exec mysql mysql -u dinespot -pdinespot_pass -e \
   "SHOW PROCESSLIST; SELECT * FROM information_schema.INNODB_TRX;"

## 복구 절차
1. Redis 락 잔존 여부 확인:
   docker compose exec redis redis-cli KEYS "lock:slot:*"
2. 잔존 락 수동 삭제 (주의: 실제 예약 진행 중인지 확인 후):
   docker compose exec redis redis-cli DEL "lock:slot:{key}"
3. 필요 시 reservation-service 재시작

## 사후 조치
- 락 TTL 로직 점검
- DB 인덱스 최적화 여부 확인
```

#### RB-03: 알림 발송 실패율 급등

```markdown
## 증상
- Grafana: HighNotificationFailureRate 알람
- 고객에게 예약 확정 알림 미도달

## 즉각 확인
1. notification_logs 최근 실패 조회:
   SELECT * FROM notification_logs WHERE status='FAILED' ORDER BY created_at DESC LIMIT 20;
2. 외부 API 상태 확인 (카카오 공식 상태 페이지)

## 복구 절차
1. KAKAO 장애 시: SMS fallback이 자동 작동해야 함 (로그 확인)
2. 전체 채널 장애 시: 점주에게 수동 알림 전송 안내
3. FAILED 로그의 dedupe_key 확인 → 재발송 필요 시 dedupe_key 삭제 후 재시도:
   docker compose exec redis redis-cli DEL "notif:dedupe:{key}"
```

#### RB-04: Redis 다운

```markdown
## 증상
- 웨이팅 순번 발급 불가
- 예약 분산 락 불가 (예약 API 실패)

## 즉각 확인
1. docker compose ps redis
2. docker compose logs redis --tail=50

## 복구 절차
1. docker compose restart redis
2. Redis 재시작 후 AOF 복구 확인 (appendonly yes 설정)
3. 웨이팅 큐 Redis → MySQL 동기화 확인:
   - waiting_entries 테이블의 WAITING 상태 항목으로 Redis 큐 재구성 필요

## 수동 큐 복구 스크립트
docker compose exec redis redis-cli
> SELECT * FROM waiting_entries WHERE status='WAITING' ORDER BY queue_no;
> ZADD waiting:queue:{restaurant_id} {queue_no} {waiting_id}  # 각 항목별 실행
```

#### RB-05: 서비스 전체 응답 불가 (Envoy 이슈)

```markdown
## 증상
- FE에서 모든 API 호출 실패
- Envoy Admin UI(9901) 접속 불가

## 즉각 확인
1. docker compose ps envoy
2. docker compose logs envoy --tail=100
3. 업스트림 서비스 직접 테스트:
   grpcurl -plaintext localhost:50051 user.v1.UserService/Login

## 복구 절차
1. docker compose restart envoy
2. Envoy 설정 문법 오류 확인:
   docker compose run --rm envoy envoy --config-path /etc/envoy/envoy.yaml --mode validate
```

---

## 14. 주차별 상세 작업 + 완료 조건

| 주차 | 작업 내용 | 완료 조건 |
|------|---------|---------|
| **W1** | 공통 로그/메트릭 인터셉터 작성 + A~D 배포 | A~D 서비스 모두에서 `trace_id` 누락률 0%, `api_request_total` 메트릭 수집 확인 |
| W2 | 매장 승인/감사로그 API 구현 | 모든 관리자 action에 audit_log 적재 확인 |
| W3 | KPI 파이프라인 구현 | 일/주 단위 지표 계산 쿼리 검증 통과 |
| W4 | Grafana 대시보드 1차 (핵심 KPI 3종) | 예약성사율/알림실패율/API응답시간 시각화 |
| W5 | 알람 규칙 파일 적용 | 테스트 이벤트로 알람 트리거 발생 확인 |
| W6 | Runbook 5개 작성 + 온콜 연동 | 팀원 리뷰 후 runbook 폴더에 merge |
| W7 | GameDay (장애 시나리오 실습) | MTTD ≤ 5분, MTTA ≤ 10분 달성 |
| W8~W9 | 알람 오탐 튜닝 + RC | 오탐률 < 5%, 미탐 0건 |
| W10 | 주간 운영리포트 자동 생성 | Grafana scheduled report 또는 스크립트 자동화 |

---

## 15. 교차 점검 항목

| 교차 | 점검 내용 | 검증 방법 |
|------|---------|---------|
| E↔A/B/C/D | 공통 로그 7개 필드 미준수 PR merge 금지 | CI에서 log format lint 자동 체크 (선택) |
| E↔C | 예약 실패 사유 코드가 대시보드 분류체계와 일치 | C 담당자와 에러 코드 목록 사전 합의 (W1 월요일) |
| E↔D | 알림 실패 이벤트가 1분 이내 집계에 반영 | Prometheus에서 `notification_sent_total` 실시간 확인 |
| E↔전원 | 모든 서비스의 `api_request_total` 메트릭 수집 확인 | Prometheus Targets 페이지에서 모든 서비스 UP 상태 |

---

## 16. 상세 테스트 케이스 목록

### ApproveRestaurant (E-FN-01)
- [ ] SUPER_ADMIN 역할로 승인 → `ACTIVE` 상태로 변경
- [ ] SUPER_ADMIN 역할로 거절 → `REJECTED` 상태 + reason 저장
- [ ] OWNER 역할로 승인 시도 → `PERMISSION_DENIED`
- [ ] 이미 ACTIVE인 매장 재승인 → `FAILED_PRECONDITION`

### WriteAuditLog (E-FN-02)
- [ ] 필수 필드 모두 포함 → `log_id` 반환
- [ ] `actor_id` 누락 → `INVALID_ARGUMENT`
- [ ] 동일 `request_id`로 2번 호출 → 2개 로그 모두 저장 (멱등성 없음, 의도된 것)

### QueryAuditLogs (E-FN-03)
- [ ] 날짜 범위 필터 적용 → 해당 기간 로그만 반환
- [ ] actor_id 필터 → 해당 유저 액션만 반환
- [ ] 30일 초과 조회 → `FAILED_PRECONDITION`
- [ ] 커서 페이지네이션 → 다음 페이지 토큰 반환 및 다음 페이지 조회

### 공통 인터셉터 (E-FN-06/07)
- [ ] A~D 모든 서비스에서 `api_request_total` 메트릭 노출 확인
- [ ] 에러 발생 시 `api_error_total` 증가 확인
- [ ] 모든 요청 로그에 `request_id`, `trace_id` 포함 확인
- [ ] 로그 JSON 형식 유효성 검사

---

## 17. 정량 목표 (SLO/품질)

| 지표 | 목표값 |
|------|--------|
| 장애 감지 MTTD | ≤ 5분 |
| 알람 미탐 | 0건 |
| 알람 오탐률 | < 5% |
| Runbook 최신화율 | 100% |
| 감사로그 누락률 | 0% (관리자 액션 기준) |
| trace_id 누락률 | 0% (전 서비스 기준) |
