---
© 2026 Suhwan Cha and contributors. All rights reserved.

The ideas, system design, architecture, data structures, prompts, and methodologies described in this document are proprietary intellectual property.

Unauthorized use for academic research, publications, commercial products, or derivative works is strictly prohibited without explicit written permission.
---

# Restaurant Reservation System — Product Requirements Document (PRD)

> Version: v1.0
> Date: 2026-03-20
> Category: Hospitality / Travel → Reservation Domain

---

## Suitability Checklist

> Judgment criteria: All 3 required items met + at least 1 recommended → **Suitable**

| No. | Criteria | Type | Result |
|-----|----------|------|--------|
| 1 | Does the Problem Description justify the Client/Server separation? | ★ Required | ✅ |
| 2 | Are there 2 or more clearly distinct Actors? | ★ Required | ✅ |
| 3 | Are there 2 or more alternative flows implied beyond the normal flow? | ★ Required | ✅ |
| 4 | Do multiple users simultaneously access the same resource? | ◎ Recommended | ✅ |
| 5 | Does the core control flow vary based on state? | ◎ Recommended | ✅ |

**Verdict: Suitable (all 3 required + 2 recommended items met)**

---

## Part 1. Problem Description

### ① System Overview

A **distributed multi-tenant restaurant reservation platform** that connects multiple restaurants and customers. Customer devices (web/app) and restaurant management terminals connect to a central server to handle table reservations, real-time waiting queues, and pre-order menus.

Customers can view real-time table availability and make reservations from any device. Restaurant owners manage reservations via a dedicated management terminal (web dashboard or POS integration). All reservation data is consistently managed on the central server, and concurrent access conflicts are controlled in server-side critical sections.

---

### ② Hardware Configuration

| Terminal Type | I/O Devices | Role |
|---------------|-------------|------|
| **Customer Terminal** (Smartphone / PC Web) | Touchscreen / Keyboard + Mouse, Display | Restaurant search, reservation & waiting registration, menu ordering |
| **Restaurant Management Terminal** (PC Web Dashboard) | Keyboard + Mouse, Display | Reservation approval/rejection, waiting queue management, menu settings |
| **Restaurant Kiosk** (Optional) | Touchscreen, Receipt Printer, QR Scanner | On-site waiting registration, reservation confirmation |
| **Central Server** (Cloud) | Network Interface | gRPC API processing, DB management, queue control |
| **Notification Gateway** | Network Interface | SMS / Kakao AlimTalk / Email dispatch |

---

### ③ Actors and Roles

| Actor | Type | Interaction with System |
|-------|------|------------------------|
| **Customer** | Human | Restaurant search & browse, table reservation, waiting registration, menu pre-order, reservation cancellation |
| **Restaurant Owner** | Human | Restaurant info/table/menu management, reservation approval/rejection, waiting seating, sales inquiry |
| **Super Admin** | Human | Restaurant registration review & approval, platform-wide monitoring, policy configuration |
| **Notification Gateway** | External System | Sends reservation confirmed/cancelled/waiting notifications via SMS, Kakao AlimTalk, Email |
| **Payment System** | External System | Handles pre-order payment approval, cancellation, and refunds |

---

### ④ Core Features (Use Case Candidates)

#### UC-01. Restaurant Search and Browse
- **Actor**: Customer
- **Normal Flow Condition**: At least 1 restaurant matches the search criteria (area, food type, date) and is in an active operating state

#### UC-02. Table Reservation Request
- **Actor**: Customer
- **Normal Flow Condition**: An available table exists for the requested date/time, the customer account is in good standing (not deactivated or suspended), and no duplicate reservation exists for the same time slot

#### UC-03. Reservation Approval/Rejection
- **Actor**: Restaurant Owner
- **Normal Flow Condition**: Reservation status is PENDING and table availability exists for the requested time → transition to CONFIRMED

#### UC-04. Waiting Registration
- **Actor**: Customer, Restaurant Kiosk
- **Normal Flow Condition**: The restaurant is currently open, the waiting queue is below the maximum capacity → register in queue and issue a sequence number

