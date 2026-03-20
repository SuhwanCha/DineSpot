# 식당 예약 시스템 — Product Requirements Document (PRD)

> 버전: v1.0
> 작성일: 2026-03-20
> 분류: 숙박/여행 → 예약 도메인

---

## 적합성 체크리스트

> 판정 기준: 필수 3개 모두 충족 + 권장 1개 이상 → **적합**

| 번호 | 기준 | 구분 | 판정 |
|------|------|------|------|
| 1 | Client/Server 분리 이유가 Problem Description에 존재하는가? | ★ 필수 | ✅ |
| 2 | Actor가 2개 이상 명확히 구분되는가? | ★ 필수 | ✅ |
| 3 | 정상 흐름 외 대안 흐름이 2개 이상 암시되는가? | ★ 필수 | ✅ |
| 4 | 여러 사용자가 동시에 같은 자원에 접근하는 상황이 있는가? | ◎ 권장 | ✅ |
| 5 | 핵심 제어 흐름이 상태에 따라 달라지는가? | ◎ 권장 | ✅ |

**판정: 적합 (필수 3개 + 권장 2개 모두 충족)**

---

## Part 1. Problem Description

### ① 시스템 개요

다수의 식당과 고객을 연결하는 **분산형 멀티테넌트 식당 예약 플랫폼**으로, 고객 단말(웹/앱)과 식당 관리 단말이 중앙 서버에 접속하여 테이블 예약, 실시간 웨이팅, 메뉴 사전 주문을 처리한다.
고객은 어느 단말에서든 실시간 잔여 테이블 현황을 조회하고 예약할 수 있으며, 식당 오너는 별도 관리 단말(웹 대시보드 또는 POS 연동)을 통해 예약을 승인·관리한다. 모든 예약 데이터는 중앙 서버에서 일관성 있게 관리되며, 동시 접근 충돌은 서버 측 임계 영역에서 제어된다.

---

### ② 하드웨어 구성

| 단말 유형 | 입출력 장치 | 역할 |
|-----------|------------|------|
| **고객 단말** (스마트폰 / PC 웹) | 터치스크린 / 키보드 + 마우스, 디스플레이 | 식당 검색, 예약·웨이팅 신청, 메뉴 주문 |
| **식당 관리 단말** (PC 웹 대시보드) | 키보드 + 마우스, 디스플레이 | 예약 승인·거절, 웨이팅 큐 관리, 메뉴 설정 |
| **식당 키오스크** (선택) | 터치스크린, 영수증 프린터, QR 스캐너 | 현장 웨이팅 등록, 예약 확인 |
| **중앙 서버** (Cloud) | 네트워크 인터페이스 | gRPC API 처리, DB 관리, 큐 제어 |
| **알림 게이트웨이** | 네트워크 인터페이스 | SMS / 카카오 알림톡 / 이메일 발송 |

---

### ③ Actor 및 역할

| Actor | 유형 | 시스템과의 상호작용 |
|-------|------|-------------------|
| **고객 (Customer)** | 사람 | 식당 검색·조회, 테이블 예약 신청, 웨이팅 등록, 메뉴 사전 주문, 예약 취소 |
| **식당 오너 (Restaurant Owner)** | 사람 | 식당 정보·테이블·메뉴 관리, 예약 승인/거절, 웨이팅 입장 처리, 매출 조회 |
| **슈퍼관리자 (Super Admin)** | 사람 | 식당 등록 심사·승인, 플랫폼 전체 모니터링, 정책 설정 |
| **알림 게이트웨이 (Notification Gateway)** | 외부 시스템 | 예약 확정/취소/웨이팅 순번 알림을 SMS·카카오 알림톡·이메일로 발송 |
| **결제 시스템 (Payment System)** | 외부 시스템 | 메뉴 사전 주문 결제 승인·취소·환불 처리 |

---

### ④ 핵심 기능 (Use Case 후보)

#### UC-01. 식당 검색 및 조회
- **Actor**: 고객
- **정상 처리 조건**: 검색어(지역, 음식 종류, 날짜)에 해당하는 식당이 1개 이상 존재하고, 해당 식당이 운영 중 상태인 경우 결과 반환

