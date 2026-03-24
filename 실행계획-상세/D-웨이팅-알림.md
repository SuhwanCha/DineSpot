# D 담당 문서 — 웨이팅/알림

> 이 문서만 보고 개발할 수 있도록 작성되었습니다.
> Proto 스키마 전체는 `00-공용-proto-스키마.md` 섹션 5-6, 5-7을 참고하세요.

---

## 0. 담당 서비스 & 디렉토리

### 담당 컨테이너

| 컨테이너명 | gRPC 포트 | 메트릭 포트 | 설명 |
|-----------|-----------|------------|------|
| `dinespot-waiting-service` | 50054 | 9103 | 웨이팅 순번/호출/SSE 스트리밍 |
| `dinespot-notification-service` | 50056 | 9105 | 알림 발송/재시도/중복방지 |

### 담당 디렉토리

```
backend/
├── internal/
│   ├── waiting/                     ← waiting-service
│   │   ├── handler.go               # gRPC 핸들러 (WaitingServiceServer 구현)
│   │   ├── service.go               # 비즈니스 로직
│   │   ├── repository.go            # DB 접근 계층
│   │   ├── sse.go                   # SSE 브로드캐스트 (gRPC Server Streaming)
│   │   ├── queue.go                 # Redis 큐 연산 래퍼
│   │   ├── noshow_timer.go          # 10분 노쇼 타이머 (Redis keyspace notification)
│   │   ├── event_consumer.go        # C의 reservation.canceled.v1 이벤트 소비
│   │   └── handler_test.go
│   └── notification/                ← notification-service
│       ├── handler.go               # gRPC 핸들러 (NotificationServiceServer 구현)
│       ├── service.go               # 발송 로직 + 재시도 + 중복방지
│       ├── repository.go            # notification_logs DB 접근
│       ├── sender_kakao.go          # 카카오 알림톡 발송
│       ├── sender_sms.go            # SMS 발송 (fallback)
│       ├── sender_email.go          # 이메일 발송 (fallback)
│       └── handler_test.go
└── pkg/
    └── redis/
        └── queue.go                 ← Sorted Set 큐 연산 (내가 작성)

infra/
└── mysql/
    └── init/
        ├── 05_waiting.sql           ← 웨이팅 관련 DDL
        └── 06_notifications.sql     ← 알림 로그 DDL
```

### docker-compose 서비스 스니펫

```yaml
waiting-service:
  build:
    context: ./backend
    dockerfile: Dockerfile
    target: service
  container_name: dinespot-waiting-service
  command: ["/app/server", "--service=waiting", "--port=50054"]
  ports:
    - "50054:50054"
    - "9103:9103"
  environment:
    SERVICE_NAME: waiting-service
    DB_DSN: "dinespot:dinespot_pass@tcp(mysql:3306)/dinespot?parseTime=true&loc=Asia%2FSeoul"
    REDIS_ADDR: "redis:6379"
    NOTIFICATION_SERVICE_ADDR: "notification-service:50056"
    NOSHOW_TTL: "10m"
    SSE_HEARTBEAT_INTERVAL: "15s"
    # Redis keyspace notification 활성화 필요 (아래 주석 참고)
    LOG_LEVEL: "info"
  depends_on:
    mysql:
      condition: service_healthy
    redis:
      condition: service_healthy
    notification-service:
      condition: service_started
  networks: [dinespot-net]

notification-service:
  build:
    context: ./backend
    dockerfile: Dockerfile
    target: service
  container_name: dinespot-notification-service
  command: ["/app/server", "--service=notification", "--port=50056"]
  ports:
    - "50056:50056"
    - "9105:9105"
  environment:
    SERVICE_NAME: notification-service
    DB_DSN: "dinespot:dinespot_pass@tcp(mysql:3306)/dinespot?parseTime=true&loc=Asia%2FSeoul"
    REDIS_ADDR: "redis:6379"
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
    LOG_LEVEL: "info"
  depends_on:
    mysql:
      condition: service_healthy
    redis:
      condition: service_healthy
  networks: [dinespot-net]
```

### 로컬 개발 명령어

