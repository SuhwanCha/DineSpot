---
© 2026 Suhwan Cha and contributors. All rights reserved.

The ideas, system design, architecture, data structures, prompts, and methodologies described in this document are proprietary intellectual property.

Unauthorized use for academic research, publications, commercial products, or derivative works is strictly prohibited without explicit written permission.
---

# DineSpot — COMET UML

> COMET (Concurrent Object Modeling and design mEThod)
> Date: 2026-03-24

---

## 1. Use Case Diagram

```plantuml
@startuml use-case-diagram
left to right direction
skinparam packageStyle rectangle

actor "고객\n(Customer)" as Customer
actor "식당 오너\n(Restaurant Owner)" as Owner
actor "슈퍼관리자\n(Super Admin)" as Admin
actor "알림 게이트웨이\n(Notification GW)" as NotifGW <<system>>
actor "결제 시스템\n(Payment System)" as Payment <<system>>
actor "식당 키오스크\n(Kiosk)" as Kiosk <<system>>

rectangle DineSpot {
  usecase "UC-01\n식당 검색 및 조회" as UC01
  usecase "UC-02\n테이블 예약 신청" as UC02
  usecase "UC-03\n예약 승인/거절" as UC03
  usecase "UC-04\n웨이팅 등록" as UC04
  usecase "UC-05\n웨이팅 입장 처리" as UC05
  usecase "UC-06\n메뉴 사전 주문" as UC06
  usecase "UC-07\n예약 취소" as UC07
  usecase "UC-08\n식당 등록 및 승인" as UC08
  usecase "UC-09\n매출·예약 대시보드 조회" as UC09
  usecase "UC-10\n알림 발송" as UC10
  usecase "UC-11\n결제 처리" as UC11
  usecase "UC-12\n노쇼 처리" as UC12
}

Customer --> UC01
Customer --> UC02
Customer --> UC04
Customer --> UC06
Customer --> UC07

Owner --> UC03
Owner --> UC05
Owner --> UC07
Owner --> UC08
Owner --> UC09

Admin --> UC08
Admin --> UC09

Kiosk --> UC04

UC02 ..> UC10 : <<include>>
UC03 ..> UC10 : <<include>>
UC05 ..> UC10 : <<include>>
UC06 ..> UC11 : <<include>>
UC12 ..> UC10 : <<include>>

UC07 ..> UC11 : <<extend>>\n환불 처리
UC02 ..> UC12 : <<extend>>\n노쇼 타이머

NotifGW <-- UC10
Payment <-- UC11
@enduml
```

---

## 2. 객체 구조 모델 (Object Structure — COMET Entity/Boundary/Control)

COMET은 각 Use Case의 참여 객체를 **Entity**, **Boundary**, **Control** 로 분류한다.

| 객체 유형 | 역할 | 해당 객체 |
|-----------|------|-----------|
| **Boundary** | Actor ↔ System 인터페이스 | CustomerUI, OwnerDashboardUI, AdminUI, KioskUI, NotificationGateway, PaymentGateway |
| **Control** | 비즈니스 로직 / 흐름 조율 | ReservationCoordinator, WaitingQueueController, RestaurantApprovalController, NoShowMonitor, ConnectionQueueController |
| **Entity** | 영속 데이터 | User, Restaurant, Table, TimeSlot, Reservation, WaitingEntry, Menu, Order, Payment |