#### UC-02. 테이블 예약 신청
- **Actor**: 고객
- **정상 처리 조건**: 요청 날짜·시간에 해당 식당의 예약 가능 테이블이 존재하고, 고객 계정이 정상(비활성화·정지 아님)이며, 동일 시간대 중복 예약이 없는 경우 예약 신청 접수

#### UC-03. 예약 승인/거절
- **Actor**: 식당 오너
- **정상 처리 조건**: 예약 상태가 PENDING이고, 해당 시간대 테이블 여유가 있을 경우 CONFIRMED로 전이

#### UC-04. 웨이팅 등록
- **Actor**: 고객, 식당 키오스크
- **정상 처리 조건**: 해당 식당이 현재 영업 중이고, 웨이팅 큐가 최대 인원 미만인 경우 큐에 등록 후 순번 발급

#### UC-05. 웨이팅 입장 처리
- **Actor**: 식당 오너
- **정상 처리 조건**: 큐의 맨 앞 고객 상태가 WAITING이고, 고객이 알림을 수신(READY 상태)한 경우 SEATED로 전이

#### UC-06. 메뉴 사전 주문
- **Actor**: 고객
- **정상 처리 조건**: 예약 상태가 CONFIRMED이고, 식당이 사전 주문을 허용하며, 선택 메뉴가 품절이 아닌 경우 주문 접수 및 결제 진행

#### UC-07. 예약 취소
- **Actor**: 고객, 식당 오너
- **정상 처리 조건**: 예약 상태가 PENDING 또는 CONFIRMED이고, 취소 가능 시간(방문 N시간 전) 이내인 경우 취소 처리 및 환불

#### UC-08. 식당 등록 및 승인
- **Actor**: 식당 오너(신청), 슈퍼관리자(심사)
- **정상 처리 조건**: 제출 서류가 완비되어 있고, 중복 사업자번호가 없는 경우 심사 후 ACTIVE 전이

#### UC-09. 매출·예약 현황 대시보드 조회
- **Actor**: 식당 오너, 슈퍼관리자
- **정상 처리 조건**: 인증된 오너 계정이며, 요청 기간 내 데이터가 존재하는 경우 집계 데이터 반환

---

### ⑤ 대안 흐름 조건

| 대안 흐름 ID | 트리거 조건 | 시스템 처리 |
|-------------|-----------|-----------|
| **AF-01** 만석 | 예약 요청 시 해당 날짜·시간에 가용 테이블 없음 | 예약 거부, 웨이팅 등록 안내 반환 |
| **AF-02** 중복 예약 | 동일 고객이 같은 날짜·시간대에 이미 예약을 보유 | 예약 거부, 기존 예약 정보 반환 |
| **AF-03** 취소 기한 초과 | 방문 시간 1시간 이내에 취소 요청 | 취소 거부 또는 취소 수수료 적용 |
| **AF-04** 결제 실패 | 사전 주문 결제 시 카드 한도 초과 또는 승인 거절 | 주문 대기 상태 유지, 재결제 유도 |
| **AF-05** 웨이팅 이탈 | 입장 가능 알림 후 N분 내 응답 없음 | READY → NO_SHOW 전이, 다음 순번 호출 |
| **AF-06** 웨이팅 큐 초과 | 웨이팅 등록 시 최대 대기 인원 초과 | 등록 거부, 예상 대기 시간 안내 |
| **AF-07** 메뉴 품절 | 사전 주문 시 선택 메뉴 재고 없음 | 해당 메뉴 선택 불가 처리, 대안 메뉴 안내 |
| **AF-08** 식당 오너 거절 | 오너가 예약 요청을 수동 거절 | PENDING → REJECTED 전이, 고객 알림 발송 |
| **AF-09** 노쇼 (No-Show) | 방문 예약 시간 초과 후 미방문 확인 | CONFIRMED → NO_SHOW 전이, 패널티 부여 |
| **AF-10** 비인가 접근 | 인증 토큰 없거나 만료된 상태로 API 호출 | 401 Unauthorized 반환, 재로그인 유도 |

---

### ⑥ 동시성 상황

