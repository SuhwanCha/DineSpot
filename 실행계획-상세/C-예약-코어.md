# C 담당 문서 — 예약 코어 (요구사항 분석 + 구현 명세)

## 1) 역할 목표
- 동일 슬롯/좌석 중복 예약을 원천 차단한다.
- 생성/변경/취소 전이가 상태 머신으로 강제된다.

## 2) 기능 요구사항 (FR)
- FR-C-01: 예약 생성
- FR-C-02: 예약 조회/목록
- FR-C-03: 예약 변경(시간/인원)
- FR-C-04: 예약 취소/노쇼 처리
- FR-C-05: 슬롯 가용성 조회
- FR-C-06: 멱등키 기반 재시도 안전성

## 3) API/함수 구현 명세

| ID | 함수(예시) | Input | Output | 실패 조건 |
|---|---|---|---|---|
| C-FN-01 | `CreateReservation(ctx, req)` | `user_id, restaurant_id, datetime, party_size, idempotency_key` | `reservation_id, status=CONFIRMED/PENDING` | 슬롯 충돌(`CONFLICT`) |
| C-FN-02 | `GetAvailability(ctx, req)` | `restaurant_id, date, party_size` | `time_slots[]` | 참조데이터 없음(`FAILED_PRECONDITION`) |
| C-FN-03 | `UpdateReservation(ctx, req)` | `reservation_id, new_datetime, new_party_size` | `status, updated_at` | 상태 전이 불가(`INVALID_STATE`) |
| C-FN-04 | `CancelReservation(ctx, req)` | `reservation_id, reason` | `status=CANCELED` | 취소 불가 시점(`FAILED_PRECONDITION`) |
| C-FN-05 | `ApplyNoShow(ctx, req)` | `reservation_id, operator_id` | `status=NO_SHOW` | 권한 없음(`PERMISSION_DENIED`) |

## 4) 상태 머신 요구사항
- 허용 전이: `PENDING -> CONFIRMED -> COMPLETED`
- 취소 전이: `PENDING/CONFIRMED -> CANCELED`
- 예외 전이: `CONFIRMED -> NO_SHOW`
- 금지 전이 호출 시 `INVALID_STATE` 반환

## 5) 주차별 상세 작업 + 완료 조건
- W1: 상태머신/락/트랜잭션 골격
  - 완료 조건: 동시 요청 2건 중 1건만 성공 테스트 통과
- W2: Create/Get API
  - 완료 조건: 멱등키 동일 재요청 시 동일 reservation_id 반환
- W3: 슬롯 계산 고도화
  - 완료 조건: B 운영시간/휴무 반영 정확도 100%
- W4: Update/Cancel + FE 연동
  - 완료 조건: 상태 전이 예외 테스트 15건 통과
- W5: D 이벤트 연동
  - 완료 조건: 취소 이벤트 수신 시 웨이팅 승격 이벤트 발행
- W6: 복구/재시도
  - 완료 조건: DB deadlock 재시도 로직 적용
- W7~W9: 부하/회귀/UAT
  - 완료 조건: p99 200ms, 중복예약 0건
- W10: 운영 지표 튜닝
  - 완료 조건: 충돌차단율/실패사유 대시보드 노출

## 6) 교차 점검 항목
- C↔B: 휴무일에는 `CreateReservation()`이 `FAILED_PRECONDITION` 반환.
- C↔A: user role이 customer 아닐 때 고객 예약 생성 차단.
- C↔D: 취소 이벤트(`reservation.canceled.v1`) payload schema 완전 일치.

## 7) 정량 목표
- 중복 예약 0건
- 예약 API p99 < 200ms
- 멱등성 위반 0건