```plantuml
@startuml object-structure
skinparam classAttributeIconSize 0

' ── Boundary Objects ──
package "Boundary" #LightBlue {
  class CustomerUI <<boundary>>
  class OwnerDashboardUI <<boundary>>
  class AdminUI <<boundary>>
  class KioskUI <<boundary>>
  class NotificationGateway <<boundary>>
  class PaymentGateway <<boundary>>
}

' ── Control Objects ──
package "Control" #LightYellow {
  class ReservationCoordinator <<control>>
  class WaitingQueueController <<control>>
  class RestaurantApprovalController <<control>>
  class NoShowMonitor <<control>>
  class ConnectionQueueController <<control>>
  class AuthController <<control>>
}

' ── Entity Objects ──
package "Entity" #LightGreen {
  class User <<entity>>
  class Restaurant <<entity>>
  class Table <<entity>>
  class TimeSlot <<entity>>
  class Reservation <<entity>>
  class WaitingEntry <<entity>>
  class Menu <<entity>>
  class Order <<entity>>
  class Payment <<entity>>
}

' Boundary → Control
CustomerUI --> ReservationCoordinator
CustomerUI --> WaitingQueueController
OwnerDashboardUI --> ReservationCoordinator
OwnerDashboardUI --> WaitingQueueController
OwnerDashboardUI --> RestaurantApprovalController
AdminUI --> RestaurantApprovalController
KioskUI --> WaitingQueueController

' Control → Entity
ReservationCoordinator --> Reservation
ReservationCoordinator --> Table
ReservationCoordinator --> TimeSlot
ReservationCoordinator --> NoShowMonitor
WaitingQueueController --> WaitingEntry
WaitingQueueController --> NoShowMonitor
RestaurantApprovalController --> Restaurant
AuthController --> User

' Control → Boundary
ReservationCoordinator --> NotificationGateway
ReservationCoordinator --> PaymentGateway
WaitingQueueController --> NotificationGateway

' Entity Relationships
User "1" -- "0..*" Reservation
User "1" -- "0..*" WaitingEntry
Restaurant "1" -- "1..*" Table
Restaurant "1" -- "0..*" Menu
Table "1" -- "1..*" TimeSlot
Reservation "1" -- "1" TimeSlot
Reservation "1" -- "0..1" Order
Order "1" -- "1" Payment
@enduml
```

---

## 3. 정적 클래스 다이어그램 (Static Class Diagram)

```plantuml
@startuml class-diagram
skinparam classAttributeIconSize 0

enum ReservationStatus {
  PENDING
  CONFIRMED
  REJECTED
  CANCELLED
  COMPLETED
  NO_SHOW
}

enum WaitingStatus {
  WAITING
  READY
  SEATED
  NO_SHOW
  CANCELLED
}

enum RestaurantStatus {
  PENDING_REVIEW
  ACTIVE
  INACTIVE
  REJECTED
}

enum UserRole {
  CUSTOMER
  OWNER
  SUPER_ADMIN
}

class User {
  +id: UUID
  +email: string
  +passwordHash: string
  +name: string
  +phone: string
  +role: UserRole
  +noShowCount: int
  +isRestricted: bool
  +createdAt: datetime
  +login(): JWT
  +updateProfile()
}

class Restaurant {
  +id: UUID
  +ownerId: UUID
  +name: string
  +address: string
  +category: string
  +operatingHours: JSON
  +status: RestaurantStatus
  +imageURL: string
  +approve()
  +deactivate()
}

class Table {
  +id: UUID
  +restaurantId: UUID
  +label: string
  +minCapacity: int
  +maxCapacity: int
  +isActive: bool
}

class TimeSlot {
  +id: UUID
  +tableId: UUID
  +date: date
  +startTime: time
  +isAvailable: bool
  +lock(ttl: int)
  +unlock()
}

class Reservation {
  +id: UUID
  +customerId: UUID
  +restaurantId: UUID
  +tableId: UUID
  +timeSlotId: UUID
  +guestCount: int
  +status: ReservationStatus
  +createdAt: datetime
  +visitAt: datetime
  +confirm()
  +reject(reason: string)
  +cancel()
  +markNoShow()
  +complete()
}

class WaitingEntry {
  +id: UUID
  +restaurantId: UUID
  +customerId: UUID
  +sequenceNo: int
  +guestCount: int
  +status: WaitingStatus
  +registeredAt: datetime
  +notifiedAt: datetime
  +callNext()
  +markSeated()
  +markNoShow()
  +cancel()
}

class Menu {
  +id: UUID
  +restaurantId: UUID
  +category: string
  +name: string
  +price: decimal
  +description: string
  +imageURL: string
  +stock: int
  +isSoldOut: bool
  +decreaseStock(qty: int)
}

class Order {
  +id: UUID
  +reservationId: UUID
  +customerId: UUID
  +items: OrderItem[]
  +totalAmount: decimal
  +status: string
  +placeOrder()
  +cancel()
}

class Payment {
  +id: UUID
  +orderId: UUID
  +pgTransactionId: string
  +amount: decimal
  +status: string
  +paidAt: datetime
  +refund()
}

User "1" --o "0..*" Reservation : places
User "1" --o "0..*" WaitingEntry : registers
Restaurant "1" *-- "1..*" Table
Restaurant "1" *-- "0..*" Menu
Table "1" *-- "1..*" TimeSlot
TimeSlot "1" --o "0..1" Reservation
Reservation "1" --o "0..1" Order
Order "1" -- "1" Payment
Order "1" *-- "1..*" OrderItem

class OrderItem {
  +menuId: UUID
  +quantity: int
  +unitPrice: decimal
}
@enduml
```