```bash
# waiting + notification + 의존 서비스 전체 시작
docker compose up -d mysql redis notification-service waiting-service

# mailhog 포함 (이메일 알림 UI로 확인)
docker compose --profile dev up -d mailhog notification-service waiting-service

# 로그 확인
docker compose logs -f waiting-service notification-service

# 재빌드
docker compose up -d --build waiting-service notification-service

# 테스트 실행
cd backend && go test ./internal/waiting/... -v
cd backend && go test ./internal/notification/... -v

# Redis keyspace notification 수동 활성화 (컨테이너 실행 후)
docker compose exec redis redis-cli config set notify-keyspace-events "Ex"

# 웨이팅 등록 테스트
grpcurl -plaintext \
  -H "authorization: Bearer {token}" \
  -d '{"restaurant_id":"rest_xyz","customer_id":"usr_abc","party_size":2,"idempotency_key":"test-key-001"}' \
  localhost:50054 waiting.v1.WaitingService/JoinWaiting

# 알림 발송 테스트 (mailhog에서 확인: http://localhost:8025)
grpcurl -plaintext \
  -H "x-internal-secret: ${INTERNAL_SECRET}" \
  -d '{
    "channel": "NOTIFICATION_CHANNEL_EMAIL",
    "template_id": "RESERVATION_CONFIRMED",
    "recipient": "test@example.com",
    "variables": {"restaurant_name":"맛있는 식당","reserved_at":"2026-03-25 19:00"},
    "dedupe_key": "RESERVATION_CONFIRMED:rsv_abc:2026-03-24"
  }' \
  localhost:50056 notification.v1.NotificationService/SendNotification
```

---

## 1. 역할 목표

- 웨이팅 순번은 언제나 단조 증가 규칙으로 관리되어야 한다. (Redis INCR 원자 연산)
- 알림은 중복/누락 없이 전달되어야 한다. (dedupe_key + 재시도 정책)
- C의 예약 취소 이벤트를 소비하여 웨이팅 자동 승격을 처리한다.

---

## 2. 기능 요구사항 (FR)

### 웨이팅 서비스
| ID | 요구사항 | 우선순위 |
|----|---------|---------|
| FR-D-01 | 웨이팅 등록 (순번 발급, 앞 대기팀 수, 예상 대기시간 반환) | P0 |
| FR-D-02 | 순번 조회 | P0 |
| FR-D-03 | 웨이팅 호출 (`CallNext`) — 최솟값 순번 고객에게 입장 알림 | P0 |
| FR-D-04 | 고객 직접 취소 | P0 |
| FR-D-05 | 입장 확인 (`ConfirmSeated`) | P0 |
| FR-D-06 | 10분 무응답 자동 노쇼 처리 + 다음 순번 자동 호출 | P0 |
| FR-D-07 | 실시간 순번 브로드캐스트 (gRPC Server Streaming → Envoy SSE) | P1 |
| FR-D-08 | 예약 취소 이벤트(`reservation.canceled.v1`) 소비 → 웨이팅 승격 | P0 |

### 알림 서비스
| ID | 요구사항 | 우선순위 |
|----|---------|---------|
| FR-D-09 | 카카오 알림톡 발송 (1순위) | P0 |
| FR-D-10 | SMS 발송 (카카오 실패 시 fallback) | P0 |
| FR-D-11 | 이메일 발송 (SMS 실패 시 fallback) | P1 |
| FR-D-12 | `dedupe_key` 기반 중복 발송 방지 | P0 |
| FR-D-13 | 재시도 정책 (max 3회, exponential backoff) | P1 |
| FR-D-14 | 발송 이력 로그 저장 + 조회 | P1 |
| FR-D-15 | `notification_opt_out=true`인 유저에게 마케팅 알림 발송 금지 | P1 |

---

## 3. API/함수 구현 명세

