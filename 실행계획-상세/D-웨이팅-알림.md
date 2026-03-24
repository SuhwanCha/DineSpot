# D 담당 문서 — 웨이팅/알림 (요구사항 분석 + 구현 명세)

## 1) 역할 목표
- 웨이팅 순번은 언제나 단조 증가/감소 규칙으로 관리되어야 한다.
- 알림은 중복/누락 없이 전달되어야 한다.

## 2) 기능 요구사항 (FR)
- FR-D-01: 웨이팅 등록
- FR-D-02: 순번 조회/호출/취소
- FR-D-03: 실시간 순번 브로드캐스트
- FR-D-04: 알림 발송/재시도/중복방지
- FR-D-05: 예약 취소 연동 자동 승격

## 3) API/함수 구현 명세

| ID | 함수(예시) | Input | Output | 실패 조건 |
|---|---|---|---|---|
| D-FN-01 | `JoinWaiting(ctx, req)` | `restaurant_id, user_id, party_size` | `waiting_id, queue_no, eta_min` | 영업종료(`FAILED_PRECONDITION`) |
| D-FN-02 | `CallNext(ctx, req)` | `restaurant_id, operator_id` | `called_waiting_id, ttl_min` | 큐 비어있음(`NOT_FOUND`) |
| D-FN-03 | `CancelWaiting(ctx, req)` | `waiting_id, reason` | `status=CANCELED` | 권한 없음(`PERMISSION_DENIED`) |
| D-FN-04 | `PublishQueueUpdate(event)` | `restaurant_id, queue_no, status` | `broadcast_count` | 채널 오류(`UNAVAILABLE`) |
| D-FN-05 | `SendNotification(ctx, req)` | `channel, template_id, recipient, dedupe_key` | `message_id, sent_at` | 중복키(`ALREADY_EXISTS`) |

## 4) 이벤트/실시간 계약
- 이벤트명: `waiting.joined.v1`, `waiting.called.v1`, `waiting.canceled.v1`
- payload 필수: `event_id`, `restaurant_id`, `waiting_id`, `queue_no`, `occurred_at`
- 클라이언트는 `event_id` 기준으로 중복 수신 제거

## 5) 주차별 상세 작업 + 완료 조건
- W1: 큐 모델/순번 계산
  - 완료 조건: 1~N 순번 재정렬 테스트 통과
- W2: 등록/호출/취소 API
  - 완료 조건: CallNext 시 queue_no 최솟값 선택 보장
- W3: SSE 브로드캐스트
  - 완료 조건: 연결 1,000개 기준 지연 1초 이내
- W4: 알림 재시도/중복방지
  - 완료 조건: dedupe_key 중복 전송 0건
- W5: C 취소 이벤트 연동
  - 완료 조건: 승격 대상 계산 정확도 100%
- W6: 단절/복구 처리
  - 완료 조건: 재연결 후 최신 상태 동기화 성공
- W7~W9: 성능/회귀/UAT
  - 완료 조건: 순번 정합성 오류 0건
- W10: 운영 고도화
  - 완료 조건: 알림 실패 대시보드 실시간 반영

## 6) 교차 점검 항목
- D↔C: `reservation.canceled.v1` 수신 후 3초 이내 승격 이벤트 발행.
- D↔A: 사용자 opt-out 상태면 마케팅성 알림 발송 금지.
- D↔E: `notification.fail.rate` 메트릭 수집 검증.

## 7) 정량 목표
- 실시간 이벤트 지연 p95 < 1s
- 알림 중복 발송률 < 0.1%
- 웨이팅 순번 정합성 오류 0건