#### UC-05. Waiting Seating Process
- **Actor**: Restaurant Owner
- **Normal Flow Condition**: The first customer in queue is in WAITING status, and the customer has received the notification (READY status) → transition to SEATED

#### UC-06. Menu Pre-Order
- **Actor**: Customer
- **Normal Flow Condition**: Reservation status is CONFIRMED, the restaurant allows pre-orders, and the selected menu is not sold out → accept order and proceed to payment

#### UC-07. Reservation Cancellation
- **Actor**: Customer, Restaurant Owner
- **Normal Flow Condition**: Reservation status is PENDING or CONFIRMED, and the cancellation window (N hours before visit) has not passed → process cancellation and refund

#### UC-08. Restaurant Registration and Approval
- **Actor**: Restaurant Owner (application), Super Admin (review)
- **Normal Flow Condition**: All required documents are complete and no duplicate business registration number exists → review and transition to ACTIVE

#### UC-09. Sales and Reservation Dashboard
- **Actor**: Restaurant Owner, Super Admin
- **Normal Flow Condition**: Authenticated owner account, data exists within the requested period → return aggregated data

---

### ⑤ Alternative Flow Conditions

| Alt Flow ID | Trigger Condition | System Response |
|-------------|-------------------|-----------------|
| **AF-01** Full | No available table for the requested date/time | Reject reservation, suggest waiting registration |
| **AF-02** Duplicate Reservation | Customer already has a reservation for the same date/time | Reject reservation, return existing reservation info |
| **AF-03** Cancellation Deadline Exceeded | Cancellation requested within 1 hour of visit time | Reject cancellation or apply cancellation fee |
| **AF-04** Payment Failure | Card limit exceeded or payment declined during pre-order | Keep order in PAYMENT_PENDING state, prompt re-payment |
| **AF-05** Waiting No-Response | No response within N minutes of seating notification | READY → NO_SHOW transition, call next in queue |
| **AF-06** Waiting Queue Full | Maximum waiting capacity exceeded at registration | Reject registration, provide estimated wait time |
| **AF-07** Menu Out of Stock | Selected menu item has no stock during pre-order | Disable that menu item, suggest alternatives |
| **AF-08** Owner Rejection | Owner manually rejects a reservation request | PENDING → REJECTED transition, notify customer |
| **AF-09** No-Show | Customer does not arrive after visit time passes | CONFIRMED → NO_SHOW transition, apply penalty |
| **AF-10** Unauthorized Access | API call with missing or expired authentication token | Return 401 Unauthorized, prompt re-login |

---

### ⑥ Concurrency Scenarios

Multiple customer devices may simultaneously attempt to reserve **the same table at the same restaurant on the same date and time**. For example, dozens of users pressing the reservation button simultaneously for a popular restaurant's 7 PM slot could result in duplicate table assignments.

**Critical Section Design:**

| Resource | Concurrent Access Scenario | Control Method |
|----------|---------------------------|----------------|
| Table Reservation Slot | Multiple customers simultaneously attempt to reserve the same time slot | Redis distributed lock (SETNX) + MySQL transaction SELECT FOR UPDATE |
| Waiting Queue Sequence | Multiple customers simultaneously register for waiting | Redis INCR atomic operation for sequence number issuance |
| Menu Stock | Multiple reservations simultaneously pre-order the same menu | MySQL atomic stock decrement (UPDATE ... WHERE stock > 0) |
| Connection Queue | Simultaneous connection surge | Redis-based queue + WAF connection limit control |

---

### ⑦ Data Management Location

All business data is **not stored on the client** but managed as a **Single Source of Truth on the central server**.

| Data Type | Storage Location | Notes |
|-----------|-----------------|-------|
| Member Info (Customer, Owner) | MySQL — `users` table | Password bcrypt hashed |
| Restaurant Info (Menu, Operating Hours, Tables) | MySQL — `restaurants`, `tables`, `menus` | Isolated per owner |
| Reservation Data | MySQL — `reservations` | Includes state transition history |
| Waiting Queue | Redis Sorted Set + MySQL (persistence) | Real-time order in Redis, history in MySQL |
| Session / Auth Token | Redis (with TTL) | Includes JWT blacklist |
| Distributed Lock | Redis (SETNX + TTL) | Prevents reservation slot conflicts |
| Image Files (Restaurant, Menu) | AWS S3 / GCS | Only URL stored in DB |
| Payment History | MySQL — `payments` | PG transaction ID stored |