다수의 고객 단말이 **동일 식당의 동일 날짜·시간대 테이블**에 동시에 예약을 시도할 수 있다. 예를 들어, 인기 식당의 저녁 7시 좌석에 수십 명이 동시에 예약 버튼을 누르는 경우 테이블 중복 배정이 발생할 수 있다.

**임계 영역(Critical Section) 설계:**

| 자원 | 동시 접근 시나리오 | 제어 방법 |
|------|----------------|----------|
| 테이블 예약 슬롯 | 여러 고객이 동시에 같은 시간대 테이블 예약 시도 | Redis 분산 락 (SETNX) + MySQL 트랜잭션 SELECT FOR UPDATE |
| 웨이팅 큐 순번 | 여러 고객이 동시에 웨이팅 등록 시도 | Redis INCR 원자 연산으로 순번 발급 |
| 메뉴 재고 | 여러 예약 건에서 동시에 같은 메뉴 사전 주문 | MySQL 재고 필드 원자적 감소 (UPDATE ... WHERE stock > 0) |
| Connection Queue | 트래픽 급증 시 동시 접속 요청 | Redis 기반 큐 + WAF Connection 상한선 제어 |

---

### ⑦ 데이터 관리 위치

모든 비즈니스 데이터는 클라이언트에 저장되지 않고 **중앙 서버에서 단일 진실 공급원(Single Source of Truth)으로 관리**된다.

| 데이터 유형 | 저장 위치 | 비고 |
|------------|---------|------|
| 회원 정보 (고객, 오너) | MySQL — `users` 테이블 | 비밀번호 bcrypt 해시 |
| 식당 정보 (메뉴, 운영시간, 테이블) | MySQL — `restaurants`, `tables`, `menus` | 오너별 격리 |
| 예약 데이터 | MySQL — `reservations` | 상태 전이 기록 포함 |
| 웨이팅 큐 | Redis Sorted Set + MySQL (영속화) | 실시간 순번 Redis, 이력 MySQL |
| 세션 / 인증 토큰 | Redis (TTL 설정) | JWT 블랙리스트 포함 |
| 분산 락 | Redis (SETNX + TTL) | 예약 슬롯 충돌 방지 |
| 이미지 파일 (식당, 메뉴) | AWS S3 / GCS | URL만 DB 저장 |
| 결제 이력 | MySQL — `payments` | PG사 트랜잭션 ID 보관 |

---

## Part 2. 상세 요구사항 명세

### FR (Functional Requirements) — 기능 요구사항

#### FR-USER: 사용자 관리

| ID | 요구사항 | 우선순위 |
|----|---------|---------|
| FR-USER-01 | 고객은 이메일+비밀번호 또는 소셜 로그인(카카오, 구글)으로 회원가입할 수 있다 | P0 |
| FR-USER-02 | 식당 오너는 사업자 정보를 포함한 별도 회원가입 절차를 거쳐야 한다 | P0 |
| FR-USER-03 | 로그인 성공 시 JWT Access Token(15분)과 Refresh Token(7일)을 발급한다 | P0 |
| FR-USER-04 | 고객은 프로필(이름, 전화번호, 알림 수신 여부)을 수정할 수 있다 | P1 |
| FR-USER-05 | 노쇼 3회 누적 시 고객 계정을 30일간 예약 제한 상태로 전환한다 | P1 |

#### FR-RESTAURANT: 식당 관리

| ID | 요구사항 | 우선순위 |
|----|---------|---------|
| FR-REST-01 | 식당 오너는 식당명, 주소, 카테고리, 운영시간, 대표 이미지를 등록할 수 있다 | P0 |
| FR-REST-02 | 등록된 식당은 슈퍼관리자의 승인(PENDING → ACTIVE) 후에만 고객에게 노출된다 | P0 |
| FR-REST-03 | 오너는 테이블을 추가하고 각 테이블의 최소/최대 수용 인원과 예약 가능 시간 슬롯을 설정할 수 있다 | P0 |
| FR-REST-04 | 오너는 특정 날짜를 휴무일로 설정할 수 있으며, 해당 날짜에는 예약이 불가하다 | P1 |
| FR-REST-05 | 고객은 식당명, 지역, 음식 카테고리, 날짜+인원 조건으로 식당을 검색할 수 있다 | P0 |
| FR-REST-06 | 식당 상세 페이지에서 실시간 잔여 테이블 수를 조회할 수 있다 | P0 |

