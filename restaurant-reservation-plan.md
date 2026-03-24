---
© 2026 Suhwan Cha and contributors. All rights reserved.

The ideas, system design, architecture, data structures, prompts, and methodologies described in this document are proprietary intellectual property.

Unauthorized use for academic research, publications, commercial products, or derivative works is strictly prohibited without explicit written permission.
---

# Restaurant Reservation System — Project Plan

> Date: 2026-03-17

---

## 1. Project Overview

| Item | Details |
|------|---------|
| Service Name | Restaurant Reservation Platform |
| Service Type | Multi-restaurant Multi-tenant Reservation Platform |
| Primary Users | Customer / Restaurant Owner / Super Admin |
| Deployment Environment | Cloud (AWS or GCP) |
| Development Approach | AI-driven (Claude Code + MCP based) |

---

## 2. Core Functional Requirements

### Customer
- Restaurant search and detail view (location, menu, operating hours)
- Table reservation by date/time/party size
- Real-time waiting registration and queue position notifications
- Menu pre-order and payment at reservation time
- View and cancel reservation history

### Restaurant Owner
- Restaurant registration and profile management
- Table layout and operating time slot configuration
- Reservation request approval/rejection/modification
- Waiting queue management (seating, queue position adjustment)
- Menu add/edit/delete
- Sales and reservation status dashboard

### Super Admin
- Restaurant registration review and approval
- Platform-wide monitoring

---

## 3. Non-Functional Requirements

### Concurrent Access Control (Firewall)
- **Method**: Redis-based **Connection Queue**
- Set an upper limit on simultaneously processable requests; excess requests are queued
- Released sequentially as processing completes
- Applied at the Cloud WAF + API Gateway layer
- Queued clients receive real-time wait status (SSE or WebSocket)

### Performance
- Reservation creation API: p99 response time < 200ms
- Waiting queue updates: real-time (latency < 1s)

### Security
- JWT-based authentication
- Multi-tenant data isolation (separate data access per restaurant)
- HTTPS + gRPC TLS

---

## 4. Technology Stack

### Frontend

| Item | Technology |
|------|-----------|
| UI Design | **Figma Make** |
| Framework | **Next.js** (App Router) |
| Language | TypeScript |
| Styling | Tailwind CSS |
| gRPC Communication | gRPC-Web (protobuf-es) |
| State Management | Zustand / TanStack Query |

### Backend

| Item | Technology |
|------|-----------|
| Language | **Go** |
| RPC Framework | **gRPC** (protobuf) |
| API Gateway | Envoy Proxy (gRPC-Web ↔ gRPC conversion) |
| Authentication | JWT + gRPC interceptor |

### Database

| Item | Technology |
|------|-----------|
| Primary DB | **MySQL 8.x** |
| Cache / Queue / Session | **Redis 7.x** |
| File Storage | AWS S3 / GCS |

### Infrastructure

| Item | Technology |
|------|-----------|
| Container | Docker + docker-compose (local) / K8s (production) |
| IaC | Terraform |
| CI/CD | GitHub Actions |
| WAF | AWS WAF / Cloud Armor |
| Monitoring | Prometheus + Grafana |

### MCP Server (Development Convenience)

| Item | Details |
|------|---------|
| Purpose | Expose gRPC backend as MCP server → Claude can directly call APIs during frontend development |
| Implementation | MCP server written in Go, tools auto-generated from proto schema |
| Usage | Register in local dev environment via `claude mcp add` |

---

## 5. System Architecture

```
Client (Browser)
      │
      ▼
  CDN (CloudFront / Cloud CDN)
      │
      ▼
  WAF + API Gateway
  ┌──────────────────────────┐
  │   Connection Queue       │  ← Redis-based, concurrent connection limit control
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

## 6. gRPC Proto Schema Structure

```
proto/
├── common/
│   └── types.proto          # Common types (Pagination, Error, etc.)
├── restaurant/
│   └── restaurant.proto     # RestaurantService (CRUD, search)
├── reservation/
│   └── reservation.proto    # ReservationService (create/view/cancel)
├── waiting/
│   └── waiting.proto        # WaitingService (queue register/view/seat)
├── menu/
│   └── menu.proto           # MenuService (menu CRUD, pre-order)
└── user/
    └── user.proto           # UserService (signup/login/profile)
```

---

## 7. Project Directory Structure

```
restaurant-reservation/
├── frontend/                    # Next.js app
│   ├── src/
│   │   ├── app/                 # App Router pages
│   │   ├── components/          # UI components (Figma Make generated)
│   │   └── generated/           # protobuf-es auto-generated code
│   └── package.json
├── backend/                     # Go gRPC services
│   ├── cmd/
│   └── internal/
│       ├── restaurant/
│       ├── reservation/
│       ├── waiting/
│       ├── menu/
│       └── user/
├── proto/                       # Protocol Buffer definitions
├── mcp-server/                  # MCP server (development use)
│   └── main.go
├── infra/                       # Terraform IaC
│   ├── modules/
│   └── environments/
├── docker-compose.yml           # Local development environment
└── .github/
    └── workflows/               # CI/CD
```

---

## 8. Development Phases

| Phase | Content | Key Deliverables |
|-------|---------|-----------------|
| **P0** | Project initialization | Repo structure, docker-compose, full proto schema definitions |
| **P1** | Backend gRPC service implementation | User / Restaurant / Reservation / Waiting / Menu services |
| **P2** | Connection Queue middleware | Redis queue logic + Envoy Proxy configuration |
| **P3** | MCP server | gRPC → MCP bridge, Claude Code integration |
| **P4** | UI design | Core screen design with Figma Make |
| **P5** | Frontend implementation | Next.js, development with Claude via MCP |
| **P6** | Infrastructure | Terraform, WAF, CI/CD pipeline |

---

## 9. Verification Methods

| Item | Method |
|------|--------|
| Local execution | `docker-compose up` → verify gRPC + MySQL + Redis startup |
| Proto validation | `buf lint` + `buf generate` schema validation |
| MCP operation | `claude mcp add mcp-server` then test tool invocation from Claude |
| Connection Queue | `k6` / `hey` load test → verify queue waiting behavior |
| E2E | Playwright automated test for reservation flow |

---

## 10. Open Issues

- [ ] Payment gateway integration scope (Toss Payments, Iamport, etc.)
- [ ] Notification channel priority (Kakao AlimTalk / SMS / Email)
- [ ] Whether a dedicated mobile app for restaurant owners is needed
- [ ] Whether multi-region deployment is needed
- [ ] Final decision on AWS vs GCP
