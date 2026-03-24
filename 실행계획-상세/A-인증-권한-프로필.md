# A 담당 문서 — 인증/권한/프로필

> 이 문서만 보고 개발할 수 있도록 작성되었습니다.
> Proto 스키마 전체는 `00-공용-proto-스키마.md` 섹션 5-2를 참고하세요.

---

## 0. 담당 서비스 & 디렉토리

### 담당 컨테이너

| 컨테이너명 | gRPC 포트 | 메트릭 포트 | 설명 |
|-----------|-----------|------------|------|
| `dinespot-user-service` | 50051 | 9100 | 인증/권한/프로필 서비스 |

### 담당 디렉토리

```
backend/
├── internal/
│   └── user/                        ← 핵심 구현
│       ├── handler.go               # gRPC 핸들러 (UserServiceServer 구현)
│       ├── service.go               # 비즈니스 로직
│       ├── repository.go            # DB 접근 계층
│       ├── rbac.go                  # RBAC 정책 테이블 + Authorize()
│       └── handler_test.go          # 단위/통합 테스트
└── pkg/
    └── middleware/
        └── auth.go                  ← B/C/D/E가 import해서 사용하는 인터셉터

infra/
└── mysql/
    └── init/
        └── 01_users.sql             ← users 테이블 DDL
```

### docker-compose 서비스 스니펫 (내 서비스만 뽑은 것)

```yaml
user-service:
  build:
    context: ./backend
    dockerfile: Dockerfile
    target: service
  container_name: dinespot-user-service
  command: ["/app/server", "--service=user", "--port=50051"]
  ports:
    - "50051:50051"    # gRPC
    - "9100:9100"      # Prometheus metrics
  environment:
    SERVICE_NAME: user-service
    DB_DSN: "dinespot:dinespot_pass@tcp(mysql:3306)/dinespot?parseTime=true&loc=Asia%2FSeoul"
    REDIS_ADDR: "redis:6379"
    JWT_SECRET: "${JWT_SECRET:-dev_jwt_secret_CHANGE_IN_PROD}"
    JWT_ACCESS_TTL: "15m"
    JWT_REFRESH_TTL: "168h"
    BCRYPT_COST: "12"
    LOG_LEVEL: "info"
  depends_on:
    mysql:
      condition: service_healthy
    redis:
      condition: service_healthy
  networks: [dinespot-net]
```

### 로컬 개발 명령어 (내 서비스만 띄우기)

```bash
# 의존 인프라 + user-service만 시작
docker compose up -d mysql redis user-service

# 내 서비스 로그 실시간 확인
docker compose logs -f user-service

# 코드 변경 후 재빌드
docker compose up -d --build user-service

# gRPC 직접 테스트 (grpcurl)
grpcurl -plaintext -d '{"email":"test@example.com","password":"pass1234","name":"홍길동"}' \
  localhost:50051 user.v1.UserService/Register

# DB 직접 접속
docker compose exec mysql mysql -u dinespot -pdinespot_pass dinespot
```

---

## 1. 역할 목표

- 모든 요청은 인증 컨텍스트(`user_id`, `tenant_id`, `role`)를 가진 상태로 downstream에 전달되어야 한다.
- 권한 정책은 코드와 테스트에서 동일한 규칙으로 강제되어야 한다.
- B/C/D/E가 내가 만든 `auth.go` 인터셉터를 import해서 별도 인증 로직 없이 사용할 수 있어야 한다.

---

## 2. 기능 요구사항 (FR)

| ID | 요구사항 | 우선순위 |
|----|---------|---------|
| FR-A-01 | 이메일/비밀번호 회원가입 (CUSTOMER, OWNER 역할) | P0 |
| FR-A-02 | 로그인 — Access Token(15분) + Refresh Token(7일) 발급 | P0 |
| FR-A-03 | Refresh Token Rotation (RTR) 기반 토큰 재발급 | P0 |
| FR-A-04 | 로그아웃 — 토큰 블랙리스트 등록 | P0 |
| FR-A-05 | 프로필 조회/수정 | P1 |
| FR-A-06 | RBAC 권한 검증 (`Authorize()`) | P0 |
| FR-A-07 | 감사로그용 actor 정보 전달 (gRPC metadata) | P0 |
| FR-A-08 | 로그인 실패 rate limit (5회/1시간 → 1시간 잠금) | P1 |
| FR-A-09 | 누적 노쇼 3회 → 계정 예약 30일 제한 (`is_restricted`) | P1 |