---

## Part 2. Detailed Requirements Specification

### FR (Functional Requirements)

#### FR-USER: User Management

| ID | Requirement | Priority |
|----|------------|----------|
| FR-USER-01 | Customers can sign up with email+password or social login (Kakao, Google) | P0 |
| FR-USER-02 | Restaurant owners must go through a separate registration process including business information | P0 |
| FR-USER-03 | Upon successful login, issue JWT Access Token (15 min) and Refresh Token (7 days) | P0 |
| FR-USER-04 | Customers can update their profile (name, phone number, notification preferences) | P1 |
| FR-USER-05 | After 3 cumulative no-shows, the customer account is restricted from reservations for 30 days | P1 |

#### FR-RESTAURANT: Restaurant Management

| ID | Requirement | Priority |
|----|------------|----------|
| FR-REST-01 | Restaurant owners can register restaurant name, address, category, operating hours, and cover image | P0 |
| FR-REST-02 | Registered restaurants are only visible to customers after Super Admin approval (PENDING → ACTIVE) | P0 |
| FR-REST-03 | Owners can add tables and set each table's min/max capacity and available reservation time slots | P0 |
| FR-REST-04 | Owners can mark specific dates as holidays; reservations are disabled on those dates | P1 |
| FR-REST-05 | Customers can search restaurants by name, area, food category, date, and party size | P0 |
| FR-REST-06 | Real-time available table count is displayed on the restaurant detail page | P0 |

#### FR-RESERVATION: Reservation Management

| ID | Requirement | Priority |
|----|------------|----------|
| FR-RES-01 | Customers select date, time, and party size on the restaurant detail page to request a reservation | P0 |
| FR-RES-02 | On reservation request, the slot is pre-empted via Redis distributed lock; lock is released if not finalized within 30 seconds | P0 |
| FR-RES-03 | When an owner approves a reservation, a confirmation notification (Kakao AlimTalk or SMS) is sent to the customer | P0 |
| FR-RES-04 | When an owner rejects a reservation, a notification with the rejection reason is sent to the customer | P0 |
| FR-RES-05 | Customers can cancel up to 1 hour before visit time; pre-paid amounts are refunded | P0 |
| FR-RES-06 | An automatic visit reminder is sent to the customer 30 minutes before the visit time | P1 |
| FR-RES-07 | If no visit is confirmed within 1 hour after the visit time, the reservation is marked NO_SHOW and a penalty is applied | P1 |
| FR-RES-08 | Customers can view their reservation history (upcoming/completed/cancelled) as a list | P0 |

#### FR-WAITING: Waiting Queue Management

| ID | Requirement | Priority |
|----|------------|----------|
| FR-WAIT-01 | Customers can register for waiting when the restaurant is open and waiting is activated | P0 |
| FR-WAIT-02 | On waiting registration, a sequence number is atomically issued via Redis INCR; returns position and estimated wait time | P0 |
| FR-WAIT-03 | When the owner processes seating, a ready-to-enter notification is sent and status changes to READY | P0 |
| FR-WAIT-04 | If the customer does not respond within 10 minutes of notification, they are marked NO_SHOW and the next in queue is called | P0 |
| FR-WAIT-05 | Customers can cancel their waiting spot; estimated wait times for subsequent customers are updated | P0 |
| FR-WAIT-06 | Waiting status is updated in real-time via SSE (Server-Sent Events) | P1 |
| FR-WAIT-07 | Waiting can also be registered at the restaurant kiosk via QR code scan | P2 |

#### FR-MENU: Menu and Pre-Order

