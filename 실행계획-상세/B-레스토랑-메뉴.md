# B 담당 문서 — 레스토랑/메뉴/운영정책

> 이 문서만 보고 개발할 수 있도록 작성되었습니다.
> Proto 스키마 전체는 `00-공용-proto-스키마.md` 섹션 5-3, 5-4를 참고하세요.

---

## 0. 담당 서비스 & 디렉토리

### 담당 컨테이너

| 컨테이너명 | gRPC 포트 | 메트릭 포트 | 설명 |
|-----------|-----------|------------|------|
| `dinespot-restaurant-service` | 50052 | 9101 | 매장/운영시간/좌석정책 |
| `dinespot-menu-service` | 50055 | 9104 | 메뉴 CRUD + 사전 주문 |

### 담당 디렉토리

```
backend/
├── internal/
│   ├── restaurant/                  ← 핵심 구현 (restaurant-service)
│   │   ├── handler.go               # gRPC 핸들러 (RestaurantServiceServer 구현)
│   │   ├── service.go               # 비즈니스 로직 (운영시간 검증 등)
│   │   ├── repository.go            # DB 접근 계층
│   │   ├── storage.go               # S3/GCS 이미지 업로드 클라이언트
│   │   └── handler_test.go
│   └── menu/                        ← 핵심 구현 (menu-service)
│       ├── handler.go               # gRPC 핸들러 (MenuServiceServer 구현)
│       ├── service.go
│       ├── repository.go
│       ├── payment_client.go        # 결제 PG 클라이언트
│       └── handler_test.go

infra/
└── mysql/
    └── init/
        ├── 02_restaurants.sql       ← 레스토랑 관련 DDL
        └── 03_menus.sql             ← 메뉴/주문 관련 DDL
```

### docker-compose 서비스 스니펫

```yaml
restaurant-service:
  build:
    context: ./backend
    dockerfile: Dockerfile
    target: service
  container_name: dinespot-restaurant-service
  command: ["/app/server", "--service=restaurant", "--port=50052"]
  ports:
    - "50052:50052"
    - "9101:9101"
  environment:
    SERVICE_NAME: restaurant-service
    DB_DSN: "dinespot:dinespot_pass@tcp(mysql:3306)/dinespot?parseTime=true&loc=Asia%2FSeoul"
    REDIS_ADDR: "redis:6379"
    USER_SERVICE_ADDR: "user-service:50051"
    STORAGE_TYPE: "${STORAGE_TYPE:-local}"
    STORAGE_LOCAL_PATH: "/tmp/dinespot-images"
    S3_BUCKET: "${S3_BUCKET:-}"
    S3_REGION: "${S3_REGION:-ap-northeast-2}"
    GCS_BUCKET: "${GCS_BUCKET:-}"
    LOG_LEVEL: "info"
  volumes:
    - restaurant_images:/tmp/dinespot-images
  depends_on:
    mysql:
      condition: service_healthy
    user-service:
      condition: service_started
  networks: [dinespot-net]

menu-service:
  build:
    context: ./backend
    dockerfile: Dockerfile
    target: service
  container_name: dinespot-menu-service
  command: ["/app/server", "--service=menu", "--port=50055"]
  ports:
    - "50055:50055"
    - "9104:9104"
  environment:
    SERVICE_NAME: menu-service
    DB_DSN: "dinespot:dinespot_pass@tcp(mysql:3306)/dinespot?parseTime=true&loc=Asia%2FSeoul"
    REDIS_ADDR: "redis:6379"
    RESTAURANT_SERVICE_ADDR: "restaurant-service:50052"
    NOTIFICATION_SERVICE_ADDR: "notification-service:50056"
    PAYMENT_GATEWAY_URL: "${PAYMENT_GATEWAY_URL:-http://mock-payment:8090}"
    PAYMENT_GATEWAY_SECRET: "${PAYMENT_GATEWAY_SECRET:-mock-secret}"
    LOG_LEVEL: "info"
  depends_on:
    mysql:
      condition: service_healthy
    restaurant-service:
      condition: service_started
  networks: [dinespot-net]
```

### 로컬 개발 명령어