---

## 3. API/함수 구현 명세

| ID | 함수 | Input | Output | 실패 조건 |
|----|------|-------|--------|----------|
| A-FN-01 | `RegisterUser(ctx, req)` | `email, password, name, role` | `user_id, created_at` | 중복 이메일 → `ALREADY_EXISTS` |
| A-FN-02 | `Login(ctx, req)` | `email, password, device_id` | `access_token, refresh_token, expires_in, user` | 비밀번호 불일치 → `UNAUTHENTICATED` |
| A-FN-03 | `RefreshToken(ctx, req)` | `refresh_token` | `new_access_token, new_refresh_token` | 폐기 토큰 → `UNAUTHENTICATED` |
| A-FN-04 | `Logout(ctx, req)` | `access_token or refresh_token` | `revoked=true` | 토큰 형식 오류 → `INVALID_ARGUMENT` |
| A-FN-05 | `Authorize(ctx, req)` | `user_id, role, action, resource` | `allow=true/false, reason` | 정책 미정의 → `INTERNAL` |
| A-FN-06 | `UpdateProfile(ctx, req)` | `name, phone, locale, notification_opt_out` | `updated_profile` | 유효성 오류 → `INVALID_ARGUMENT` |

---

## 4. DB 스키마

파일 위치: `infra/mysql/init/01_users.sql`

```sql
CREATE TABLE IF NOT EXISTS users (
  id                  CHAR(36)      NOT NULL,
  email               VARCHAR(255)  NOT NULL,
  password_hash       VARCHAR(60)   NOT NULL,  -- bcrypt, cost=12
  name                VARCHAR(100)  NOT NULL,
  phone               VARCHAR(20)   DEFAULT NULL,
  role                ENUM('CUSTOMER','OWNER','SUPER_ADMIN') NOT NULL DEFAULT 'CUSTOMER',
  no_show_count       INT           NOT NULL DEFAULT 0,
  is_restricted       TINYINT(1)    NOT NULL DEFAULT 0,
  restricted_until    DATETIME      DEFAULT NULL,
  notification_opt_out TINYINT(1)   NOT NULL DEFAULT 0,
  locale              VARCHAR(10)   NOT NULL DEFAULT 'ko-KR',
  created_at          DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at          DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uq_users_email (email),
  KEY idx_users_role (role)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 로그인 실패 카운터는 Redis에서 관리 (아래 Redis 섹션 참고)
-- 소셜 로그인은 별도 테이블로 분리 (MVP 범위 외, 추후 확장)
```

---

## 5. JWT 구조 및 토큰 관리

### 5-1. Access Token (JWT)

```
Header: { "alg": "HS256", "typ": "JWT" }

Payload:
{
  "sub":  "usr_abc123",          // user_id (UUID)
  "tid":  "tenant_xyz",          // tenant_id (= owner_id, CUSTOMER는 user_id 사용)
  "rol":  "OWNER",               // UserRole enum 문자열
  "iat":  1742800000,
  "exp":  1742800900,            // iat + 900 (15분)
  "jti":  "unique-token-id"      // UUID v4, 블랙리스트 키로 사용
}
```

### 5-2. Refresh Token

- Opaque token (UUID v4 생성)
- Redis에 저장: `auth:refresh:{user_id}:{device_id}` → refresh_token 값
- TTL: 7일 (168시간)
- RTR(Refresh Token Rotation): `RefreshToken()` 호출 시 기존 토큰 삭제 + 새 토큰 발급
- 탈취 감지: 폐기된 refresh_token으로 재시도 시 해당 user_id의 모든 refresh_token 삭제

### 5-3. 토큰 블랙리스트 (로그아웃)

- Redis 키: `auth:blacklist:{jti}` → "1"
- TTL: access_token의 잔여 만료시간 (expire_at - now)
- 모든 서비스의 `auth.go` 인터셉터는 요청마다 이 키를 확인

---

## 6. Redis 키 패턴 (내 담당)

