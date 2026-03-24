# C 담당 문서 — 예약 코어

> 이 문서만 보고 개발할 수 있도록 작성되었습니다.
> Proto 스키마 전체는 `00-공용-proto-스키마.md` 섹션 5-5를 참고하세요.

---

## 0. 담당 서비스 & 디렉토리

### 담당 컨테이너

| 컨테이너명 | gRPC 포트 | 메트릭 포트 | 설명 |
|-----------|-----------|------------|------|
| `dinespot-reservation-service` | 50053 | 9102 | 예약 생성/조회/변경/취소 |

### 담당 디렉토리

```
backend/
├── internal/
│   └── reservation/                 ← 핵심 구현
│       ├── handler.go               # gRPC 핸들러 (ReservationServiceServer 구현)
│       ├── service.go               # 비즈니스 로직 (슬롯 계산, 상태 전이 조율)
│       ├── repository.go            # DB 접근 계층
│       ├── statemachine.go          # 상태 머신 (전이 규칙 + 유효성 검사)
│       ├── slot_calculator.go       # 가용 슬롯 계산 (B 데이터 기반)
│       ├── noshow_monitor.go        # 노쇼 체크 크론 (방문 시간 +60분 후)
│       └── handler_test.go
└── pkg/
    └── redis/
        └── lock.go                  ← 분산 락 구현 (내가 작성)

infra/
└── mysql/
    └── init/
        └── 04_reservations.sql      ← 예약 관련 DDL
```

### docker-compose 서비스 스니펫

```yaml
reservation-service:
  build:
    context: ./backend
    dockerfile: Dockerfile
    target: service
  container_name: dinespot-reservation-service
  command: ["/app/server", "--service=reservation", "--port=50053"]
  ports:
    - "50053:50053"
    - "9102:9102"
  environment:
    SERVICE_NAME: reservation-service
    DB_DSN: "dinespot:dinespot_pass@tcp(mysql:3306)/dinespot?parseTime=true&loc=Asia%2FSeoul"
    REDIS_ADDR: "redis:6379"
    USER_SERVICE_ADDR: "user-service:50051"
    RESTAURANT_SERVICE_ADDR: "restaurant-service:50052"
    NOTIFICATION_SERVICE_ADDR: "notification-service:50056"
    RESERVATION_LOCK_TTL: "30s"
    NOSHOW_CHECK_INTERVAL: "5m"
    NOSHOW_GRACE_PERIOD: "60m"
    LOG_LEVEL: "info"
  depends_on:
    mysql:
      condition: service_healthy
    redis:
      condition: service_healthy
    restaurant-service:
      condition: service_started
    notification-service:
      condition: service_started
  networks: [dinespot-net]
```

### 로컬 개발 명령어

```bash
# reservation-service + 의존 서비스 전체 시작
docker compose up -d mysql redis user-service restaurant-service notification-service reservation-service

# 내 서비스 로그 확인
docker compose logs -f reservation-service

# 재빌드
docker compose up -d --build reservation-service

# 테스트 실행
cd backend && go test ./internal/reservation/... -v

# 동시성 테스트 (k6)
k6 run infra/k6/concurrent_reservation.js
# → 100개 동시 요청 중 정확히 1개만 성공해야 함

# gRPC 예약 생성 테스트
grpcurl -plaintext \
  -H "authorization: Bearer {token}" \
  -d '{
    "customer_id": "usr_abc",
    "restaurant_id": "rest_xyz",
    "reserved_at": "2026-03-25T19:00:00+09:00",
    "party_size": 2,
    "idempotency_key": "550e8400-e29b-41d4-a716-446655440000"
  }' \
  localhost:50053 reservation.v1.ReservationService/CreateReservation
```

---

## 1. 역할 목표

- 동일 슬롯/좌석 중복 예약을 원천 차단한다. (Redis 분산 락 + MySQL SELECT FOR UPDATE)
- 생성/변경/취소 전이가 상태 머신으로 강제된다.
- 멱등키를 통해 네트워크 재시도 시 중복 예약이 발생하지 않는다.

---

## 2. 기능 요구사항 (FR)