```bash
# restaurant-service + 의존 서비스 시작
docker compose up -d mysql redis user-service restaurant-service

# menu-service 포함
docker compose up -d mysql redis user-service restaurant-service menu-service

# 내 서비스 로그 확인
docker compose logs -f restaurant-service menu-service

# 코드 변경 후 재빌드
docker compose up -d --build restaurant-service menu-service

# 테스트 실행 (로컬)
cd backend && go test ./internal/restaurant/... -v
cd backend && go test ./internal/menu/... -v

# gRPC 직접 테스트
grpcurl -plaintext \
  -H "authorization: Bearer {token}" \
  -d '{"tenant_id":"t1","name":"맛있는 식당","address":"서울시 강남구","category":"KOREAN","timezone":"Asia/Seoul"}' \
  localhost:50052 restaurant.v1.RestaurantService/CreateRestaurant
```

---

## 1. 역할 목표

- 예약 엔진(C)이 신뢰 가능한 매장 운영 데이터(좌석/영업시간/휴무)를 조회할 수 있게 한다.
- 점주 UI와 API 데이터가 완전히 동일한 규칙으로 검증되어야 한다.
- `GetReservationReferenceData()`는 C의 **핵심 의존 API**이므로 응답 스키마가 변경되면 반드시 C 담당자에게 사전 고지 필요.

---

## 2. 기능 요구사항 (FR)

| ID | 요구사항 | 우선순위 |
|----|---------|---------|
| FR-B-01 | 레스토랑 CRUD (이름/주소/카테고리/설명/이미지) | P0 |
| FR-B-02 | 메뉴 CRUD + 품절 상태 관리 | P0 |
| FR-B-03 | 운영시간/브레이크타임 관리 (요일별) | P0 |
| FR-B-04 | 특정 날짜 휴무일 설정 | P1 |
| FR-B-05 | 좌석 정책(테이블 타입/인원 범위/개수/슬롯 단위) 관리 | P0 |
| FR-B-06 | 테넌트 격리 (점주는 자신의 매장만 접근) | P0 |
| FR-B-07 | 매장 검색 API (지역/카테고리/날짜/인원) | P0 |
| FR-B-08 | 이미지 업로드 (S3/GCS, 로컬 폴백) | P1 |
| FR-B-09 | 매장 등록 신청 (상태: PENDING_REVIEW) | P0 |
| FR-B-10 | 사전 주문 처리 + 결제 PG 연동 | P1 |

---

## 3. API/함수 구현 명세

| ID | 함수 | Input | Output | 실패 조건 |
|----|------|-------|--------|----------|
| B-FN-01 | `CreateRestaurant(ctx, req)` | `tenant_id, name, address, category, timezone` | `restaurant_id` | 중복 매장명(`ALREADY_EXISTS`) |
| B-FN-02 | `UpdateOperatingHours(ctx, req)` | `restaurant_id, hours[]` (7일치) | `version` | 시간 역전/브레이크타임 중복(`INVALID_ARGUMENT`) |
| B-FN-03 | `UpsertSeatPolicy(ctx, req)` | `restaurant_id, table_type, min_party, max_party, table_count, slot_duration_min` | `policy_id` | min > max(`INVALID_ARGUMENT`) |
| B-FN-04 | `SetHoliday(ctx, req)` | `restaurant_id, holiday_date, reason` | `holiday_id` | 과거 날짜 설정(`INVALID_ARGUMENT`) |
| B-FN-05 | `UpsertMenuItem(ctx, req)` | `restaurant_id, category, name, price, stock, status` | `menu_item_id` | 음수 가격(`INVALID_ARGUMENT`) |
| B-FN-06 | `GetReservationReferenceData(ctx, req)` | `restaurant_id, target_date` | `holiday_flag, hours, seat_policies, timezone` | 타 테넌트 접근(`PERMISSION_DENIED`) |
| B-FN-07 | `SearchRestaurants(ctx, req)` | `area, category, date, time, party_size, page` | `restaurants[], page` | |
| B-FN-08 | `PlacePreOrder(ctx, req)` | `reservation_id, items[], idempotency_key` | `order_id, total_amount, payment_url` | 품절 메뉴(`FAILED_PRECONDITION`) |

---

## 4. DB 스키마

### 파일 위치: `infra/mysql/init/02_restaurants.sql`