#### FR-RESERVATION: 예약 관리

| ID | 요구사항 | 우선순위 |
|----|---------|---------|
| FR-RES-01 | 고객은 식당 상세 페이지에서 날짜, 시간, 인원을 선택하여 예약을 신청한다 | P0 |
| FR-RES-02 | 예약 신청 시 해당 슬롯을 Redis 분산 락으로 선점하고, 30초 내 최종 확정되지 않으면 락을 해제한다 | P0 |
| FR-RES-03 | 오너가 예약을 승인하면 고객에게 확정 알림(카카오 알림톡 또는 SMS)을 발송한다 | P0 |
| FR-RES-04 | 오너가 예약을 거절하면 거절 사유와 함께 고객 알림을 발송한다 | P0 |
| FR-RES-05 | 고객은 방문 1시간 전까지 예약을 취소할 수 있으며, 사전 결제된 금액은 환불된다 | P0 |
| FR-RES-06 | 방문 시간 30분 전에 고객에게 방문 알림을 자동 발송한다 | P1 |
| FR-RES-07 | 방문 시간 초과 후 1시간 내 미방문이 확인되면 NO_SHOW로 처리하고 패널티를 적용한다 | P1 |
| FR-RES-08 | 고객은 자신의 예약 내역(예정/완료/취소)을 목록으로 조회할 수 있다 | P0 |

#### FR-WAITING: 웨이팅 관리

| ID | 요구사항 | 우선순위 |
|----|---------|---------|
| FR-WAIT-01 | 고객은 해당 식당이 영업 중이고 웨이팅이 활성화된 경우 웨이팅을 등록할 수 있다 | P0 |
| FR-WAIT-02 | 웨이팅 등록 시 Redis INCR로 원자적으로 순번을 발급하고, 앞 대기 팀 수와 예상 대기 시간을 반환한다 | P0 |
| FR-WAIT-03 | 오너가 입장 처리를 하면 해당 고객에게 입장 가능 알림을 발송하고 상태를 READY로 변경한다 | P0 |
| FR-WAIT-04 | 알림 발송 후 10분 내 고객이 응답하지 않으면 NO_SHOW로 처리하고 다음 순번을 호출한다 | P0 |
| FR-WAIT-05 | 고객은 웨이팅을 취소할 수 있으며, 취소 시 뒤 순번들의 예상 대기 시간이 갱신된다 | P0 |
| FR-WAIT-06 | 웨이팅 상태는 SSE(Server-Sent Events)로 실시간 업데이트된다 | P1 |
| FR-WAIT-07 | 식당 키오스크에서도 QR 코드 스캔을 통해 웨이팅을 등록할 수 있다 | P2 |

#### FR-MENU: 메뉴 및 사전 주문

| ID | 요구사항 | 우선순위 |
|----|---------|---------|
| FR-MENU-01 | 오너는 메뉴 카테고리, 메뉴명, 가격, 설명, 이미지, 재고 여부를 등록할 수 있다 | P0 |
| FR-MENU-02 | 고객은 예약 확정 후 방문 전까지 메뉴를 사전 주문할 수 있다 | P1 |
| FR-MENU-03 | 사전 주문 시 결제가 완료되어야 주문이 확정된다 | P1 |
| FR-MENU-04 | 결제 실패 시 주문은 PAYMENT_PENDING 상태로 유지되고, 재결제 링크를 발송한다 | P1 |
| FR-MENU-05 | 품절 메뉴는 주문 선택 UI에서 비활성화 표시된다 | P1 |

#### FR-ADMIN: 관리자

| ID | 요구사항 | 우선순위 |
|----|---------|---------|
| FR-ADMIN-01 | 슈퍼관리자는 식당 등록 신청 목록을 조회하고 승인/반려 처리할 수 있다 | P0 |
| FR-ADMIN-02 | 슈퍼관리자는 전체 예약 현황, 식당별 매출, 노쇼 통계를 조회할 수 있다 | P1 |
| FR-ADMIN-03 | 슈퍼관리자는 이상 트래픽 발생 시 특정 IP 또는 계정을 차단할 수 있다 | P1 |