---

## 4. 동적 상호작용 다이어그램 (Sequence Diagrams)

### UC-02: 테이블 예약 신청

```plantuml
@startuml seq-reservation
actor Customer
participant CustomerUI <<boundary>>
participant ReservationCoordinator <<control>>
participant ConnectionQueueController <<control>>
participant TimeSlot <<entity>>
participant Reservation <<entity>>
participant NotificationGateway <<boundary>>
participant "Redis\n(분산 락)" as Redis
participant "MySQL" as DB

Customer -> CustomerUI: 날짜/시간/인원 선택 후 예약 신청

CustomerUI -> ConnectionQueueController: 요청 진입
alt 동시 접속 상한 초과
  ConnectionQueueController --> CustomerUI: 대기 상태 반환 (SSE)
  CustomerUI --> Customer: 잠시 후 재시도 안내
else 처리 가능
  ConnectionQueueController --> CustomerUI: 통과
end

CustomerUI -> ReservationCoordinator: CreateReservation(customerId, restaurantId, timeSlotId, guestCount)
ReservationCoordinator -> Redis: SETNX lock:{timeSlotId} TTL=30s

alt 락 획득 실패 (중복 요청)
  Redis --> ReservationCoordinator: FAIL
  ReservationCoordinator --> CustomerUI: 예약 불가 (이미 선점됨)
  CustomerUI --> Customer: 웨이팅 등록 안내
else 락 획득 성공
  Redis --> ReservationCoordinator: OK
  ReservationCoordinator -> TimeSlot: checkAvailability()
  TimeSlot -> DB: SELECT FOR UPDATE
  DB --> TimeSlot: 가용 슬롯 확인

  alt 슬롯 만석
    ReservationCoordinator -> Redis: DEL lock:{timeSlotId}
    ReservationCoordinator --> CustomerUI: AF-01 만석 처리
    CustomerUI --> Customer: 웨이팅 등록 안내
  else 슬롯 가용
    ReservationCoordinator -> Reservation: create(PENDING)
    Reservation -> DB: INSERT reservation
    DB --> Reservation: reservationId
    ReservationCoordinator -> TimeSlot: markUnavailable()
    TimeSlot -> DB: UPDATE isAvailable=false
    ReservationCoordinator -> Redis: DEL lock:{timeSlotId}
    ReservationCoordinator -> NotificationGateway: notify(Owner, 예약 신청 알림)
    ReservationCoordinator --> CustomerUI: 예약 접수 완료 (PENDING)
    CustomerUI --> Customer: 예약 접수 확인 화면
  end
end
@enduml
```

---

### UC-03: 예약 승인/거절

```plantuml
@startuml seq-approval
actor Owner
participant OwnerDashboardUI <<boundary>>
participant ReservationCoordinator <<control>>
participant Reservation <<entity>>
participant NotificationGateway <<boundary>>
participant "MySQL" as DB

Owner -> OwnerDashboardUI: 예약 목록 조회
OwnerDashboardUI -> ReservationCoordinator: ListPendingReservations(restaurantId)
ReservationCoordinator -> DB: SELECT reservations WHERE status=PENDING
DB --> ReservationCoordinator: 예약 목록
ReservationCoordinator --> OwnerDashboardUI: 예약 목록 반환
OwnerDashboardUI --> Owner: 예약 목록 표시

alt 승인
  Owner -> OwnerDashboardUI: 승인 클릭
  OwnerDashboardUI -> ReservationCoordinator: ApproveReservation(reservationId)
  ReservationCoordinator -> Reservation: confirm()
  Reservation -> DB: UPDATE status=CONFIRMED
  ReservationCoordinator -> NotificationGateway: notify(Customer, 예약 확정 알림톡/SMS)
  ReservationCoordinator --> OwnerDashboardUI: 승인 완료
else 거절
  Owner -> OwnerDashboardUI: 거절 + 사유 입력
  OwnerDashboardUI -> ReservationCoordinator: RejectReservation(reservationId, reason)
  ReservationCoordinator -> Reservation: reject(reason)
  Reservation -> DB: UPDATE status=REJECTED
  ReservationCoordinator -> NotificationGateway: notify(Customer, 거절 사유 + 알림)
  ReservationCoordinator --> OwnerDashboardUI: 거절 완료
end
@enduml
```