```sql
-- 레스토랑
CREATE TABLE IF NOT EXISTS restaurants (
  id          CHAR(36)      NOT NULL,
  tenant_id   CHAR(36)      NOT NULL,   -- = owner_id
  owner_id    CHAR(36)      NOT NULL,
  name        VARCHAR(200)  NOT NULL,
  address     VARCHAR(500)  NOT NULL,
  category    VARCHAR(50)   NOT NULL,   -- 'KOREAN','JAPANESE','CHINESE','WESTERN','CAFE','ETC'
  timezone    VARCHAR(50)   NOT NULL DEFAULT 'Asia/Seoul',
  status      ENUM('PENDING_REVIEW','ACTIVE','INACTIVE','REJECTED') NOT NULL DEFAULT 'PENDING_REVIEW',
  image_url   VARCHAR(1000) DEFAULT NULL,
  description TEXT          DEFAULT NULL,
  created_at  DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at  DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_restaurants_tenant_id (tenant_id),
  KEY idx_restaurants_status (status),
  KEY idx_restaurants_category (category)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 요일별 운영시간
CREATE TABLE IF NOT EXISTS operating_hours (
  id              CHAR(36)   NOT NULL,
  restaurant_id   CHAR(36)   NOT NULL,
  day_of_week     TINYINT    NOT NULL,  -- 0=월, 1=화, ..., 6=일
  open_time       TIME       NOT NULL,
  close_time      TIME       NOT NULL,
  is_closed       TINYINT(1) NOT NULL DEFAULT 0,
  PRIMARY KEY (id),
  UNIQUE KEY uq_oh_restaurant_day (restaurant_id, day_of_week),
  KEY idx_oh_restaurant_id (restaurant_id),
  CONSTRAINT fk_oh_restaurant FOREIGN KEY (restaurant_id) REFERENCES restaurants(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 브레이크타임 (운영시간 내 휴식)
CREATE TABLE IF NOT EXISTS break_times (
  id                  CHAR(36) NOT NULL,
  operating_hours_id  CHAR(36) NOT NULL,
  start_time          TIME     NOT NULL,
  end_time            TIME     NOT NULL,
  PRIMARY KEY (id),
  KEY idx_bt_oh_id (operating_hours_id),
  CONSTRAINT fk_bt_oh FOREIGN KEY (operating_hours_id) REFERENCES operating_hours(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 특정 날짜 휴무일
CREATE TABLE IF NOT EXISTS holidays (
  id            CHAR(36)     NOT NULL,
  restaurant_id CHAR(36)     NOT NULL,
  holiday_date  DATE         NOT NULL,
  reason        VARCHAR(200) DEFAULT NULL,
  PRIMARY KEY (id),
  UNIQUE KEY uq_holiday_restaurant_date (restaurant_id, holiday_date),
  CONSTRAINT fk_holiday_restaurant FOREIGN KEY (restaurant_id) REFERENCES restaurants(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 좌석 정책
CREATE TABLE IF NOT EXISTS seat_policies (
  id                CHAR(36)    NOT NULL,
  restaurant_id     CHAR(36)    NOT NULL,
  table_type        VARCHAR(50) NOT NULL,  -- 'BOOTH','TABLE','BAR','PRIVATE'
  min_party         INT         NOT NULL,
  max_party         INT         NOT NULL,
  table_count       INT         NOT NULL,
  slot_duration_min INT         NOT NULL DEFAULT 90,
  PRIMARY KEY (id),
  UNIQUE KEY uq_seat_policy (restaurant_id, table_type),
  KEY idx_sp_restaurant_id (restaurant_id),
  CONSTRAINT fk_sp_restaurant FOREIGN KEY (restaurant_id) REFERENCES restaurants(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 파일 위치: `infra/mysql/init/03_menus.sql`

```sql
-- 메뉴 아이템
CREATE TABLE IF NOT EXISTS menu_items (
  id            CHAR(36)      NOT NULL,
  restaurant_id CHAR(36)      NOT NULL,
  category      VARCHAR(100)  DEFAULT NULL,
  name          VARCHAR(200)  NOT NULL,
  price         BIGINT        NOT NULL,   -- 원 단위 (DECIMAL 대신 정수로 오차 방지)
  description   TEXT          DEFAULT NULL,
  image_url     VARCHAR(1000) DEFAULT NULL,
  stock         INT           NOT NULL DEFAULT -1,   -- -1 = 무제한
  is_sold_out   TINYINT(1)    NOT NULL DEFAULT 0,
  status        ENUM('ACTIVE','INACTIVE') NOT NULL DEFAULT 'ACTIVE',
  sort_order    INT           NOT NULL DEFAULT 0,
  created_at    DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_menu_items_restaurant_status (restaurant_id, status),
  CONSTRAINT fk_mi_restaurant FOREIGN KEY (restaurant_id) REFERENCES restaurants(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 주문
CREATE TABLE IF NOT EXISTS orders (
  id              CHAR(36)      NOT NULL,
  reservation_id  CHAR(36)      NOT NULL,
  customer_id     CHAR(36)      NOT NULL,
  total_amount    BIGINT        NOT NULL,
  status          ENUM('PENDING','CONFIRMED','PAYMENT_PENDING','CANCELLED') NOT NULL DEFAULT 'PENDING',
  idempotency_key VARCHAR(128)  DEFAULT NULL,
  created_at      DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at      DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uq_orders_idempotency (idempotency_key),
  KEY idx_orders_reservation_id (reservation_id),
  KEY idx_orders_customer_id (customer_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 주문 아이템
CREATE TABLE IF NOT EXISTS order_items (
  id           CHAR(36) NOT NULL,
  order_id     CHAR(36) NOT NULL,
  menu_item_id CHAR(36) NOT NULL,
  quantity     INT      NOT NULL,
  unit_price   BIGINT   NOT NULL,
  PRIMARY KEY (id),
  KEY idx_oi_order_id (order_id),
  CONSTRAINT fk_oi_order FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 결제
CREATE TABLE IF NOT EXISTS payments (
  id                CHAR(36)     NOT NULL,
  order_id          CHAR(36)     NOT NULL,
  pg_transaction_id VARCHAR(200) DEFAULT NULL,
  amount            BIGINT       NOT NULL,
  status            ENUM('PENDING','PAID','FAILED','REFUNDED') NOT NULL DEFAULT 'PENDING',
  paid_at           DATETIME     DEFAULT NULL,
  created_at        DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uq_payments_order (order_id),
  KEY idx_payments_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 5. `operatingHours` 검증 규칙

`UpdateOperatingHours()` 구현 시 아래 규칙을 서버 측에서 반드시 검증해야 합니다.

```
규칙 1: open_time < close_time (시간 역전 금지)
         예외: 새벽 영업 (ex: 22:00 ~ 02:00) → close_time이 다음 날을 의미
         → 구현: close_time < open_time이면 익일 마감으로 처리 (close_time += 24h)

규칙 2: break_time은 open_time ~ close_time 범위 내에 있어야 함

규칙 3: break_time 간 겹침 금지
         예: 14:00~15:00 와 14:30~16:00 → INVALID_ARGUMENT

규칙 4: 7개 요일(0~6) 모두 설정 필수 (is_closed=true로 표시 가능)

규칙 5: slot_duration_min은 30의 배수 (30, 60, 90, 120 허용)
```

```go
// 운영시간 검증 예시 (service.go)
func validateOperatingHours(hours []*OperatingHourSlot) error {
    if len(hours) != 7 {
        return status.Error(codes.InvalidArgument, "must provide all 7 days")
    }
    for _, h := range hours {
        if h.IsClosed {
            continue
        }
        // open_time 파싱
        open, _ := time.Parse("15:04", h.OpenTime)
        close, _ := time.Parse("15:04", h.CloseTime)
        if !close.After(open) && h.CloseTime != "00:00" {
            return status.Errorf(codes.InvalidArgument,
                "day %d: close_time must be after open_time", h.DayOfWeek)
        }
        // break_time 겹침 검증 (O(n²), n이 작으므로 허용)
        for i, b1 := range h.BreakTimes {
            for j, b2 := range h.BreakTimes {
                if i >= j {
                    continue
                }
                if breaksOverlap(b1, b2) {
                    return status.Error(codes.InvalidArgument, "break times must not overlap")
                }
            }
        }
    }
    return nil
}
```

---

## 6. 이미지 업로드 흐름

```
FE                      restaurant-service             S3/GCS / 로컬 스토리지
│                              │                              │
│  multipart/form-data POST    │                              │
│──────────────────────────────▶                              │
│  (gRPC-Web는 binary stream)  │                              │
│                              │  PutObject(key, body)        │
│                              │─────────────────────────────▶
│                              │  URL 반환                    │
│                              │◀─────────────────────────────
│                              │  DB UPDATE image_url=URL     │
│  image_url 반환              │                              │
│◀──────────────────────────────                              │
```

**이미지 키 규칙:**
- `restaurants/{restaurant_id}/cover.{ext}`
- `menus/{restaurant_id}/{menu_item_id}.{ext}`

**로컬 개발 (STORAGE_TYPE=local):**
- `/tmp/dinespot-images/` 폴더에 저장
- URL 형식: `http://localhost:50052/static/{path}`

**환경변수 분기:**
```go
// storage.go
func NewStorage(cfg StorageConfig) Storage {
    switch cfg.Type {
    case "s3":
        return newS3Storage(cfg.S3Bucket, cfg.S3Region)
    case "gcs":
        return newGCSStorage(cfg.GCSBucket)
    default: // "local"
        return newLocalStorage(cfg.LocalPath)
    }
}
```

---

## 7. `GetReservationReferenceData()` 구현 상세

이 함수는 C(예약 서비스)의 **핵심 의존 API**입니다. 응답 정확도가 100%여야 합니다.

```
입력: restaurant_id="rest_xyz", target_date="2026-03-24" (화요일)

처리 순서:
1. restaurants 테이블에서 restaurant_id, tenant_id 확인
2. 호출자의 tenant_id와 일치 확인 (PERMISSION_DENIED 체크)
3. holidays 테이블에서 target_date 조회 → holiday_flag 결정
4. operating_hours 테이블에서 target_date의 요일(1=화) 행 조회
   + break_times JOIN
5. seat_policies 테이블에서 해당 restaurant_id 전체 조회
6. Redis 캐시 `cache:ref_data:{restaurant_id}:{date}` 확인
   → HIT: 캐시 반환 (TTL 5분)
   → MISS: DB 조회 후 캐시 저장

출력:
{
  "restaurant_id": "rest_xyz",
  "target_date": "2026-03-24",
  "holiday_flag": false,
  "hours": [{
    "day_of_week": 1,
    "open_time": "11:00",
    "close_time": "22:00",
    "is_closed": false,
    "break_times": [{"start_time":"15:00","end_time":"17:00"}]
  }],
  "seat_policies": [
    {"table_type":"TABLE","min_party":2,"max_party":4,"table_count":10,"slot_duration_min":90},
    {"table_type":"BOOTH","min_party":2,"max_party":6,"table_count":5,"slot_duration_min":90}
  ],
  "timezone": "Asia/Seoul"
}
```

**주의:** `holiday_flag=true`이면 C는 `CreateReservation()`에서 즉시 `FAILED_PRECONDITION` 반환합니다.

---

## 8. Redis 캐시 키 (내 담당)

| 키 | TTL | 설명 |
|----|-----|------|
| `cache:ref_data:{restaurant_id}:{date}` | 5분 | `GetReservationReferenceData` 응답 캐시 |

```go
// 캐시 키 생성
func RefDataCacheKey(restaurantID, date string) string {
    return fmt.Sprintf("cache:ref_data:%s:%s", restaurantID, date)
}
```

---

## 9. 결제 흐름 (menu-service)

```
FE                  menu-service          Payment GW (Toss 등)
│                       │                       │
│ PlacePreOrder()        │                       │
│───────────────────────▶                        │
│                       │  재고 확인 (SELECT)    │
│                       │  주문 생성 (INSERT)    │
│                       │  재고 차감 (UPDATE WHERE stock > 0)
│                       │                       │
│                       │  결제 요청             │
│                       │───────────────────────▶
│                       │  결제 결과             │
│                       │◀───────────────────────
│                       │                       │
│ (성공 시)             │  payments INSERT       │
│ order_id, CONFIRMED   │  orders UPDATE CONFIRMED
│◀───────────────────────                        │
│                       │                       │
│ (실패 시)             │  orders UPDATE PAYMENT_PENDING
│ order_id, PAYMENT_PENDING, payment_url         │
│◀───────────────────────                        │
│                       │                       │
│ (재결제 시도)          │  재결제 요청           │
│ RetryPayment()        │───────────────────────▶
│───────────────────────▶                        │
```

**재고 차감 쿼리 (동시성 안전):**
```sql
UPDATE menu_items
SET stock = stock - ?,
    is_sold_out = CASE WHEN stock - ? <= 0 THEN 1 ELSE 0 END
WHERE id = ? AND stock > 0 AND stock >= ?;
-- 영향 행 0이면 FAILED_PRECONDITION(품절) 반환
```

---

## 10. 주차별 상세 작업 + 완료 조건

| 주차 | 작업 내용 | 완료 조건 |
|------|---------|---------|
| W1 | DB 스키마 마이그레이션 (`02_restaurants.sql`, `03_menus.sql`) | `docker compose up` 시 테이블 자동 생성, `down → up` 재현 가능 |
| W2 | Restaurant/Menu CRUD API 구현 | 생성/수정/삭제 통합테스트 **20건** 통과 |
| W3 | 운영시간/브레이크타임/휴무일/좌석정책 API + `GetReservationReferenceData()` | C에서 호출 가능한 gRPC 응답 규약 고정, C 담당자 연동 확인 |
| W4 | 점주 관리 UI 연동 + 이미지 업로드 | FE 폼 검증과 BE 검증 규칙 1:1 일치 |
| W5 | C 연동 검증 + Redis 캐시 적용 | C 예약 계산 시 B 참조 데이터 mismatch 0건 |
| W6 | 검색 API 성능 튜닝 (인덱스 확인) | 주요 조회 p95 < 150ms |
| W7~W9 | 테넌시 격리 회귀 테스트 + UAT | 테넌트 격리 위반 0건 |
| W10 | 운영 대응 (이상 쿼리 모니터링) | 관리자 수동 보정 SQL 없이 운영 가능 |

---

## 11. 교차 점검 항목

| 교차 | 점검 내용 | 검증 방법 |
|------|---------|---------|
| B↔C | `holiday_flag=true`일 때 C는 예약 생성 금지 | C 통합 테스트에서 휴무일 예약 시도 → `FAILED_PRECONDITION` 확인 |
| B↔A | 점주가 타 tenant의 restaurant_id 조회 시 `PERMISSION_DENIED` 고정 | `x-tenant-id` mismatch 테스트 |
| B↔E | `restaurant.status_changed.v1` 이벤트 발행 확인 | E의 메트릭 대시보드에서 이벤트 수신 확인 |
| B↔C | `slot_duration_min`이 C의 슬롯 계산에 올바르게 반영 | C가 90분 슬롯 기준으로 09:00~21:00 슬롯 개수 계산 정확성 테스트 |

---

## 12. 상세 테스트 케이스 목록

### CreateRestaurant (B-FN-01)
- [ ] 정상 생성 (PENDING_REVIEW 상태로 생성)
- [ ] 동일 테넌트 내 매장명 중복 → 허용 (다른 매장으로 등록 가능)
- [ ] 타 테넌트 이름으로 매장 생성 → 허용 (테넌트 격리이므로 독립)
- [ ] 필수 필드 누락 (name 없음) → `INVALID_ARGUMENT`

### UpdateOperatingHours (B-FN-02)
- [ ] 정상 업데이트 (7일치)
- [ ] close_time < open_time (자정 넘기는 경우) → 정상 처리
- [ ] 브레이크타임 겹침 → `INVALID_ARGUMENT`
- [ ] 7개 미만 요일 데이터 → `INVALID_ARGUMENT`
- [ ] 타 테넌트 매장 수정 시도 → `PERMISSION_DENIED`

### UpsertSeatPolicy (B-FN-03)
- [ ] 신규 정책 생성
- [ ] 기존 정책 업데이트 (UPSERT)
- [ ] min_party > max_party → `INVALID_ARGUMENT`
- [ ] slot_duration_min = 45 (30 배수 아님) → `INVALID_ARGUMENT`

### GetReservationReferenceData (B-FN-06)
- [ ] 영업일 조회 → holiday_flag=false, hours/seat_policies 반환
- [ ] 휴무일 조회 → holiday_flag=true
- [ ] 정기 휴무 요일 조회 → is_closed=true
- [ ] 타 테넌트 접근 → `PERMISSION_DENIED`
- [ ] 5분 이내 재호출 → Redis 캐시 히트 (DB 쿼리 없음)

### UpsertMenuItem (B-FN-05)
- [ ] 신규 메뉴 생성
- [ ] 음수 가격 → `INVALID_ARGUMENT`
- [ ] 재고 0 → is_sold_out 자동 true
- [ ] 타 테넌트 메뉴 수정 → `PERMISSION_DENIED`

---

## 13. 정량 목표 (SLO/품질)

| 지표 | 목표값 |
|------|--------|
| 매장/메뉴 CRUD 성공률 | ≥ 99.9% |
| `GetReservationReferenceData` 응답 정확도 | 100% |
| 테넌시 격리 위반 | 0건 |
| 주요 조회 API p95 | < 150ms |
| 이미지 업로드 성공률 | ≥ 99.5% |
