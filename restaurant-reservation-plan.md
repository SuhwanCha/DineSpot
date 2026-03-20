# 식당 예약 시스템 — 프로젝트 계획서

> 작성일: 2026-03-17

---

## 1. 프로젝트 개요

| 항목 | 내용 |
|------|------|
| 서비스명 | 식당 예약 플랫폼 |
| 서비스 유형 | 다중 식당 멀티테넌트 예약 플랫폼 |
| 주요 사용자 | 고객 / 식당 오너 / 슈퍼관리자 |
| 배포 환경 | Cloud (AWS 또는 GCP) |
| 개발 방식 | AI-driven (Claude Code + MCP 기반) |

---

## 2. 핵심 기능 요구사항

### 고객 (Customer)
- 식당 검색 및 상세 조회 (위치, 메뉴, 운영시간)
- 날짜/시간/인원 기반 테이블 예약
- 실시간 웨이팅 등록 및 순번 알림
- 예약 시 메뉴 사전 주문 및 결제
- 예약 내역 조회/취소

### 식당 오너 (Restaurant Owner)
- 식당 등록 및 프로필 관리
- 테이블 레이아웃 및 운영 시간대 설정
- 예약 요청 승인/거절/수정
- 웨이팅 큐 관리 (입장 처리, 순번 조정)
- 메뉴 등록/수정/삭제
- 매출 및 예약 현황 대시보드

### 슈퍼관리자
- 식당 심사 및 등록 승인
- 플랫폼 전체 모니터링

---

## 3. 비기능 요구사항

### 동시 접속자 제어 (방화벽)
- **방식**: Redis 기반 **Connection Queue**
- 동시 처리 가능한 요청 수에 상한선을 두고, 초과 요청은 큐에 적재
- 처리 완료 시 순차적으로 해제
- Cloud WAF + API Gateway 레이어에서 적용
- 큐 대기 중인 클라이언트에 실시간 대기 상태 반환 (SSE 또는 WebSocket)

### 성능
- 예약 생성 API: p99 응답시간 < 200ms
- 웨이팅 큐 업데이트: 실시간 반영 (지연 < 1s)

### 보안
- JWT 기반 인증
- 멀티테넌트 데이터 격리 (식당별 데이터 접근 분리)
- HTTPS + gRPC TLS

---

## 4. 기술 스택

### Frontend

| 항목 | 기술 |
|------|------|
| UI 디자인 | **Figma Make** |
| 프레임워크 | **Next.js** (App Router) |
| 언어 | TypeScript |
| 스타일 | Tailwind CSS |
| gRPC 통신 | gRPC-Web (protobuf-es) |
| 상태 관리 | Zustand / TanStack Query |

### Backend

| 항목 | 기술 |
|------|------|
| 언어 | **Go** |
| RPC 프레임워크 | **gRPC** (protobuf) |
| API Gateway | Envoy Proxy (gRPC-Web ↔ gRPC 변환) |
| 인증 | JWT + gRPC interceptor |

### 데이터베이스

| 항목 | 기술 |
|------|------|
| 주 DB | **MySQL 8.x** |
| 캐시 / 큐 / 세션 | **Redis 7.x** |
| 파일 스토리지 | AWS S3 / GCS |

### 인프라

| 항목 | 기술 |
|------|------|
| 컨테이너 | Docker + docker-compose (로컬) / K8s (프로덕션) |
| IaC | Terraform |
| CI/CD | GitHub Actions |
| WAF | AWS WAF / Cloud Armor |
| 모니터링 | Prometheus + Grafana |

### MCP Server (개발 편의)

| 항목 | 내용 |
|------|------|
| 목적 | gRPC 백엔드를 MCP 서버로 노출 → 프론트 개발 시 Claude가 API 직접 호출 가능 |
| 구현 | Go로 MCP 서버 작성, proto 스키마 기반 tool 자동 생성 |
| 사용 | `claude mcp add` 로 로컬 개발 환경에 등록 |

---

## 5. 시스템 아키텍처