| ID | 요구사항 | 우선순위 |
|----|---------|---------|
| FR-C-01 | 예약 생성 (슬롯 선점 → PENDING) | P0 |
| FR-C-02 | 예약 조회/목록 (고객, 점주 모두 가능) | P0 |
| FR-C-03 | 예약 변경 (시간/인원 변경) | P1 |
| FR-C-04 | 예약 취소 (방문 1시간 전까지) | P0 |
| FR-C-05 | 슬롯 가용성 조회 | P0 |
| FR-C-06 | 멱등키 기반 재시도 안전성 | P0 |
| FR-C-07 | 점주 예약 확정/거절 | P0 |
| FR-C-08 | 노쇼 처리 (방문 시간 +60분 자동) | P1 |
| FR-C-09 | 예약 상태 변경 시 D(알림)에 이벤트 발행 | P0 |

---

## 3. API/함수 구현 명세

| ID | 함수 | Input | Output | 실패 조건 |
|----|------|-------|--------|----------|
| C-FN-01 | `CreateReservation(ctx, req)` | `customer_id, restaurant_id, reserved_at, party_size, idempotency_key` | `reservation_id, status=PENDING` | 슬롯 충돌 → `ABORTED`, 휴무일 → `FAILED_PRECONDITION` |
| C-FN-02 | `GetAvailability(ctx, req)` | `restaurant_id, date, party_size` | `time_slots[]` | B 참조데이터 없음 → `FAILED_PRECONDITION` |
| C-FN-03 | `UpdateReservation(ctx, req)` | `reservation_id, new_reserved_at, new_party_size` | `status, updated_at` | 상태 전이 불가 → `FAILED_PRECONDITION` |
| C-FN-04 | `CancelReservation(ctx, req)` | `reservation_id, reason` | `status=CANCELLED` | 방문 1시간 이내 → `FAILED_PRECONDITION` |
| C-FN-05 | `ApplyNoShow(ctx, req)` | `reservation_id, operator_id` | `status=NO_SHOW` | CONFIRMED 아님 → `FAILED_PRECONDITION` |
| C-FN-06 | `ConfirmReservation(ctx, req)` | `reservation_id, operator_id` | `status=CONFIRMED` | PENDING 아님 → `FAILED_PRECONDITION` |
| C-FN-07 | `RejectReservation(ctx, req)` | `reservation_id, reason, operator_id` | `status=REJECTED` | PENDING 아님 → `FAILED_PRECONDITION` |

---

## 4. DB 스키마

파일 위치: `infra/mysql/init/04_reservations.sql`

```sql
-- 예약 메인 테이블
CREATE TABLE IF NOT EXISTS reservations (
  id                CHAR(36)     NOT NULL,
  customer_id       CHAR(36)     NOT NULL,
  restaurant_id     CHAR(36)     NOT NULL,
  party_size        INT          NOT NULL,
  reserved_at       DATETIME     NOT NULL,   -- 방문 예정 일시 (KST)
  status            ENUM('PENDING','CONFIRMED','REJECTED','CANCELLED','COMPLETED','NO_SHOW')
                    NOT NULL DEFAULT 'PENDING',
  rejection_reason  VARCHAR(500) DEFAULT NULL,
  cancel_reason     VARCHAR(500) DEFAULT NULL,
  idempotency_key   VARCHAR(128) DEFAULT NULL,
  created_at        DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at        DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uq_reservations_idempotency (idempotency_key),
  KEY idx_reservations_customer (customer_id),
  KEY idx_reservations_restaurant_reserved_at (restaurant_id, reserved_at),
  KEY idx_reservations_status (status),
  -- 동일 슬롯 중복 방지: 같은 restaurant + reserved_at에 PENDING/CONFIRMED가 중복되면 안 됨
  -- 이 유니크 제약 대신 SELECT FOR UPDATE + 비즈니스 로직으로 처리
  CONSTRAINT fk_res_restaurant FOREIGN KEY (restaurant_id) REFERENCES restaurants(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 예약 상태 변경 이력 (감사 + 디버깅용)
CREATE TABLE IF NOT EXISTS reservation_status_history (
  id             CHAR(36)    NOT NULL,
  reservation_id CHAR(36)    NOT NULL,
  from_status    VARCHAR(20) DEFAULT NULL,
  to_status      VARCHAR(20) NOT NULL,
  changed_by     CHAR(36)    NOT NULL,   -- user_id
  reason         VARCHAR(500) DEFAULT NULL,
  created_at     DATETIME    NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_rsh_reservation_id (reservation_id),
  CONSTRAINT fk_rsh_reservation FOREIGN KEY (reservation_id) REFERENCES reservations(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 5. 상태 머신 구현

파일 위치: `backend/internal/reservation/statemachine.go`

```
상태 전이 테이블:

현재 상태       | 이벤트               | 다음 상태  | 처리
----------------|----------------------|------------|--------------------
PENDING         | ConfirmReservation   | CONFIRMED  | 고객 확정 알림 발송
PENDING         | RejectReservation    | REJECTED   | 고객 거절 알림 발송
PENDING         | CancelReservation    | CANCELLED  | 슬롯 락 해제
CONFIRMED       | CancelReservation    | CANCELLED  | 환불 처리, 알림 발송
CONFIRMED       | CompleteReservation  | COMPLETED  | (자동 또는 수동)
CONFIRMED       | ApplyNoShow          | NO_SHOW    | 패널티 부여
(기타)          | (모든 이벤트)        | -          | FAILED_PRECONDITION
```

```go
// backend/internal/reservation/statemachine.go
package reservation

import (
    "fmt"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

type ReservationStatus string

const (
    StatusPending   ReservationStatus = "PENDING"
    StatusConfirmed ReservationStatus = "CONFIRMED"
    StatusRejected  ReservationStatus = "REJECTED"
    StatusCancelled ReservationStatus = "CANCELLED"
    StatusCompleted ReservationStatus = "COMPLETED"
    StatusNoShow    ReservationStatus = "NO_SHOW"
)

type Transition struct {
    From  ReservationStatus
    Event string
    To    ReservationStatus
}

var allowedTransitions = []Transition{
    {StatusPending,   "confirm",  StatusConfirmed},
    {StatusPending,   "reject",   StatusRejected},
    {StatusPending,   "cancel",   StatusCancelled},
    {StatusConfirmed, "cancel",   StatusCancelled},
    {StatusConfirmed, "complete", StatusCompleted},
    {StatusConfirmed, "noshow",   StatusNoShow},
}

// Transition 유효성 검사 — 유효하면 다음 상태 반환, 유효하지 않으면 FAILED_PRECONDITION
func ValidateTransition(current ReservationStatus, event string) (ReservationStatus, error) {
    for _, t := range allowedTransitions {
        if t.From == current && t.Event == event {
            return t.To, nil
        }
    }
    return "", status.Errorf(codes.FailedPrecondition,
        "invalid state transition: %s → event:%s is not allowed", current, event)
}
```

---

## 6. 분산 락 구현

파일 위치: `backend/pkg/redis/lock.go`

**Redis 키 패턴:** `lock:slot:{restaurant_id}:{reserved_at_unix}`
- `reserved_at_unix`: Unix timestamp (초 단위), 슬롯 단위로 내림 (90분 단위면 // 5400 * 5400)
- TTL: 30초

```go
// backend/pkg/redis/lock.go
package redis

import (
    "context"
    "fmt"
    "time"

    goredis "github.com/redis/go-redis/v9"
)

type DistributedLock struct {
    client *goredis.Client
    key    string
    token  string   // 락 소유자 식별 (UUID, 본인만 해제 가능)
    ttl    time.Duration
}

// SlotLockKey: 슬롯 단위로 정렬된 락 키 생성
func SlotLockKey(restaurantID string, reservedAt time.Time, slotDurationMin int) string {
    slotSec := int64(slotDurationMin) * 60
    slotStart := (reservedAt.Unix() / slotSec) * slotSec
    return fmt.Sprintf("lock:slot:%s:%d", restaurantID, slotStart)
}

// AcquireLock: SETNX 방식으로 락 획득 시도
// 반환: (lock, nil) 성공 / (nil, error) 실패 (이미 선점됨)
func AcquireLock(ctx context.Context, client *goredis.Client, key, token string, ttl time.Duration) (*DistributedLock, error) {
    ok, err := client.SetNX(ctx, key, token, ttl).Result()
    if err != nil {
        return nil, fmt.Errorf("redis error: %w", err)
    }
    if !ok {
        return nil, fmt.Errorf("lock already held: %s", key)
    }
    return &DistributedLock{client: client, key: key, token: token, ttl: ttl}, nil
}

// ReleaseLock: 본인 소유의 락만 해제 (Lua script로 원자적 확인+삭제)
var releaseLockScript = goredis.NewScript(`
    if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
    else
        return 0
    end
`)

func (l *DistributedLock) Release(ctx context.Context) error {
    return releaseLockScript.Run(ctx, l.client, []string{l.key}, l.token).Err()
}
```

---

## 7. 멱등키 처리

**Redis 키 패턴:** `idempotent:rsv:{idempotency_key}`
- 값: `reservation_id` 문자열
- TTL: 24시간

```go
// service.go — CreateReservation 내부 처리 흐름
func (s *Service) CreateReservation(ctx context.Context, req *CreateReservationRequest) (*CreateReservationResponse, error) {
    // 1. 멱등키 확인 (이미 처리된 요청이면 기존 결과 반환)
    if req.IdempotencyKey != "" {
        cacheKey := fmt.Sprintf("idempotent:rsv:%s", req.IdempotencyKey)
        if existingID, err := s.redis.Get(ctx, cacheKey).Result(); err == nil {
            // 이미 처리된 요청 → 기존 reservation_id 반환
            existing, _ := s.repo.GetReservation(ctx, existingID)
            return &CreateReservationResponse{
                ReservationId: existingID,
                Status:        existing.Status,
            }, nil
        }
    }

    // 2. B에서 참조 데이터 조회 (휴무일/운영시간/좌석정책)
    refData, err := s.restaurantClient.GetReservationReferenceData(ctx, req.RestaurantId, req.ReservedAt[:10])
    if err != nil {
        return nil, err
    }
    if refData.HolidayFlag {
        return nil, status.Error(codes.FailedPrecondition, "restaurant is closed on this date")
    }

    // 3. 슬롯 유효성 검사 (운영시간 범위 내인지)
    if err := validateSlot(req.ReservedAt, refData); err != nil {
        return nil, err
    }

    // 4. Redis 분산 락 획득 (TTL=30초)
    lockKey := redis.SlotLockKey(req.RestaurantId, parsedTime, seatPolicy.SlotDurationMin)
    lock, err := redis.AcquireLock(ctx, s.redis, lockKey, uuid.New().String(), 30*time.Second)
    if err != nil {
        return nil, status.Error(codes.Aborted, "slot is being reserved by another request")
    }
    defer lock.Release(ctx)

    // 5. MySQL SELECT FOR UPDATE — DB 수준 중복 체크
    tx, _ := s.db.BeginTx(ctx, nil)
    defer tx.Rollback()

    count, _ := s.repo.CountActiveReservationsForSlot(ctx, tx, req.RestaurantId, parsedTime)
    if count >= seatPolicy.TableCount {
        return nil, status.Error(codes.Aborted, "no available tables for this slot")
    }

    // 6. 예약 INSERT
    reservationID := uuid.New().String()
    if err := s.repo.CreateReservation(ctx, tx, &Reservation{
        ID:             reservationID,
        CustomerID:     req.CustomerId,
        RestaurantID:   req.RestaurantId,
        PartySize:      req.PartySize,
        ReservedAt:     parsedTime,
        Status:         StatusPending,
        IdempotencyKey: req.IdempotencyKey,
    }); err != nil {
        return nil, status.Error(codes.Internal, "failed to create reservation")
    }
    tx.Commit()

    // 7. 멱등키 캐싱 (24시간)
    if req.IdempotencyKey != "" {
        cacheKey := fmt.Sprintf("idempotent:rsv:%s", req.IdempotencyKey)
        s.redis.Set(ctx, cacheKey, reservationID, 24*time.Hour)
    }

    // 8. 점주에게 예약 신청 알림 발송
    go s.notifyOwner(req.RestaurantId, reservationID, "RESERVATION_REQUESTED")

    return &CreateReservationResponse{
        ReservationId: reservationID,
        Status:        "PENDING",
    }, nil
}
```

---

## 8. 슬롯 가용성 계산 (`GetAvailability`)

파일 위치: `backend/internal/reservation/slot_calculator.go`

```
입력: restaurant_id="rest_xyz", date="2026-03-24", party_size=2

처리 흐름:
1. B의 GetReservationReferenceData 호출
   - holiday_flag=true → 빈 슬롯 목록 반환
   - is_closed=true → 빈 슬롯 목록 반환

2. seat_policies에서 party_size를 수용 가능한 정책 선택
   - min_party <= party_size <= max_party 인 정책 찾기
   - 해당 테이블 총 개수: table_count

3. 운영시간 내에서 slot_duration_min 간격으로 슬롯 목록 생성
   - 예: open=11:00, close=22:00, slot=90분
   → 11:00, 12:30, 14:00, 15:30, 17:00, 18:30, 20:00
   - break_time 범위의 슬롯 제외

4. 각 슬롯별로 reservations 테이블에서 CONFIRMED/PENDING 예약 수 카운트
   - remaining_seats = table_count - active_reservation_count

5. remaining_seats > 0 이면 available=true

출력:
[
  {"reserved_at":"11:00","available":true,"remaining_seats":10},
  {"reserved_at":"12:30","available":true,"remaining_seats":8},
  {"reserved_at":"19:00","available":false,"remaining_seats":0},
  ...
]
```

---

## 9. 노쇼 처리 (`noshow_monitor.go`)

방문 예정 시간 + 60분이 지났는데 CONFIRMED 상태인 예약을 자동으로 NO_SHOW 처리합니다.

```go
// noshow_monitor.go
package reservation

import (
    "context"
    "time"
)

// 5분마다 실행되는 크론 함수
func (s *Service) RunNoShowMonitor(ctx context.Context) {
    ticker := time.NewTicker(s.cfg.NoShowCheckInterval)
    defer ticker.Stop()

    for {
        select {
        case <-ctx.Done():
            return
        case <-ticker.C:
            s.processNoShows(ctx)
        }
    }
}

func (s *Service) processNoShows(ctx context.Context) {
    // CONFIRMED 상태이면서 reserved_at + grace_period(60분)이 지난 예약 조회
    threshold := time.Now().Add(-s.cfg.NoShowGracePeriod)

    reservations, err := s.repo.FindConfirmedBeforeThreshold(ctx, threshold)
    if err != nil {
        s.logger.Error("failed to query no-show candidates", "error", err)
        return
    }

    for _, rsv := range reservations {
        if err := s.applyNoShow(ctx, rsv.ID, "system"); err != nil {
            s.logger.Error("failed to apply no-show", "reservation_id", rsv.ID, "error", err)
        }
    }
}
```

---

## 10. 이벤트 발행 (C → D, Redis Streams)

예약 취소 시 D(웨이팅)에 이벤트를 발행합니다. D는 이 이벤트를 소비하여 웨이팅 자동 승격을 처리합니다.

**발행 키:** `events:reservation.canceled.v1`

```go
// service.go — CancelReservation 내 이벤트 발행
func (s *Service) publishCancelEvent(ctx context.Context, rsv *Reservation) {
    event := map[string]interface{}{
        "event_id":       uuid.New().String(),
        "event_name":     "reservation.canceled.v1",
        "occurred_at":    time.Now().Format(time.RFC3339),
        "reservation_id": rsv.ID,
        "restaurant_id":  rsv.RestaurantID,
        "customer_id":    rsv.CustomerID,
        "reserved_at":    rsv.ReservedAt.Format(time.RFC3339),
        "party_size":     rsv.PartySize,
        "cancel_reason":  rsv.CancelReason,
    }
    payload, _ := json.Marshal(event)
    s.redis.XAdd(ctx, &goredis.XAddArgs{
        Stream: "events:reservation.canceled.v1",
        Values: map[string]interface{}{"data": string(payload)},
    })
}
```

**이벤트 스키마 (D 담당자 참고):**
```json
{
  "event_id":       "550e8400-e29b-41d4-a716-446655440000",
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

---

## 11. B 서비스 호출 (내가 의존하는 B의 함수)

> B 담당자가 구현 후 제공하는 gRPC 함수입니다.
> B의 `GetReservationReferenceData()` 응답 구조가 변경되면 반드시 나에게 사전 고지 필요.

```go
// B 클라이언트 호출 예시
func (s *Service) getRefData(ctx context.Context, restaurantID, date string) (*restaurant.GetReservationReferenceDataResponse, error) {
    conn, err := grpc.DialContext(ctx, s.cfg.RestaurantServiceAddr, grpc.WithTransportCredentials(insecure.NewCredentials()))
    if err != nil {
        return nil, err
    }
    client := restaurant.NewRestaurantServiceClient(conn)
    return client.GetReservationReferenceData(ctx, &restaurant.GetReservationReferenceDataRequest{
        RestaurantId: restaurantID,
        TargetDate:   date,
    })
}
```

---

## 12. Redis 키 패턴 (내 담당)

| 키 | TTL | 설명 |
|----|-----|------|
| `lock:slot:{restaurant_id}:{slot_start_unix}` | 30초 | 예약 슬롯 분산 락 |
| `idempotent:rsv:{idempotency_key}` | 24시간 | 멱등키 → reservation_id 매핑 |

```go
func SlotLockKey(restaurantID string, slotStartUnix int64) string {
    return fmt.Sprintf("lock:slot:%s:%d", restaurantID, slotStartUnix)
}

func IdempotentReservationKey(idempotencyKey string) string {
    return fmt.Sprintf("idempotent:rsv:%s", idempotencyKey)
}
```

---

## 13. 주차별 상세 작업 + 완료 조건

| 주차 | 작업 내용 | 완료 조건 |
|------|---------|---------|
| W1 | 상태 머신 + 분산 락 + 트랜잭션 골격 구현 | 동시 요청 100개 중 정확히 1개만 성공하는 k6 테스트 통과 |
| W2 | CreateReservation + GetAvailability API | 멱등키 동일 재요청 시 동일 `reservation_id` 반환 |
| W3 | 슬롯 계산 고도화 (B 운영시간/휴무 반영) | `GetAvailability` 정확도 100% (B 참조 데이터 기반) |
| W4 | Confirm/Reject/Cancel + FE 연동 | 상태 전이 예외 테스트 **15건** 통과 |
| W5 | `reservation.canceled.v1` 이벤트 발행 + D 연동 | D에서 취소 이벤트 수신 후 웨이팅 승격 이벤트 발행 확인 |
| W6 | DB deadlock 재시도 로직 + 복구 처리 | 의도적 deadlock 주입 시 재시도 후 성공 |
| W7~W9 | 부하 테스트/회귀/UAT | p99 200ms, 중복예약 0건 |
| W10 | 운영 지표 튜닝 | 충돌 차단율/실패 사유 대시보드 노출 |

---

## 14. 교차 점검 항목

| 교차 | 점검 내용 | 검증 방법 |
|------|---------|---------|
| C↔B | 휴무일에는 `CreateReservation()` → `FAILED_PRECONDITION` 반환 | B에서 holiday 설정 후 C 통합 테스트 |
| C↔A | `x-role=OWNER`인 유저가 고객 예약 생성 시도 → `PERMISSION_DENIED` | auth interceptor 단위 테스트 |
| C↔D | 취소 이벤트 `reservation.canceled.v1` payload schema 완전 일치 | D 담당자와 스키마 JSON 대조 후 서명 |
| C↔E | 예약 실패 사유 코드(`ABORTED`, `FAILED_PRECONDITION` 등)가 E 대시보드 분류체계와 일치 | E 담당자와 사유 코드 목록 합의 |

---

## 15. 상세 테스트 케이스 목록

### CreateReservation (C-FN-01)
- [ ] 정상 예약 생성 → `reservation_id` 반환, status=PENDING
- [ ] 동시 100개 요청 같은 슬롯 → 정확히 1개만 성공
- [ ] 멱등키로 동일 요청 재시도 → 동일 `reservation_id` 반환
- [ ] 휴무일 예약 시도 → `FAILED_PRECONDITION`
- [ ] 운영시간 외 예약 시도 → `FAILED_PRECONDITION`
- [ ] 브레이크타임 슬롯 예약 시도 → `FAILED_PRECONDITION`
- [ ] CUSTOMER role이 아닌 유저 예약 시도 → `PERMISSION_DENIED`
- [ ] is_restricted=true 유저 예약 시도 → `PERMISSION_DENIED`

### GetAvailability (C-FN-02)
- [ ] 정상 조회 → 운영시간 기반 슬롯 목록 반환
- [ ] 만석 슬롯은 available=false
- [ ] 휴무일 조회 → 빈 목록 반환
- [ ] 인원 수용 불가 party_size → 해당 정책 슬롯 absent

### 상태 전이 (C-FN-03~07)
- [ ] PENDING → CONFIRMED (점주 확정)
- [ ] PENDING → REJECTED (점주 거절)
- [ ] PENDING → CANCELLED (고객 취소)
- [ ] CONFIRMED → CANCELLED (고객 취소, 방문 2시간 전) → 성공
- [ ] CONFIRMED → CANCELLED (방문 30분 전) → `FAILED_PRECONDITION`
- [ ] CONFIRMED → NO_SHOW (노쇼 처리)
- [ ] CANCELLED에 추가 상태 전이 시도 → `FAILED_PRECONDITION`
- [ ] REJECTED에 confirm 시도 → `FAILED_PRECONDITION`

### 노쇼 모니터 (자동)
- [ ] CONFIRMED 상태 + 방문시간 + 70분 경과 → 자동 NO_SHOW 처리
- [ ] CONFIRMED 상태 + 방문시간 + 30분 경과 → 아직 변경 없음

---

## 16. 정량 목표 (SLO/품질)

| 지표 | 목표값 |
|------|--------|
| 중복 예약 | 0건 |
| 예약 API p99 응답 시간 | < 200ms |
| 멱등성 위반 | 0건 |
| 슬롯 가용성 계산 정확도 | 100% |
| deadlock 재시도 성공률 | 100% |