| ID | Requirement | Priority |
|----|------------|----------|
| FR-MENU-01 | Owners can register menu category, name, price, description, image, and stock status | P0 |
| FR-MENU-02 | Customers can pre-order menu items after reservation confirmation and before the visit | P1 |
| FR-MENU-03 | Payment must be completed for a pre-order to be confirmed | P1 |
| FR-MENU-04 | On payment failure, the order remains in PAYMENT_PENDING status and a re-payment link is sent | P1 |
| FR-MENU-05 | Out-of-stock menu items are shown as disabled in the ordering UI | P1 |

#### FR-ADMIN: Administrator

| ID | Requirement | Priority |
|----|------------|----------|
| FR-ADMIN-01 | Super Admin can view the restaurant registration request list and approve or reject applications | P0 |
| FR-ADMIN-02 | Super Admin can view overall reservation status, sales by restaurant, and no-show statistics | P1 |
| FR-ADMIN-03 | Super Admin can block specific IPs or accounts in case of abnormal traffic | P1 |

---

### NFR (Non-Functional Requirements)

#### NFR-PERF: Performance

| ID | Requirement |
|----|------------|
| NFR-PERF-01 | Reservation creation API response time: p99 < 200ms (under normal load) |
| NFR-PERF-02 | Restaurant search API response time: p99 < 300ms |
| NFR-PERF-03 | Waiting sequence number issuance: p99 < 50ms (Redis atomic operation) |
| NFR-PERF-04 | Target concurrent connections: 1,000 simultaneous connections |

#### NFR-CONCURRENCY: Concurrency and Consistency

| ID | Requirement |
|----|------------|
| NFR-CONC-01 | For concurrent requests on the same reservation slot, exactly 1 must succeed (0% duplicate reservations) |
| NFR-CONC-02 | Duplicate waiting sequence numbers must never be issued |
| NFR-CONC-03 | When the Connection Queue is exceeded, clients must receive real-time queue status updates |
| NFR-CONC-04 | Distributed lock TTL is set to 30 seconds and must auto-release on server failure |

#### NFR-AVAIL: Availability and Recovery

| ID | Requirement |
|----|------------|
| NFR-AVAIL-01 | Service availability: 99.9% or higher (monthly downtime < 44 minutes) |
| NFR-AVAIL-02 | Reservation data must maintain at least 1 MySQL Read Replica |
| NFR-AVAIL-03 | In case of Redis failure, reservation processing must be able to fallback to MySQL alone |

#### NFR-SEC: Security

| ID | Requirement |
|----|------------|
| NFR-SEC-01 | All APIs must use HTTPS (TLS 1.2+) and gRPC TLS for encrypted communication |
| NFR-SEC-02 | JWT Access Token expiry: 15 minutes / Refresh Token expiry: 7 days |
| NFR-SEC-03 | Restaurant owners can only access their own restaurant data (multi-tenant isolation) |
| NFR-SEC-04 | Passwords must be stored as bcrypt hash (cost=12 or higher) |
| NFR-SEC-05 | CSRF token validation applied to API requests (web clients) |

#### NFR-SCALE: Scalability

| ID | Requirement |
|----|------------|
| NFR-SCALE-01 | gRPC services must be horizontally scalable via Kubernetes HPA based on traffic |
| NFR-SCALE-02 | Failure of a single gRPC service must not impact other services |

---

## Part 3. State Transition Specification

### Reservation State Machine

```
                    Owner Approves
  [PENDING] ──────────────────────▶ [CONFIRMED]
      │                                  │
      │ Owner Rejects / Customer Cancel  │ Customer Cancel (up to 1hr before)
      ▼                                  ▼
  [REJECTED]                        [CANCELLED]

  [CONFIRMED] ─── Visit Completed ──▶ [COMPLETED]
      │
      └──── Visit Time + 1 Hour ────▶ [NO_SHOW]
```

| State | Description |
|-------|-------------|
| PENDING | Customer submitted reservation, awaiting owner confirmation |
| CONFIRMED | Owner approved the reservation |
| REJECTED | Owner rejected the reservation |
| CANCELLED | Cancelled by customer or owner |
| COMPLETED | Visit completed |
| NO_SHOW | No visit after visit time elapsed |

### Waiting State Machine

```
  [WAITING] ──── Owner Calls ────▶ [READY]
      │                                │
      │ Customer Cancel                │ 10 min no response
      ▼                                ▼
  [CANCELLED]                     [NO_SHOW]

  [READY] ──── Customer Enters ──▶ [SEATED]
```