```
Client (Browser)
      │
      ▼
  CDN (CloudFront / Cloud CDN)
      │
      ▼
  WAF + API Gateway
  ┌──────────────────────────┐
  │   Connection Queue       │  ← Redis 기반, 동시 접속 상한 제어
  │   (Rate Controller)      │
  └──────────────────────────┘
      │
  ┌───┴──────────────────┐
  ▼                      ▼
Next.js              Envoy Proxy
(SSR / Static)       (gRPC-Web → gRPC)
                          │
           ┌──────────────┼──────────────┐
           ▼              ▼              ▼
    Restaurant Svc  Reservation Svc  Waiting Svc
    (Go / gRPC)     (Go / gRPC)      (Go / gRPC)
           │              │              │
           └──────────────┼──────────────┘
                          ▼
                    MySQL + Redis
```

---

## 6. gRPC Proto 스키마 구조

```
proto/
├── common/
│   └── types.proto          # 공통 타입 (Pagination, Error 등)
├── restaurant/
│   └── restaurant.proto     # RestaurantService (CRUD, 검색)
├── reservation/
│   └── reservation.proto    # ReservationService (예약 생성/조회/취소)
├── waiting/
│   └── waiting.proto        # WaitingService (큐 등록/조회/입장)
├── menu/
│   └── menu.proto           # MenuService (메뉴 CRUD, 사전 주문)
└── user/
    └── user.proto           # UserService (회원가입/로그인/프로필)
```

---

## 7. 프로젝트 디렉토리 구조

```
restaurant-reservation/
├── frontend/                    # Next.js 앱
│   ├── src/
│   │   ├── app/                 # App Router 페이지
│   │   ├── components/          # UI 컴포넌트 (Figma Make 생성)
│   │   └── generated/           # protobuf-es 자동 생성 코드
│   └── package.json
├── backend/                     # Go gRPC 서비스
│   ├── cmd/
│   └── internal/
│       ├── restaurant/
│       ├── reservation/
│       ├── waiting/
│       ├── menu/
│       └── user/
├── proto/                       # Protocol Buffer 정의
├── mcp-server/                  # MCP 서버 (개발용)
│   └── main.go
├── infra/                       # Terraform IaC
│   ├── modules/
│   └── environments/
├── docker-compose.yml           # 로컬 개발 환경
└── .github/
    └── workflows/               # CI/CD
```

---

## 8. 개발 단계 (Phase)

| Phase | 내용 | 주요 산출물 |
|-------|------|------------|
| **P0** | 프로젝트 초기화 | repo 구조, docker-compose, proto 스키마 전체 정의 |
| **P1** | 백엔드 gRPC 서비스 구현 | User / Restaurant / Reservation / Waiting / Menu 서비스 |
| **P2** | Connection Queue 미들웨어 | Redis 큐 로직 + Envoy Proxy 설정 |
| **P3** | MCP 서버 | gRPC → MCP 브릿지, Claude Code 연동 |
| **P4** | UI 디자인 | Figma Make로 핵심 화면 설계 |
| **P5** | 프론트엔드 구현 | Next.js, MCP 활용해 Claude와 함께 개발 |
| **P6** | 인프라 | Terraform, WAF, CI/CD 파이프라인 |

---

## 9. 검증 방법

| 항목 | 방법 |
|------|------|
| 로컬 실행 | `docker-compose up` → gRPC + MySQL + Redis 기동 확인 |
| Proto 검증 | `buf lint` + `buf generate` 스키마 유효성 확인 |
| MCP 동작 | `claude mcp add mcp-server` 후 Claude에서 도구 호출 테스트 |
| Connection Queue | `k6` / `hey` 부하 테스트 → 큐 대기 동작 확인 |
| E2E | Playwright로 예약 플로우 자동화 테스트 |

---

## 10. 미결 사항

- [ ] 결제 게이트웨이 연동 범위 (토스페이먼츠, 아임포트 등)
- [ ] 알림 채널 우선순위 (카카오 알림톡 / SMS / 이메일)
- [ ] 식당 오너 전용 모바일 앱 필요 여부
- [ ] 멀티 리전 배포 필요 여부
- [ ] AWS vs GCP 최종 결정