---

### UC-04 / UC-05: 웨이팅 등록 및 입장 처리

```plantuml
@startuml seq-waiting
actor Customer
actor Owner
participant CustomerUI <<boundary>>
participant OwnerDashboardUI <<boundary>>
participant WaitingQueueController <<control>>
participant WaitingEntry <<entity>>
participant NotificationGateway <<boundary>>
participant "Redis\n(Sorted Set)" as Redis
participant "MySQL" as DB

== 웨이팅 등록 (UC-04) ==
Customer -> CustomerUI: 웨이팅 등록 요청 (인원 선택)
CustomerUI -> WaitingQueueController: RegisterWaiting(customerId, restaurantId, guestCount)
WaitingQueueController -> Redis: INCR waiting:{restaurantId}:seq → sequenceNo
WaitingQueueController -> Redis: ZADD waiting:{restaurantId} score=sequenceNo customerId
WaitingQueueController -> WaitingEntry: create(WAITING, sequenceNo)
WaitingEntry -> DB: INSERT waiting_entries
WaitingQueueController --> CustomerUI: 순번, 앞 대기팀 수, 예상 대기 시간 반환
CustomerUI --> Customer: 웨이팅 등록 완료 (순번 표시)

== 웨이팅 입장 처리 (UC-05) ==
Owner -> OwnerDashboardUI: 입장 처리 클릭 (다음 순번 호출)
OwnerDashboardUI -> WaitingQueueController: CallNext(restaurantId)
WaitingQueueController -> Redis: ZRANGE waiting:{restaurantId} 0 0 → 최상위 customerId
WaitingQueueController -> WaitingEntry: markReady()
WaitingEntry -> DB: UPDATE status=READY, notifiedAt=NOW()
WaitingQueueController -> NotificationGateway: notify(Customer, 입장 가능 알림)
WaitingQueueController --> OwnerDashboardUI: 입장 알림 발송 완료

note over WaitingQueueController: 10분 타이머 시작

alt 고객 입장 확인
  Customer -> CustomerUI: 입장 확인
  CustomerUI -> WaitingQueueController: ConfirmSeated(waitingId)
  WaitingQueueController -> WaitingEntry: markSeated()
  WaitingEntry -> DB: UPDATE status=SEATED
  WaitingQueueController -> Redis: ZREM waiting:{restaurantId} customerId
else 10분 무응답 (AF-05)
  WaitingQueueController -> WaitingEntry: markNoShow()
  WaitingEntry -> DB: UPDATE status=NO_SHOW
  WaitingQueueController -> Redis: ZREM waiting:{restaurantId} customerId
  WaitingQueueController -> NotificationGateway: notify(NextCustomer, 입장 가능 알림)
end
@enduml
```

---

### UC-06: 메뉴 사전 주문 및 결제