| State | Description |
|-------|-------------|
| WAITING | Registered in the waiting queue |
| READY | Owner sent seating notification |
| SEATED | Customer has been seated |
| NO_SHOW | No response within 10 minutes of notification |
| CANCELLED | Customer directly cancelled |

### Restaurant Registration State Machine

```
  [PENDING_REVIEW] ── Approved ──▶ [ACTIVE]
          │                            │
          │ Rejected                   │ Owner deactivation request
          ▼                            ▼
      [REJECTED]                  [INACTIVE]
```

---

## Part 4. Technical Architecture

### System Diagram

```
Customer Terminal (Browser/Mobile)       Restaurant Management Terminal (Browser)
         │                                              │
         └──────────────────────┬──────────────────────┘
                                │ HTTPS
                                ▼
                    CDN (CloudFront / Cloud CDN)
                                │
                                ▼
                    WAF (AWS WAF / Cloud Armor)
                                │
                    ┌─────────────────────┐
                    │  Connection Queue   │  ← Redis-based concurrent connection limit
                    │  (API Gateway)      │
                    └─────────────────────┘
                                │
             ┌──────────────────┼──────────────┐
             ▼                                 ▼
        Next.js (SSR)                   Envoy Proxy
        Static Asset Serving            gRPC-Web → gRPC conversion
                                               │
          ┌────────────────────────────────────┼──────────────────────┐
          ▼                                    ▼                       ▼
   UserService                       ReservationService          WaitingService
   (Go / gRPC)                       (Go / gRPC)                 (Go / gRPC)
          ▼                                    ▼                       ▼
   RestaurantService              MenuService              NotificationService
   (Go / gRPC)                    (Go / gRPC)              (Go / gRPC)
          │                                    │                       │
          └────────────────────────────────────┼───────────────────────┘
                                               ▼
                                    ┌────────────────┐
                                    │  MySQL 8.x     │  (Primary + Read Replica)
                                    │  Redis 7.x     │  (Cache / Lock / Queue)
                                    │  AWS S3 / GCS  │  (Images)
                                    └────────────────┘
                                               │
                                    External System Integration
                                    ├── Payment PG (Toss Payments, etc.)
                                    └── Notification (Kakao AlimTalk / SMS)
```

### Technology Stack Summary

| Layer | Technology | Rationale |
|-------|-----------|-----------|
| Frontend | Next.js (App Router) + TypeScript + Tailwind CSS | SSR/SEO, gRPC-Web integration |
| UI Design | Figma Make | AI-driven UI generation |
| Backend | Go + gRPC (protobuf) | High performance, low latency, type safety |
| API Gateway | Envoy Proxy | gRPC-Web conversion, load balancing |
| Primary DB | MySQL 8.x | ACID transactions, reservation consistency |
| Cache/Queue/Lock | Redis 7.x | Distributed lock, sequence issuance, Connection Queue |
| Infrastructure | Docker / K8s / Terraform | Container orchestration, IaC |
| CI/CD | GitHub Actions | Automated build, test, deploy |
| MCP Server | Go (gRPC → MCP bridge) | Convenience for Claude Code frontend development |
| Monitoring | Prometheus + Grafana | Real-time metrics, alerting |

---

## Part 5. gRPC Proto Schema Structure

```
proto/
├── common/
│   └── types.proto           # Common types: Pagination, Error, Timestamp
├── user/
│   └── user.proto            # UserService: Register, Login, GetProfile, UpdateProfile
├── restaurant/
│   └── restaurant.proto      # RestaurantService: Create, Update, Search, GetDetail, Approve
├── table/
│   └── table.proto           # TableService: CreateTable, UpdateTable, GetAvailableSlots
├── reservation/
│   └── reservation.proto     # ReservationService: Create, Confirm, Reject, Cancel, List
├── waiting/
│   └── waiting.proto         # WaitingService: Enqueue, Dequeue, CallNext, Cancel, GetStatus
├── menu/
│   └── menu.proto            # MenuService: CreateMenu, UpdateMenu, ListMenus, PreOrder
└── notification/
    └── notification.proto    # NotificationService: SendReservationAlert, SendWaitingAlert
```