| 키 | TTL | 값 | 설명 |
|----|-----|----|------|
| `auth:blacklist:{jti}` | access_token 잔여 만료시간 | `"1"` | 로그아웃된 토큰 |
| `auth:refresh:{user_id}:{device_id}` | 7일 | refresh_token 값 | 유효한 refresh_token |
| `auth:login_fail:{email}` | 1시간 | 실패 횟수 (int) | rate limit 카운터 |

```go
// Redis 키 생성 함수 예시
func BlacklistKey(jti string) string {
    return fmt.Sprintf("auth:blacklist:%s", jti)
}

func RefreshTokenKey(userID, deviceID string) string {
    return fmt.Sprintf("auth:refresh:%s:%s", userID, deviceID)
}

func LoginFailKey(email string) string {
    return fmt.Sprintf("auth:login_fail:%s", email)
}
```

---

## 7. RBAC 정책 테이블

> `rbac.go`에 아래 매트릭스를 코드로 구현합니다.
> `Authorize(userID, role, action, resource)` 호출 시 이 테이블을 참조합니다.

| action | CUSTOMER | OWNER | SUPER_ADMIN | 비고 |
|--------|----------|-------|-------------|------|
| `reservation:create` | ✅ (본인) | ❌ | ❌ | is_restricted=true이면 CUSTOMER도 거부 |
| `reservation:read` | ✅ (본인) | ✅ (본인 매장) | ✅ | |
| `reservation:confirm` | ❌ | ✅ (본인 매장) | ✅ | |
| `reservation:cancel` | ✅ (본인, 방문 1시간 전) | ✅ (본인 매장) | ✅ | |
| `reservation:noshow` | ❌ | ✅ (본인 매장) | ✅ | |
| `restaurant:create` | ❌ | ✅ (본인) | ✅ | |
| `restaurant:update` | ❌ | ✅ (본인 tenant) | ✅ | tenant_mismatch 시 reason 포함 |
| `restaurant:approve` | ❌ | ❌ | ✅ | |
| `waiting:join` | ✅ | ❌ | ❌ | |
| `waiting:call` | ❌ | ✅ (본인 매장) | ✅ | |
| `menu:manage` | ❌ | ✅ (본인 매장) | ✅ | |
| `admin:audit_read` | ❌ | ❌ | ✅ | |
| `profile:read` | ✅ (본인) | ✅ (본인) | ✅ | |
| `profile:update` | ✅ (본인) | ✅ (본인) | ✅ | |

```go
// backend/internal/user/rbac.go

type PolicyRule struct {
    Action       string
    AllowedRoles []string
    SelfOnly     bool    // true: resource가 본인 소유인지 추가 확인 필요
}

var PolicyTable = []PolicyRule{
    {Action: "reservation:create", AllowedRoles: []string{"CUSTOMER"}, SelfOnly: true},
    {Action: "reservation:confirm", AllowedRoles: []string{"OWNER", "SUPER_ADMIN"}},
    {Action: "reservation:cancel", AllowedRoles: []string{"CUSTOMER", "OWNER", "SUPER_ADMIN"}, SelfOnly: true},
    {Action: "reservation:noshow", AllowedRoles: []string{"OWNER", "SUPER_ADMIN"}},
    {Action: "restaurant:create", AllowedRoles: []string{"OWNER", "SUPER_ADMIN"}},
    {Action: "restaurant:update", AllowedRoles: []string{"OWNER", "SUPER_ADMIN"}, SelfOnly: true},
    {Action: "restaurant:approve", AllowedRoles: []string{"SUPER_ADMIN"}},
    {Action: "waiting:join", AllowedRoles: []string{"CUSTOMER"}},
    {Action: "waiting:call", AllowedRoles: []string{"OWNER", "SUPER_ADMIN"}, SelfOnly: true},
    {Action: "menu:manage", AllowedRoles: []string{"OWNER", "SUPER_ADMIN"}, SelfOnly: true},
    {Action: "admin:audit_read", AllowedRoles: []string{"SUPER_ADMIN"}},
    {Action: "profile:read", AllowedRoles: []string{"CUSTOMER", "OWNER", "SUPER_ADMIN"}, SelfOnly: true},
    {Action: "profile:update", AllowedRoles: []string{"CUSTOMER", "OWNER", "SUPER_ADMIN"}, SelfOnly: true},
}
```

---