```plantuml
@startuml seq-preorder
actor Customer
participant CustomerUI <<boundary>>
participant ReservationCoordinator <<control>>
participant Menu <<entity>>
participant Order <<entity>>
participant Payment <<entity>>
participant PaymentGateway <<boundary>>
participant NotificationGateway <<boundary>>
participant "MySQL" as DB

Customer -> CustomerUI: 메뉴 선택 및 주문 요청
CustomerUI -> ReservationCoordinator: PlacePreOrder(reservationId, items[])

ReservationCoordinator -> Menu: checkAvailability(items[])
Menu -> DB: SELECT stock WHERE menuId IN (items)

alt 품절 메뉴 포함 (AF-07)
  ReservationCoordinator --> CustomerUI: 품절 메뉴 안내, 대안 메뉴 반환
  CustomerUI --> Customer: 품절 표시
else 모두 가용
  ReservationCoordinator -> Order: create(items[], totalAmount)
  Order -> DB: INSERT orders, order_items
  ReservationCoordinator -> Menu: decreaseStock(items[])
  Menu -> DB: UPDATE stock WHERE stock > 0

  ReservationCoordinator -> PaymentGateway: requestPayment(orderId, amount)
  PaymentGateway --> ReservationCoordinator: 결제 결과

  alt 결제 성공
    ReservationCoordinator -> Payment: create(PAID)
    Payment -> DB: INSERT payments
    ReservationCoordinator -> Order: markConfirmed()
    ReservationCoordinator -> NotificationGateway: notify(Customer, 주문 확정 알림)
    ReservationCoordinator --> CustomerUI: 주문 완료
  else 결제 실패 (AF-04)
    ReservationCoordinator -> Order: markPaymentPending()
    Order -> DB: UPDATE status=PAYMENT_PENDING
    ReservationCoordinator -> NotificationGateway: notify(Customer, 재결제 링크 발송)
    ReservationCoordinator --> CustomerUI: 결제 실패, 재결제 안내
  end
end
@enduml
```

---

## 5. 상태 머신 다이어그램 (State Machine Diagrams)

### 예약 (Reservation) 상태 머신

```plantuml
@startuml sm-reservation
[*] --> PENDING : 고객 예약 신청\n[가용 슬롯 존재 & 중복 없음]

PENDING --> CONFIRMED : 오너 승인\n/ 고객 확정 알림 발송
PENDING --> REJECTED : 오너 거절\n/ 거절 사유 알림 발송
PENDING --> CANCELLED : 고객 취소\n/ 분산 락 해제

CONFIRMED --> CANCELLED : 고객 취소\n[방문 1시간 전까지]\n/ 환불 처리
CONFIRMED --> COMPLETED : 방문 완료 처리
CONFIRMED --> NO_SHOW : 방문 시간 +1시간 경과\n/ 패널티 부여

CANCELLED --> [*]
REJECTED --> [*]
COMPLETED --> [*]
NO_SHOW --> [*]

note right of PENDING
  Redis 분산 락 (TTL=30s)
  으로 해당 TimeSlot 선점
end note

note right of NO_SHOW
  NoShowCount++
  3회 누적 시 계정 30일 제한
end note
@enduml
```

---

### 웨이팅 (WaitingEntry) 상태 머신

```plantuml
@startuml sm-waiting
[*] --> WAITING : 고객 웨이팅 등록\n[큐 최대 인원 미만]\n/ Redis INCR 순번 발급

WAITING --> READY : 오너 입장 호출\n/ 입장 가능 알림 발송\n/ 10분 타이머 시작
WAITING --> CANCELLED : 고객 직접 취소\n/ 뒤 순번 예상 시간 갱신

READY --> SEATED : 고객 입장 확인
READY --> NO_SHOW : 10분 무응답\n/ 다음 순번 자동 호출

SEATED --> [*]
NO_SHOW --> [*]
CANCELLED --> [*]

note right of READY
  타이머: 10분 (FR-WAIT-04)
  타임아웃 → NoShowMonitor 트리거
end note
@enduml
```

---

### 식당 등록 (Restaurant) 상태 머신

```plantuml
@startuml sm-restaurant
[*] --> PENDING_REVIEW : 오너 식당 등록 신청\n[사업자 정보 완비]

PENDING_REVIEW --> ACTIVE : 슈퍼관리자 승인\n/ 고객 검색에 노출
PENDING_REVIEW --> REJECTED : 슈퍼관리자 반려\n/ 반려 사유 통보

ACTIVE --> INACTIVE : 오너 비활성화 요청
INACTIVE --> ACTIVE : 오너 재활성화 요청

REJECTED --> PENDING_REVIEW : 정보 보완 후 재신청
INACTIVE --> [*]
REJECTED --> [*]
@enduml
```

---

## 6. 동시성 설계 (Concurrency Design)

### 임계 영역 및 동시성 제어 요약