---

### NFR (Non-Functional Requirements) — 비기능 요구사항

#### NFR-PERF: 성능

| ID | 요구사항 |
|----|---------|
| NFR-PERF-01 | 예약 생성 API 응답시간: p99 < 200ms (정상 부하 기준) |
| NFR-PERF-02 | 식당 검색 API 응답시간: p99 < 300ms |
| NFR-PERF-03 | 웨이팅 순번 발급: p99 < 50ms (Redis 원자 연산) |
| NFR-PERF-04 | 동시 접속 처리 목표: 1,000 concurrent connections |

#### NFR-CONCURRENCY: 동시성 및 정합성

| ID | 요구사항 |
|----|---------|
| NFR-CONC-01 | 동일 예약 슬롯에 대한 동시 요청 시 단 1건만 성공해야 한다 (중복 예약 0%) |
| NFR-CONC-02 | 웨이팅 순번 중복 발급이 발생해서는 안 된다 |
| NFR-CONC-03 | Connection Queue 초과 시 클라이언트에게 대기 상태를 실시간으로 전달해야 한다 |
| NFR-CONC-04 | 분산 락 TTL은 30초로 설정하며, 서버 장애 시 자동 해제되어야 한다 |

#### NFR-AVAIL: 가용성 및 복구

| ID | 요구사항 |
|----|---------|
| NFR-AVAIL-01 | 서비스 가용성: 99.9% 이상 (월간 다운타임 < 44분) |
| NFR-AVAIL-02 | 예약 데이터는 MySQL 복제본(Read Replica) 1개 이상 유지 |
| NFR-AVAIL-03 | Redis 장애 시 예약 처리가 MySQL 단독으로 fallback 가능해야 한다 |

#### NFR-SEC: 보안

| ID | 요구사항 |
|----|---------|
| NFR-SEC-01 | 모든 API는 HTTPS (TLS 1.2 이상) 및 gRPC TLS로 암호화 통신 |
| NFR-SEC-02 | JWT Access Token 만료: 15분 / Refresh Token 만료: 7일 |
| NFR-SEC-03 | 식당 오너는 자신의 식당 데이터에만 접근 가능 (멀티테넌트 격리) |
| NFR-SEC-04 | 비밀번호는 bcrypt(cost=12) 이상으로 해시 저장 |
| NFR-SEC-05 | API 요청에 CSRF 토큰 검증 적용 (웹 클라이언트) |

#### NFR-SCALE: 확장성

| ID | 요구사항 |
|----|---------|
| NFR-SCALE-01 | gRPC 서비스는 Kubernetes HPA로 트래픽에 따라 수평 확장 가능해야 한다 |
| NFR-SCALE-02 | 단일 gRPC 서비스 장애가 다른 서비스에 영향을 주지 않아야 한다 |

---

## Part 3. 상태 전이 명세 (State Transition)

### 예약 (Reservation) 상태

```
                    오너 승인
  [PENDING] ──────────────────────▶ [CONFIRMED]
      │                                  │
      │ 오너 거절 / 고객 취소              │ 고객 취소 (1시간 전까지)
      ▼                                  ▼
  [REJECTED]                        [CANCELLED]

  [CONFIRMED] ───── 방문 완료 ────▶ [COMPLETED]
      │
      └──── 방문 시간 초과 (1시간) ──▶ [NO_SHOW]
```

| 상태 | 설명 |
|------|------|
| PENDING | 고객이 예약 신청, 오너 확인 대기 |
| CONFIRMED | 오너가 예약 승인 완료 |
| REJECTED | 오너가 예약 거절 |
| CANCELLED | 고객 또는 오너가 취소 |
| COMPLETED | 방문 완료 처리 |
| NO_SHOW | 방문 시간 초과 후 미방문 |

### 웨이팅 (Waiting) 상태