## 8. gRPC Auth Interceptor (B/C/D/E가 import하는 인터페이스)

파일 위치: `backend/pkg/middleware/auth.go`

이 파일은 **A 담당자가 작성**하고, B/C/D/E는 이 인터셉터를 서버 시작 시 등록만 합니다.

```go
// backend/pkg/middleware/auth.go
package middleware

import (
    "context"
    "strings"

    "google.golang.org/grpc"
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/metadata"
    "google.golang.org/grpc/status"
)

// AuthContext — 인터셉터가 context에 주입하는 값
type AuthContext struct {
    UserID   string
    TenantID string
    Role     string
}

type contextKey string
const authContextKey contextKey = "auth_context"

// GetAuthContext — 핸들러에서 인증 정보 추출
func GetAuthContext(ctx context.Context) (*AuthContext, bool) {
    v, ok := ctx.Value(authContextKey).(*AuthContext)
    return v, ok
}

// UnaryAuthInterceptor — gRPC 서버에 등록
// 사용법: grpc.NewServer(grpc.ChainUnaryInterceptor(middleware.UnaryAuthInterceptor(jwtSecret, redisClient, skipMethods)))
func UnaryAuthInterceptor(jwtSecret string, redis RedisClient, skipMethods []string) grpc.UnaryServerInterceptor {
    return func(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
        // skip 목록 (Register, Login, RefreshToken)
        for _, m := range skipMethods {
            if strings.HasSuffix(info.FullMethod, m) {
                return handler(ctx, req)
            }
        }

        md, ok := metadata.FromIncomingContext(ctx)
        if !ok {
            return nil, status.Error(codes.Unauthenticated, "missing metadata")
        }

        tokens := md.Get("authorization")
        if len(tokens) == 0 {
            return nil, status.Error(codes.Unauthenticated, "missing authorization header")
        }

        token := strings.TrimPrefix(tokens[0], "Bearer ")
        claims, err := validateJWT(token, jwtSecret)
        if err != nil {
            return nil, status.Errorf(codes.Unauthenticated, "invalid token: %v", err)
        }

        // 블랙리스트 확인
        if blacklisted, _ := redis.Exists(ctx, "auth:blacklist:"+claims.JTI); blacklisted {
            return nil, status.Error(codes.Unauthenticated, "token revoked")
        }

        // context에 인증 정보 주입
        authCtx := &AuthContext{
            UserID:   claims.Subject,
            TenantID: claims.TenantID,
            Role:     claims.Role,
        }
        newCtx := context.WithValue(ctx, authContextKey, authCtx)

        // gRPC metadata에도 주입 (서비스 간 전달용)
        newMD := metadata.Pairs(
            "x-user-id",   claims.Subject,
            "x-tenant-id", claims.TenantID,
            "x-role",      claims.Role,
        )
        newCtx = metadata.NewOutgoingContext(newCtx, newMD)

        return handler(newCtx, req)
    }
}

// B/C/D/E 서버 등록 예시:
// var skipMethods = []string{"Register", "Login", "RefreshToken", "GetAvailability"}
// server := grpc.NewServer(
//     grpc.ChainUnaryInterceptor(
//         middleware.UnaryAuthInterceptor(cfg.JWTSecret, redisClient, skipMethods),
//         middleware.UnaryLoggingInterceptor(),   // E 담당
//         middleware.UnaryMetricsInterceptor(),   // E 담당
//     ),
// )
```

---

## 9. 주차별 상세 작업 + 완료 조건

| 주차 | 작업 내용 | 완료 조건 |
|------|---------|---------|
| W1 | JWT 구조 확정 + RBAC 정책 테이블 코드화 | `Authorize()` 정책 100% 코드 반영, 타 담당자 리뷰 승인 |
| W2 | Register/Login/RefreshToken/Logout API 구현 | 성공/실패 단위테스트 **25개 이상** 통과 |
| W3 | `auth.go` interceptor 구현 + B/C/D/E에 배포 | B/C/D/E 서비스에서 인터셉터 적용 후 smoke test 통과 |
| W4 | Profile API + FE 연동 | FE에서 토큰 만료 시 자동 재발급 후 재시도 성공 |
| W5 | 로그인 rate limit + 노쇼 계정 제한 로직 | rate limit 초과 테스트 통과, is_restricted 체크 연동 |
| W6 | CI smoke test 자동화 | GitHub Actions에서 auth smoke 5분 이내 완료 |
| W7~W9 | 보안 회귀 테스트 + UAT 대응 | 토큰 재사용/위조 테스트 실패율 0% |
| W10 | 운영 모니터링 튜닝 | 로그인 실패율 대시보드/알람 동작 확인 |