### 웨이팅 서비스
| ID | 함수 | Input | Output | 실패 조건 |
|----|------|-------|--------|----------|
| D-FN-01 | `JoinWaiting(ctx, req)` | `restaurant_id, customer_id, party_size, idempotency_key` | `waiting_id, queue_no, ahead_count, eta_min` | 영업종료 → `FAILED_PRECONDITION` |
| D-FN-02 | `CallNext(ctx, req)` | `restaurant_id, operator_id` | `called_waiting_id, queue_no, ttl_min` | 큐 비어있음 → `NOT_FOUND` |
| D-FN-03 | `CancelWaiting(ctx, req)` | `waiting_id, reason` | `status=CANCELLED` | 권한 없음 → `PERMISSION_DENIED` |
| D-FN-04 | `ConfirmSeated(ctx, req)` | `waiting_id, customer_id` | `status=SEATED` | READY 상태 아님 → `FAILED_PRECONDITION` |
| D-FN-05 | `StreamQueueUpdates(req)` | `restaurant_id` | `stream<QueueUpdateEvent>` | 연결 오류 → `UNAVAILABLE` |

### 알림 서비스
| ID | 함수 | Input | Output | 실패 조건 |
|----|------|-------|--------|----------|
| D-FN-06 | `SendNotification(ctx, req)` | `channel, template_id, recipient, variables, dedupe_key` | `message_id, status, sent_at` | 중복키 → `status=SKIPPED` |
| D-FN-07 | `GetNotificationLog(ctx, req)` | `dedupe_key` | `message_id, status, error_msg, sent_at` | 존재하지 않음 → `NOT_FOUND` |

---

## 4. DB 스키마

### 파일 위치: `infra/mysql/init/05_waiting.sql`