---

## Part 6. Project Directory Structure

```
restaurant-reservation/
├── frontend/                       # Next.js app
│   ├── src/
│   │   ├── app/                    # App Router pages
│   │   │   ├── (customer)/         # Customer views
│   │   │   ├── (owner)/            # Restaurant owner dashboard
│   │   │   └── (admin)/            # Super admin
│   │   ├── components/             # UI components (Figma Make generated)
│   │   └── generated/              # protobuf-es auto-generated code
│   └── package.json
├── backend/                        # Go gRPC services
│   ├── cmd/
│   │   └── server/main.go
│   ├── internal/
│   │   ├── user/
│   │   ├── restaurant/
│   │   ├── reservation/            # Includes distributed lock logic
│   │   ├── waiting/                # Redis sequence queue logic
│   │   ├── menu/
│   │   └── notification/
│   ├── pkg/
│   │   ├── redis/                  # Distributed lock, queue utilities
│   │   ├── mysql/                  # DB connection pool
│   │   └── middleware/             # JWT auth interceptor
│   └── go.mod
├── proto/                          # Protocol Buffer definitions
├── mcp-server/                     # MCP server (development convenience)
│   └── main.go                     # gRPC → MCP tool bridge
├── infra/                          # Terraform IaC
│   ├── modules/
│   │   ├── vpc/
│   │   ├── k8s/
│   │   └── database/
│   └── environments/
│       ├── dev/
│       └── prod/
├── docker-compose.yml              # Local development environment
└── .github/
    └── workflows/
        ├── ci.yml                  # Build + test
        └── cd.yml                  # Deploy
```

---

## Part 7. Development Phases

| Phase | Content | Key Deliverables | Completion Criteria |
|-------|---------|-----------------|---------------------|
| **P0** | Project initialization | Repo structure, docker-compose, full proto schema definitions | Proto compilation succeeds |
| **P1** | Backend gRPC services | User / Restaurant / Reservation / Waiting / Menu services | gRPC unit tests pass |
| **P2** | Concurrency control | Redis distributed lock, sequence issuance, Connection Queue | Concurrency tests pass |
| **P3** | MCP server | gRPC → MCP bridge, Claude Code integration | Successful tool invocation from Claude |
| **P4** | UI design | Core screen design with Figma Make | 5 key screens completed |
| **P5** | Frontend | Next.js development with MCP assistance | E2E tests pass |
| **P6** | Infrastructure | Terraform, WAF, CI/CD | Production deployment successful |

---

## Part 8. Verification Methods

| Item | Verification Method |
|------|---------------------|
| Local execution | `docker-compose up` → verify gRPC + MySQL + Redis startup |
| Proto validation | `buf lint` + `buf generate` schema validation |
| Unit tests | `go test ./...` — business logic per service |
| Concurrency test | `k6` script with 100 concurrent reservation requests → verify exactly 1 succeeds |
| Connection Queue | `hey -c 2000` load test → verify queue waiting response |
| MCP operation | `claude mcp add` then invoke gRPC API directly from Claude |
| E2E tests | Playwright — full flow: reservation request → approval → cancellation |
| Security | OWASP ZAP scan, JWT expiry and tampering scenario tests |

---

## Part 9. Open Issues

| # | Item | Decision Required By |
|---|------|---------------------|
| 1 | Payment PG selection (Toss Payments vs Iamport) | Before P1 kickoff |
| 2 | Notification channel priority (Kakao AlimTalk / SMS / Email) | Before P1 kickoff |
| 3 | Final cloud platform decision (AWS vs GCP) | Before P6 kickoff |
| 4 | Whether a dedicated mobile app for restaurant owners is needed | Before P4 kickoff |
| 5 | Whether multi-region deployment is needed | Before P6 kickoff |
| 6 | No-show penalty policy (threshold count, restriction period) | Before P1 kickoff |
| 7 | Cancellation fee policy (fee rate for cancellations within N hours of visit) | Before P1 kickoff |