```
  [WAITING] ──── 오너 호출 ────▶ [READY]
      │                              │
      │ 고객 취소                     │ 10분 무응답
      ▼                              ▼
  [CANCELLED]                   [NO_SHOW]

  [READY] ──── 고객 입장 ────▶ [SEATED]
```

| 상태 | 설명 |
|------|------|
| WAITING | 대기 큐에 등록된 상태 |
| READY | 오너가 입장 가능 알림 발송 |
| SEATED | 고객 입장 완료 |
| NO_SHOW | 알림 후 10분 미응답 |
| CANCELLED | 고객이 직접 취소 |

### 식당 (Restaurant) 등록 상태

```
  [PENDING_REVIEW] ── 승인 ──▶ [ACTIVE]
          │                        │
          │ 반려                    │ 오너 비활성화 요청
          ▼                        ▼
      [REJECTED]               [INACTIVE]
```

---

## Part 4. 기술 아키텍처

### 시스템 구성도

```
고객 단말 (Browser/Mobile)       식당 관리 단말 (Browser)
         │                                │
         └──────────────┬─────────────────┘
                        │ HTTPS
                        ▼
              CDN (CloudFront / Cloud CDN)
                        │
                        ▼
              WAF (AWS WAF / Cloud Armor)
                        │
              ┌─────────────────────┐
              │  Connection Queue   │  ← Redis 기반 동시 접속 상한 제어
              │  (API Gateway)      │
              └─────────────────────┘
                        │
         ┌──────────────┼──────────────┐
         ▼                             ▼
    Next.js (SSR)              Envoy Proxy
    정적 에셋 서빙              gRPC-Web → gRPC 변환
                                       │
              ┌────────────────────────┼──────────────────────┐
              ▼                        ▼                       ▼
       UserService            ReservationService        WaitingService
       (Go / gRPC)             (Go / gRPC)               (Go / gRPC)
              ▼                        ▼                       ▼
       RestaurantService         MenuService            NotificationService
       (Go / gRPC)               (Go / gRPC)             (Go / gRPC)
              │                        │                       │
              └────────────────────────┼───────────────────────┘
                                       ▼
                              ┌────────────────┐
                              │  MySQL 8.x     │  (Primary + Read Replica)
                              │  Redis 7.x     │  (캐시 / 락 / 큐)
                              │  AWS S3 / GCS  │  (이미지)
                              └────────────────┘
                                       │
                              외부 시스템 연동
                              ├── 결제 PG (토스페이먼츠 등)
                              └── 알림 (카카오 알림톡 / SMS)
```

### 기술 스택 요약

| 레이어 | 기술 | 선택 근거 |
|--------|------|----------|
| Frontend | Next.js (App Router) + TypeScript + Tailwind CSS | SSR/SEO, gRPC-Web 통합 |
| UI 디자인 | Figma Make | AI-driven UI 생성 |
| Backend | Go + gRPC (protobuf) | 고성능, 낮은 레이턴시, 타입 안전 |
| API Gateway | Envoy Proxy | gRPC-Web 변환, 로드밸런싱 |
| 주 DB | MySQL 8.x | ACID 트랜잭션, 예약 정합성 보장 |
| 캐시/큐/락 | Redis 7.x | 분산 락, 순번 발급, Connection Queue |
| 인프라 | Docker / K8s / Terraform | 컨테이너 오케스트레이션, IaC |
| CI/CD | GitHub Actions | 자동화 빌드·테스트·배포 |
| MCP Server | Go (gRPC → MCP 브릿지) | Claude Code 프론트 개발 편의 |
| 모니터링 | Prometheus + Grafana | 실시간 메트릭, 알림 |

---

## Part 5. gRPC Proto 스키마 구조