```sql
-- 웨이팅 항목
CREATE TABLE IF NOT EXISTS waiting_entries (
  id              CHAR(36)    NOT NULL,
  restaurant_id   CHAR(36)    NOT NULL,
  customer_id     CHAR(36)    NOT NULL,
  queue_no        INT         NOT NULL,
  party_size      INT         NOT NULL,
  status          ENUM('WAITING','READY','SEATED','NO_SHOW','CANCELLED') NOT NULL DEFAULT 'WAITING',
  idempotency_key VARCHAR(128) DEFAULT NULL,
  registered_at   DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  notified_at     DATETIME    DEFAULT NULL,   -- CallNext 시각
  seated_at       DATETIME    DEFAULT NULL,   -- ConfirmSeated 시각
  created_at      DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at      DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uq_we_idempotency (idempotency_key),
  KEY idx_we_restaurant_status (restaurant_id, status),
  KEY idx_we_customer_id (customer_id),
  KEY idx_we_queue_no (restaurant_id, queue_no),
  CONSTRAINT fk_we_restaurant FOREIGN KEY (restaurant_id) REFERENCES restaurants(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 파일 위치: `infra/mysql/init/06_notifications.sql`

```sql
-- 알림 발송 이력
CREATE TABLE IF NOT EXISTS notification_logs (
  id            CHAR(36)     NOT NULL,
  dedupe_key    VARCHAR(128) NOT NULL,
  channel       ENUM('KAKAO','SMS','EMAIL') NOT NULL,
  template_id   VARCHAR(100) NOT NULL,
  recipient     VARCHAR(200) NOT NULL,
  variables     JSON         DEFAULT NULL,
  status        ENUM('SENT','FAILED','SKIPPED') NOT NULL,
  error_msg     VARCHAR(500) DEFAULT NULL,
  retry_count   INT          NOT NULL DEFAULT 0,
  sent_at       DATETIME     DEFAULT NULL,
  created_at    DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uq_notif_dedupe (dedupe_key),
  KEY idx_notif_status (status),
  KEY idx_notif_created_at (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 5. Redis 키 패턴 및 큐 설계

| 키 | TTL | 설명 |
|----|-----|------|
| `waiting:seq:{restaurant_id}` | 영구 (당일 자정 리셋 크론) | 순번 카운터 (INCR) |
| `waiting:queue:{restaurant_id}` | 영구 | Sorted Set (score=queue_no, member=waiting_id) |
| `waiting:noshow_timer:{waiting_id}` | 10분 | 노쇼 TTL 키 (만료 시 keyspace notification) |
| `notif:dedupe:{dedupe_key}` | 24시간 | 알림 중복 방지 (SET 후 TTL 확인) |

```go
// backend/pkg/redis/queue.go

// 웨이팅 큐 관련 키 생성
func WaitingSeqKey(restaurantID string) string {
    return fmt.Sprintf("waiting:seq:%s", restaurantID)
}

func WaitingQueueKey(restaurantID string) string {
    return fmt.Sprintf("waiting:queue:%s", restaurantID)
}

func NoShowTimerKey(waitingID string) string {
    return fmt.Sprintf("waiting:noshow_timer:%s", waitingID)
}

func NotifDedupeKey(dedupeKey string) string {
    return fmt.Sprintf("notif:dedupe:%s", dedupeKey)
}

// 순번 발급 (원자적 INCR, 중복 절대 불가)
func IssueQueueNo(ctx context.Context, client *goredis.Client, restaurantID string) (int64, error) {
    return client.Incr(ctx, WaitingSeqKey(restaurantID)).Result()
}

// 큐에 추가 (Sorted Set, score=queue_no)
func EnqueueWaiting(ctx context.Context, client *goredis.Client, restaurantID, waitingID string, queueNo int64) error {
    return client.ZAdd(ctx, WaitingQueueKey(restaurantID), goredis.Z{
        Score:  float64(queueNo),
        Member: waitingID,
    }).Err()
}

// 큐에서 최솟값(가장 먼저 온 사람) 조회
func PeekFront(ctx context.Context, client *goredis.Client, restaurantID string) (string, error) {
    result, err := client.ZRange(ctx, WaitingQueueKey(restaurantID), 0, 0).Result()
    if err != nil || len(result) == 0 {
        return "", fmt.Errorf("queue is empty")
    }
    return result[0], nil
}

// 큐에서 제거 (SEATED, NO_SHOW, CANCELLED 시)
func DequeueWaiting(ctx context.Context, client *goredis.Client, restaurantID, waitingID string) error {
    return client.ZRem(ctx, WaitingQueueKey(restaurantID), waitingID).Err()
}

// 앞 대기 팀 수 조회
func GetAheadCount(ctx context.Context, client *goredis.Client, restaurantID, waitingID string) (int64, error) {
    rank, err := client.ZRank(ctx, WaitingQueueKey(restaurantID), waitingID).Result()
    if err != nil {
        return 0, err
    }
    return rank, nil   // 0-indexed, 그대로가 "앞에 있는 팀 수"
}
```

---

## 6. 웨이팅 등록 처리 흐름 (`JoinWaiting`)

```
FE                          waiting-service              Redis              MySQL
│                               │                          │                  │
│  JoinWaiting(req)             │                          │                  │
│───────────────────────────────▶                          │                  │
│                               │  멱등키 확인              │                  │
│                               │  (있으면 기존 결과 반환)  │                  │
│                               │                          │                  │
│                               │  INCR waiting:seq:{rid}  │                  │
│                               │─────────────────────────▶                  │
│                               │  queue_no 반환           │                  │
│                               │◀─────────────────────────                  │
│                               │                          │                  │
│                               │  ZADD waiting:queue:{rid}│                  │
│                               │  score=queue_no          │                  │
│                               │─────────────────────────▶                  │
│                               │                          │                  │
│                               │  INSERT waiting_entries  │                  │
│                               │─────────────────────────────────────────────▶
│                               │                          │                  │
│                               │  ZRANK → ahead_count     │                  │
│                               │─────────────────────────▶                  │
│                               │                          │                  │
│  {waiting_id, queue_no,        │                          │                  │
│   ahead_count, eta_min}        │                          │                  │
│◀───────────────────────────────                          │                  │
│                               │  브로드캐스트 이벤트 발행 │                  │
│                               │  (SSE 구독 클라이언트들)  │                  │
```

**예상 대기 시간 계산:**
```
eta_min = ahead_count × avg_seat_duration_min

avg_seat_duration_min은 가장 작은 slot_duration_min을 B에서 조회
(간단 구현: 기본값 20분 사용 → 추후 실제 통계로 개선)
```

---

## 7. 노쇼 타이머 구현 (10분)

`CallNext()` 실행 시 `waiting:noshow_timer:{waiting_id}` 키를 10분 TTL로 SET합니다.
Redis의 **Keyspace Notification** 기능을 사용하여 키 만료 시 자동으로 노쇼 처리합니다.

```go
// noshow_timer.go

// 1. Redis 서버에서 keyspace notification 활성화 필요:
//    redis-cli config set notify-keyspace-events "Ex"
//    (docker-compose의 redis command에 추가하거나 시작 스크립트에서 실행)

// 2. 노쇼 타이머 설정 (CallNext 후 호출)
func (s *Service) setNoShowTimer(ctx context.Context, waitingID string) error {
    key := redis.NoShowTimerKey(waitingID)
    return s.redis.Set(ctx, key, waitingID, 10*time.Minute).Err()
}

// 3. 만료 이벤트 구독 및 처리
func (s *Service) subscribeNoShowTimers(ctx context.Context) {
    pubsub := s.redis.PSubscribe(ctx, "__keyevent@0__:expired")
    defer pubsub.Close()

    for msg := range pubsub.Channel() {
        key := msg.Payload
        // "waiting:noshow_timer:" 접두사 확인
        if !strings.HasPrefix(key, "waiting:noshow_timer:") {
            continue
        }
        waitingID := strings.TrimPrefix(key, "waiting:noshow_timer:")
        if err := s.processNoShow(ctx, waitingID); err != nil {
            s.logger.Error("no-show processing failed", "waiting_id", waitingID, "error", err)
        }
    }
}

// 4. 노쇼 처리 실행
func (s *Service) processNoShow(ctx context.Context, waitingID string) error {
    entry, err := s.repo.GetWaitingEntry(ctx, waitingID)
    if err != nil {
        return err
    }
    // READY 상태인 경우만 처리 (이미 SEATED면 타이머 무시)
    if entry.Status != "READY" {
        return nil
    }

    // 상태 업데이트 → NO_SHOW
    if err := s.repo.UpdateStatus(ctx, waitingID, "NO_SHOW"); err != nil {
        return err
    }

    // Redis Sorted Set에서 제거
    redis.DequeueWaiting(ctx, s.redis, entry.RestaurantID, waitingID)

    // 고객에게 노쇼 알림
    s.notifyCustomer(entry.CustomerID, "WAITING_NO_SHOW", map[string]string{
        "queue_no": fmt.Sprintf("%d", entry.QueueNo),
    })

    // 다음 순번 자동 호출
    go s.autoCallNext(ctx, entry.RestaurantID)

    // SSE 브로드캐스트
    s.broadcast(entry.RestaurantID)

    return nil
}
```

---

## 8. SSE 브로드캐스트 (`StreamQueueUpdates`)

gRPC Server Streaming을 사용합니다. Envoy가 이를 HTTP SSE로 변환하여 브라우저에 전달합니다.

```go
// sse.go
package waiting

import (
    "sync"
    waitingv1 "github.com/dinespot/backend/internal/gen/waiting/v1"
)

// 레스토랑별 구독자 관리
type BroadcastHub struct {
    mu          sync.RWMutex
    subscribers map[string][]chan *waitingv1.QueueUpdateEvent   // key: restaurant_id
}

var hub = &BroadcastHub{
    subscribers: make(map[string][]chan *waitingv1.QueueUpdateEvent),
}

// StreamQueueUpdates gRPC 핸들러
func (h *Handler) StreamQueueUpdates(
    req *waitingv1.StreamQueueRequest,
    stream waitingv1.WaitingService_StreamQueueUpdatesServer,
) error {
    ch := make(chan *waitingv1.QueueUpdateEvent, 10)
    hub.subscribe(req.RestaurantId, ch)
    defer hub.unsubscribe(req.RestaurantId, ch)

    ctx := stream.Context()
    heartbeat := time.NewTicker(15 * time.Second)
    defer heartbeat.Stop()

    for {
        select {
        case <-ctx.Done():
            return nil
        case <-heartbeat.C:
            // 연결 유지를 위한 heartbeat (빈 이벤트)
            // 실제로는 현재 큐 상태 스냅샷 전송
            snapshot := h.svc.GetQueueSnapshot(ctx, req.RestaurantId)
            if err := stream.Send(snapshot); err != nil {
                return err
            }
        case event := <-ch:
            if err := stream.Send(event); err != nil {
                return err
            }
        }
    }
}

// 내부에서 이벤트 발행 시 호출
func (hub *BroadcastHub) Broadcast(restaurantID string, event *waitingv1.QueueUpdateEvent) {
    hub.mu.RLock()
    defer hub.mu.RUnlock()

    for _, ch := range hub.subscribers[restaurantID] {
        select {
        case ch <- event:
        default:
            // 채널 가득 찬 경우 드랍 (클라이언트 재연결 유도)
        }
    }
}
```

---

## 9. 예약 취소 이벤트 소비 (C → D 연동)

C가 `events:reservation.canceled.v1` Redis Stream에 이벤트를 발행하면, D가 이를 소비합니다.

```go
// event_consumer.go
package waiting

func (s *Service) StartEventConsumer(ctx context.Context) {
    go s.consumeReservationCanceledEvents(ctx)
}

func (s *Service) consumeReservationCanceledEvents(ctx context.Context) {
    streamKey := "events:reservation.canceled.v1"
    consumerGroup := "waiting-service"
    consumerName := "waiting-consumer-1"

    // Consumer Group 생성 (이미 있으면 무시)
    s.redis.XGroupCreateMkStream(ctx, streamKey, consumerGroup, "$")

    for {
        results, err := s.redis.XReadGroup(ctx, &goredis.XReadGroupArgs{
            Group:    consumerGroup,
            Consumer: consumerName,
            Streams:  []string{streamKey, ">"},
            Count:    10,
            Block:    5 * time.Second,
        }).Result()

        if err != nil {
            if errors.Is(err, goredis.Nil) {
                continue   // timeout, 다시 시도
            }
            s.logger.Error("event consumer error", "error", err)
            time.Sleep(1 * time.Second)
            continue
        }

        for _, stream := range results {
            for _, msg := range stream.Messages {
                if err := s.handleReservationCanceled(ctx, msg); err != nil {
                    s.logger.Error("failed to handle event", "msg_id", msg.ID, "error", err)
                    // NACK: 재처리 대기 (pending list에 유지)
                } else {
                    // ACK: 처리 완료
                    s.redis.XAck(ctx, streamKey, consumerGroup, msg.ID)
                }
            }
        }
    }
}

func (s *Service) handleReservationCanceled(ctx context.Context, msg goredis.XMessage) error {
    var event ReservationCanceledEvent
    if err := json.Unmarshal([]byte(msg.Values["data"].(string)), &event); err != nil {
        return err
    }

    // 해당 레스토랑의 WAITING 상태 중 가장 앞번호 웨이팅 승격
    // (예약이 취소되었으니 자리가 생겼음을 암시)
    frontWaitingID, err := redis.PeekFront(ctx, s.redis, event.RestaurantID)
    if err != nil {
        return nil   // 큐가 비어있으면 정상
    }

    // 3초 이내에 승격 이벤트 발행 (D↔C 교차 점검 목표)
    return s.promoteToCalled(ctx, frontWaitingID, event.RestaurantID)
}
```

---

## 10. 알림 발송 구조 (notification-service)

### 채널 우선순위 (fallback 체인)

```
1순위: KAKAO (카카오 알림톡)
  ↓ 실패 (API 오류, 카카오 미가입 등)
2순위: SMS
  ↓ 실패
3순위: EMAIL
  ↓ 3회 모두 실패
결과: status=FAILED, error_msg 기록
```

### 알림 템플릿 목록 및 변수

| template_id | 수신자 | 필수 변수 |
|-------------|--------|---------|
| `RESERVATION_REQUESTED` | 점주 | `customer_name`, `reserved_at`, `party_size` |
| `RESERVATION_CONFIRMED` | 고객 | `restaurant_name`, `reserved_at`, `party_size` |
| `RESERVATION_REJECTED` | 고객 | `restaurant_name`, `rejection_reason` |
| `RESERVATION_CANCELLED` | 상대방 | `restaurant_name`, `reserved_at`, `cancel_reason` |
| `RESERVATION_REMINDER` | 고객 | `restaurant_name`, `reserved_at`, `address` |
| `WAITING_REGISTERED` | 고객 | `restaurant_name`, `queue_no`, `ahead_count`, `eta_min` |
| `WAITING_CALLED` | 고객 | `restaurant_name`, `queue_no`, `ttl_min` |
| `WAITING_NO_SHOW` | 고객 | `restaurant_name`, `queue_no` |
| `ORDER_CONFIRMED` | 고객 | `restaurant_name`, `order_id`, `total_amount` |
| `ORDER_PAYMENT_FAILED` | 고객 | `order_id`, `payment_url` |

### `dedupe_key` 생성 규칙

```
규칙: "{template_id}:{primary_id}:{date}"

예시:
- 예약 확정: "RESERVATION_CONFIRMED:rsv_abc123:2026-03-24"
- 웨이팅 호출: "WAITING_CALLED:wait_xyz:2026-03-24"
- 결제 실패: "ORDER_PAYMENT_FAILED:ord_def:2026-03-24"

같은 날 같은 예약에 대해 중복 발송 방지
TTL: 24시간 (Redis key: notif:dedupe:{dedupe_key})
```

### 재시도 정책

```
시도 횟수    대기 시간
1회          즉시
2회          2초 후
3회          4초 후  (backoff_base × 2^(attempt-1))
3회 초과     포기, status=FAILED
```

```go
// service.go — SendNotification with retry
func (s *Service) SendNotification(ctx context.Context, req *notificationv1.SendNotificationRequest) (*notificationv1.SendNotificationResponse, error) {
    // 1. 중복 체크
    dedupeKey := redis.NotifDedupeKey(req.DedupeKey)
    if exists, _ := s.redis.Exists(ctx, dedupeKey).Result(); exists > 0 {
        // 이미 발송됨, SKIPPED 반환
        return &notificationv1.SendNotificationResponse{
            Status: notificationv1.NotificationStatus_NOTIFICATION_STATUS_SKIPPED,
        }, nil
    }

    // 2. 채널 순서대로 fallback 시도
    channels := []NotificationChannel{
        {Type: "KAKAO", Sender: s.kakaoSender},
        {Type: "SMS",   Sender: s.smsSender},
        {Type: "EMAIL", Sender: s.emailSender},
    }

    var lastErr error
    for attempt := 0; attempt < s.cfg.RetryMaxCount; attempt++ {
        channel := channels[attempt % len(channels)]
        if attempt > 0 {
            backoff := s.cfg.RetryBackoffBase * time.Duration(1<<uint(attempt-1))
            if backoff > s.cfg.RetryBackoffMax {
                backoff = s.cfg.RetryBackoffMax
            }
            time.Sleep(backoff)
        }

        messageID, err := channel.Sender.Send(ctx, req.Recipient, req.TemplateId, req.Variables)
        if err != nil {
            lastErr = err
            s.logger.Warn("notification send failed, retrying",
                "channel", channel.Type,
                "attempt", attempt+1,
                "error", err)
            continue
        }

        // 3. 성공 → dedupe 등록 + DB 저장
        s.redis.Set(ctx, dedupeKey, messageID, s.cfg.DedupeTTL)
        s.repo.SaveLog(ctx, req, messageID, "SENT", attempt+1, "")
        return &notificationv1.SendNotificationResponse{
            MessageId: messageID,
            Status:    notificationv1.NotificationStatus_NOTIFICATION_STATUS_SENT,
            SentAt:    time.Now().Format(time.RFC3339),
        }, nil
    }

    // 4. 모두 실패
    s.repo.SaveLog(ctx, req, "", "FAILED", s.cfg.RetryMaxCount, lastErr.Error())
    return &notificationv1.SendNotificationResponse{
        Status: notificationv1.NotificationStatus_NOTIFICATION_STATUS_FAILED,
    }, nil
}
```

---

## 11. 이벤트 발행 (D 내부, Redis Streams)

| 이벤트 | Stream 키 | 소비자 |
|--------|-----------|--------|
| `waiting.joined.v1` | `events:waiting.joined.v1` | E (메트릭 수집) |
| `waiting.called.v1` | `events:waiting.called.v1` | E (메트릭 수집) |
| `waiting.canceled.v1` | `events:waiting.canceled.v1` | E (메트릭 수집) |

각 이벤트 페이로드는 `00-공용-proto-스키마.md` 섹션 7을 참고하세요.

---

## 12. 주차별 상세 작업 + 완료 조건

| 주차 | 작업 내용 | 완료 조건 |
|------|---------|---------|
| W1 | 큐 모델 + 순번 계산 + DB 스키마 | 1~N 순번 재정렬(앞 취소 후 뒤 번호 안 바뀜) 테스트 통과 |
| W2 | 등록/호출/취소/입장확인 API | `CallNext` 시 Sorted Set 최솟값 선택 보장 테스트 |
| W3 | SSE 브로드캐스트 + notification 기본 발송 | SSE 연결 1,000개 기준 이벤트 지연 1초 이내 |
| W4 | 알림 재시도/중복방지/fallback 체인 | dedupe_key 중복 발송 0건, 3채널 fallback 테스트 통과 |
| W5 | C 취소 이벤트 소비 + 웨이팅 자동 승격 | 예약 취소 이벤트 수신 후 **3초 이내** 승격 이벤트 발행 |
| W6 | SSE 단절/재연결 처리 + heartbeat | 재연결 후 최신 큐 상태 스냅샷 동기화 성공 |
| W7~W9 | 성능/회귀/UAT | 순번 정합성 오류 0건, 알림 중복 < 0.1% |
| W10 | 운영 고도화 | 알림 실패 대시보드 실시간 반영 |

---

## 13. 교차 점검 항목

| 교차 | 점검 내용 | 검증 방법 |
|------|---------|---------|
| D↔C | `reservation.canceled.v1` 수신 후 **3초 이내** 승격 이벤트 발행 | 통합 테스트: C에서 취소 발행 → D에서 3초 내 `waiting.called.v1` 발행 확인 |
| D↔A | `notification_opt_out=true` 유저에게 마케팅 알림 발송 금지 | A에서 user 정보 조회 후 opt_out 체크 로직 테스트 |
| D↔E | `notification.fail.rate` 메트릭 수집 확인 | E의 Prometheus에서 `notification_sent_total{status="FAILED"}` 증가 확인 |
| D↔E | 웨이팅 이벤트 3종이 E 집계에 1분 이내 반영 | E 대시보드에서 이벤트 수 실시간 확인 |

---

## 14. 상세 테스트 케이스 목록

### JoinWaiting (D-FN-01)
- [ ] 정상 등록 → queue_no 발급 (단조 증가)
- [ ] 100명 동시 등록 → queue_no 중복 없음 (INCR 원자성)
- [ ] 멱등키 재시도 → 동일 queue_no 반환
- [ ] 영업종료 후 등록 시도 → `FAILED_PRECONDITION`
- [ ] eta_min 계산 정확성 (ahead_count × avg_duration)

### CallNext (D-FN-02)
- [ ] 정상 호출 → 최솟값 queue_no 고객에게 알림
- [ ] 빈 큐에서 호출 → `NOT_FOUND`
- [ ] 호출 후 noshow_timer 키 생성 확인 (TTL=10분)
- [ ] 권한 없는 유저(CUSTOMER) 호출 시도 → `PERMISSION_DENIED`

### 노쇼 처리
- [ ] CallNext 후 10분 이내 ConfirmSeated → SEATED, 타이머 해제
- [ ] CallNext 후 10분 경과 → 자동 NO_SHOW + 다음 순번 자동 호출
- [ ] NO_SHOW 후 SSE 브로드캐스트 확인

### SendNotification (D-FN-06)
- [ ] 카카오 정상 발송 → SENT
- [ ] 카카오 실패 → SMS fallback → SENT
- [ ] 카카오 + SMS 실패 → Email fallback → SENT
- [ ] 모두 실패 3회 → FAILED, DB에 error_msg 기록
- [ ] 동일 dedupe_key 재발송 → SKIPPED (DB 조회 없이 Redis로 판단)
- [ ] dedupe_key 24시간 후 만료 → 재발송 가능

---

## 15. 정량 목표 (SLO/품질)

| 지표 | 목표값 |
|------|--------|
| 실시간 이벤트 지연 p95 | < 1초 |
| 알림 중복 발송률 | < 0.1% |
| 웨이팅 순번 정합성 오류 | 0건 |
| 예약 취소 → 웨이팅 승격 지연 | < 3초 |
| SSE 연결 지원 동시 수 | ≥ 1,000개 |