---

## 10. 교차 점검 항목 (구체)

| 교차 | 점검 내용 | 검증 방법 |
|------|---------|---------|
| A↔C | `x-user-id` 누락 시 C의 모든 API가 즉시 `UNAUTHENTICATED` 반환 | C 쪽 integration test에서 metadata 없이 호출 |
| A↔B | 점주가 타 테넌트 restaurant_id 수정 시 `PERMISSION_DENIED` + `reason: tenant_mismatch` | B 쪽 테스트에서 다른 tenant_id로 수정 시도 |
| A↔E | 모든 관리자 action의 audit log에 `actor_id`, `actor_role`, `request_id` 3필드 적재 | E의 audit log 조회 API로 확인 |
| A↔D | 사용자가 `notification_opt_out=true`이면 D가 마케팅성 알림 발송 금지 | D 쪽 SendNotification에서 opt_out 상태 확인 로직 |

---

## 11. 상세 테스트 케이스 목록

### Register (A-FN-01)
- [ ] 정상 회원가입 (CUSTOMER)
- [ ] 정상 회원가입 (OWNER)
- [ ] 이메일 중복 → `ALREADY_EXISTS`
- [ ] 이메일 형식 오류 → `INVALID_ARGUMENT`
- [ ] 비밀번호 7자 미만 → `INVALID_ARGUMENT`
- [ ] SUPER_ADMIN 역할로 회원가입 시도 → `PERMISSION_DENIED`

### Login (A-FN-02)
- [ ] 정상 로그인 → access/refresh token 모두 발급
- [ ] 잘못된 비밀번호 → `UNAUTHENTICATED`
- [ ] 존재하지 않는 이메일 → `UNAUTHENTICATED` (이메일 존재 여부 노출 금지)
- [ ] 5회 실패 → rate limit, `RESOURCE_EXHAUSTED`
- [ ] rate limit 1시간 후 자동 해제
- [ ] is_restricted=true 계정 로그인 → 로그인 성공, 예약 시 차단 (로그인 자체는 허용)

### RefreshToken (A-FN-03)
- [ ] 유효한 refresh_token → 새 access/refresh token 발급, 기존 refresh_token 폐기
- [ ] 폐기된 refresh_token 재사용 → `UNAUTHENTICATED` + 해당 user 모든 토큰 강제 폐기
- [ ] 만료된 refresh_token → `UNAUTHENTICATED`
- [ ] 형식이 잘못된 refresh_token → `INVALID_ARGUMENT`

### Logout (A-FN-04)
- [ ] access_token으로 로그아웃 → jti 블랙리스트 등록
- [ ] 로그아웃 후 해당 access_token으로 API 호출 → `UNAUTHENTICATED`
- [ ] refresh_token으로 로그아웃 → Redis에서 해당 refresh_token 삭제

### Authorize (A-FN-05)
- [ ] CUSTOMER가 `reservation:create` → allow=true
- [ ] CUSTOMER가 `reservation:confirm` → allow=false
- [ ] OWNER가 본인 테넌트 `restaurant:update` → allow=true
- [ ] OWNER가 타 테넌트 `restaurant:update` → allow=false + reason=tenant_mismatch
- [ ] is_restricted=true CUSTOMER가 `reservation:create` → allow=false + reason=account_restricted

### UpdateProfile (A-FN-06)
- [ ] 정상 업데이트 (name, phone)
- [ ] 타인 프로필 수정 시도 → `PERMISSION_DENIED`
- [ ] 전화번호 형식 오류 → `INVALID_ARGUMENT`

---

## 12. 정량 목표 (SLO/품질)

| 지표 | 목표값 |
|------|--------|
| 인증 API p95 응답 시간 | < 120ms |
| 로그인 성공률 | ≥ 99.5% |
| 권한 우회 취약점 | 0건 |
| Auth 관련 P1 버그 | 0건 |
| 토큰 재사용 공격 탐지율 | 100% |