```
proto/
├── common/
│   └── types.proto           # Pagination, Error, Timestamp 공통 타입
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

## Part 6. 프로젝트 디렉토리 구조

```
restaurant-reservation/
├── frontend/                       # Next.js 앱
│   ├── src/
│   │   ├── app/                    # App Router 페이지
│   │   │   ├── (customer)/         # 고객 화면
│   │   │   ├── (owner)/            # 식당 오너 대시보드
│   │   │   └── (admin)/            # 슈퍼관리자
│   │   ├── components/             # UI 컴포넌트 (Figma Make 생성)
│   │   └── generated/              # protobuf-es 자동 생성 코드
│   └── package.json
├── backend/                        # Go gRPC 서비스
│   ├── cmd/
│   │   └── server/main.go
│   ├── internal/
│   │   ├── user/
│   │   ├── restaurant/
│   │   ├── reservation/            # 분산 락 로직 포함
│   │   ├── waiting/                # Redis 순번 큐 로직
│   │   ├── menu/
│   │   └── notification/
│   ├── pkg/
│   │   ├── redis/                  # 분산 락, 큐 유틸
│   │   ├── mysql/                  # DB 연결 풀
│   │   └── middleware/             # JWT 인증 interceptor
│   └── go.mod
├── proto/                          # Protocol Buffer 정의
├── mcp-server/                     # MCP 서버 (개발 편의용)
│   └── main.go                     # gRPC → MCP tool 브릿지
├── infra/                          # Terraform IaC
│   ├── modules/
│   │   ├── vpc/
│   │   ├── k8s/
│   │   └── database/
│   └── environments/
│       ├── dev/
│       └── prod/
├── docker-compose.yml              # 로컬 개발 환경
└── .github/
    └── workflows/
        ├── ci.yml                  # 빌드 + 테스트
        └── cd.yml                  # 배포
```

---

## Part 7. 개발 단계 (Phase)

| Phase | 내용 | 주요 산출물 | 기준 |
|-------|------|-----------|------|
| **P0** | 프로젝트 초기화 | repo 구조, docker-compose, proto 스키마 전체 정의 | proto 컴파일 성공 |
| **P1** | 백엔드 gRPC 서비스 | User / Restaurant / Reservation / Waiting / Menu 서비스 | gRPC 단위 테스트 통과 |
| **P2** | 동시성 제어 | Redis 분산 락, 순번 발급, Connection Queue | 동시성 테스트 통과 |
| **P3** | MCP 서버 | gRPC → MCP 브릿지, Claude Code 연동 | Claude에서 도구 호출 성공 |
| **P4** | UI 디자인 | Figma Make로 핵심 화면 설계 | 주요 5개 화면 완성 |
| **P5** | 프론트엔드 | Next.js, MCP 활용 개발 | E2E 테스트 통과 |
| **P6** | 인프라 | Terraform, WAF, CI/CD | 프로덕션 배포 성공 |

---

## Part 8. 검증 방법

| 항목 | 검증 방법 |
|------|---------|
| 로컬 실행 | `docker-compose up` → gRPC + MySQL + Redis 기동 확인 |
| Proto 유효성 | `buf lint` + `buf generate` 스키마 검증 |
| 단위 테스트 | `go test ./...` — 각 서비스 비즈니스 로직 |
| 동시성 테스트 | `k6` 스크립트로 100명 동시 예약 요청 → 단 1건만 성공 확인 |
| Connection Queue | `hey -c 2000` 부하 테스트 → 큐 대기 응답 확인 |
| MCP 동작 | `claude mcp add` 후 Claude에서 gRPC API 직접 호출 |
| E2E 테스트 | Playwright — 예약 신청→승인→취소 전체 플로우 |
| 보안 | OWASP ZAP 스캔, JWT 만료·위변조 시나리오 테스트 |

---

## Part 9. 미결 사항

| # | 항목 | 결정 필요 시점 |
|---|------|--------------|
| 1 | 결제 PG 선택 (토스페이먼츠 vs 아임포트) | P1 착수 전 |
| 2 | 알림 채널 우선순위 (카카오 알림톡 / SMS / 이메일) | P1 착수 전 |
| 3 | 클라우드 플랫폼 최종 결정 (AWS vs GCP) | P6 착수 전 |
| 4 | 식당 오너 전용 모바일 앱 필요 여부 | P4 착수 전 |
| 5 | 멀티 리전 배포 필요 여부 | P6 착수 전 |
| 6 | 노쇼 패널티 정책 (횟수 기준, 제한 기간) | P1 착수 전 |
| 7 | 취소 수수료 정책 (방문 N시간 이내 취소 시 수수료율) | P1 착수 전 |