```plantuml
@startuml concurrency
package "Connection Layer" {
  component [WAF\n(AWS WAF / Cloud Armor)] as WAF
  component [Connection Queue\n(Redis List)] as CQ
  note right of CQ
    동시 처리 상한 초과 시
    SSE로 대기 상태 반환
  end note
}

package "Reservation Critical Section" {
  component [Redis 분산 락\nSETNX lock:{slotId} TTL=30s] as Lock
  component [MySQL\nSELECT FOR UPDATE] as SlotDB
  note right of Lock
    락 획득 실패 → 즉시 거부
    서버 장애 시 TTL 만료 자동 해제
  end note
}

package "Waiting Queue Critical Section" {
  component [Redis INCR\nwaiting:{restaurantId}:seq] as SeqGen
  component [Redis Sorted Set\nwaiting:{restaurantId}] as WaitQ
  note right of SeqGen
    원자 연산으로 순번 중복 0%
  end note
}

package "Menu Stock Critical Section" {
  component [MySQL\nUPDATE stock\nWHERE stock > 0] as StockDB
  note right of StockDB
    원자적 감소로 음수 재고 방지
  end note
}

WAF --> CQ : 요청 제어
CQ --> Lock : 예약 요청 전달
Lock --> SlotDB : 슬롯 검증/선점
@enduml
```

---

## 7. 배포 아키텍처 다이어그램

```plantuml
@startuml deployment
node "Client" {
  component [Browser / Mobile App] as Client
}

node "CDN" {
  component [CloudFront / Cloud CDN] as CDN
}

node "Edge" {
  component [WAF] as WAF
  component [API Gateway\n+ Connection Queue] as APIGW
}

node "Application Cluster (K8s)" {
  component [Next.js\n(SSR)] as NextJS
  component [Envoy Proxy\n(gRPC-Web → gRPC)] as Envoy

  node "gRPC Services" {
    component [UserService\n:50051] as UserSvc
    component [RestaurantService\n:50052] as RestSvc
    component [ReservationService\n:50053] as ResvSvc
    component [WaitingService\n:50054] as WaitSvc
    component [MenuService\n:50055] as MenuSvc
  }

  component [MCP Server\n(dev only)] as MCP
}

node "Data Layer" {
  database "MySQL 8.x\n(Primary + Read Replica)" as MySQL
  database "Redis 7.x\n(Cache / Queue / Lock)" as Redis
  storage "S3 / GCS\n(Images)" as Storage
}

node "External" {
  component [Notification Gateway\n(Kakao / SMS / Email)] as NotifGW
  component [Payment System\n(PG사)] as PaySys
}

Client --> CDN : HTTPS
CDN --> WAF
WAF --> APIGW
APIGW --> NextJS
APIGW --> Envoy : gRPC-Web
Envoy --> UserSvc
Envoy --> RestSvc
Envoy --> ResvSvc
Envoy --> WaitSvc
Envoy --> MenuSvc

UserSvc --> MySQL
RestSvc --> MySQL
ResvSvc --> MySQL
ResvSvc --> Redis
WaitSvc --> Redis
WaitSvc --> MySQL
MenuSvc --> MySQL
MenuSvc --> Storage

ResvSvc --> NotifGW
WaitSvc --> NotifGW
ResvSvc --> PaySys
@enduml
```

---

## 8. COMET 설계 요약

| 설계 단계 | 산출물 | 비고 |
|-----------|--------|------|
| **Use Case Modeling** | UC-01 ~ UC-12 유스케이스 다이어그램 | 9개 기능 + 3개 보조 UC |
| **Object Structuring** | Boundary / Control / Entity 분류 | 6 Boundary, 6 Control, 9 Entity |
| **Static Model** | 클래스 다이어그램 + 상태 열거형 | MySQL 스키마 직접 매핑 |
| **Dynamic Model** | 시퀀스 다이어그램 4종 | UC-02, 03, 04/05, 06 |
| **State Machine** | Reservation / Waiting / Restaurant | 3개 상태 머신 |
| **Concurrency Design** | 임계 영역 식별 및 제어 전략 | Redis 락 + MySQL FOR UPDATE |
| **Deployment** | 배포 아키텍처 | K8s + gRPC 마이크로서비스 |
