# E 담당 문서 — 관리자/운영/관측성 (요구사항 분석 + 구현 명세)

## 1) 역할 목표
- 운영자는 서비스 이상을 5분 내 감지하고 조치 시작할 수 있어야 한다.
- 감사로그/대시보드/알람이 제품 운영의 단일 진실 공급원(SSOT) 역할을 해야 한다.

## 2) 기능 요구사항 (FR)
- FR-E-01: 관리자 승인/제재 워크플로우
- FR-E-02: 감사로그 저장/조회
- FR-E-03: KPI 대시보드
- FR-E-04: 알람 규칙/온콜 연동
- FR-E-05: 장애 대응 Runbook

## 3) API/함수 구현 명세

| ID | 함수(예시) | Input | Output | 실패 조건 |
|---|---|---|---|---|
| E-FN-01 | `ApproveRestaurant(ctx, req)` | `admin_id, restaurant_id, decision, reason` | `approval_status, approved_at` | 권한없음(`PERMISSION_DENIED`) |
| E-FN-02 | `WriteAuditLog(ctx, event)` | `actor_id, action, resource, request_id, result` | `log_id` | 필드누락(`INVALID_ARGUMENT`) |
| E-FN-03 | `QueryAuditLogs(ctx, filter)` | `actor_id, date_range, action` | `logs[], next_cursor` | 범위과다(`FAILED_PRECONDITION`) |
| E-FN-04 | `GetKPI(ctx, req)` | `tenant_id, from, to, granularity` | `reservation_rate, cancel_rate, wait_avg` | 데이터없음(`NOT_FOUND`) |
| E-FN-05 | `EvaluateAlerts(metrics)` | `error_rate, latency_p99, queue_depth` | `alerts[]` | 수집실패(`UNAVAILABLE`) |

## 4) 관측성 계약
- 로그 필수 필드: `timestamp`, `level`, `service`, `trace_id`, `request_id`, `tenant_id`, `user_id`
- 메트릭 필수: `api_latency_ms`, `api_error_total`, `queue_depth`, `notification_fail_total`
- 알람 기준(초기값):
  - `api_error_rate > 2% (5m)`
  - `reservation_p99 > 200ms (10m)`
  - `notification_fail_rate > 1% (10m)`

## 5) 주차별 상세 작업 + 완료 조건
- W1: 공통 로그/트레이스 규약 배포
  - 완료 조건: A~D 서비스에서 trace_id 누락률 0%
- W2: 승인/감사로그 API
  - 완료 조건: 모든 관리자 action이 audit log에 남음
- W3: KPI 파이프라인
  - 완료 조건: 일/주 단위 지표 계산 검증 통과
- W4: 대시보드 1차
  - 완료 조건: 핵심 KPI 3종 시각화
- W5: 알람 룰 적용
  - 완료 조건: 테스트 이벤트로 알람 트리거 검증
- W6: 온콜/런북 정비
  - 완료 조건: 장애 시나리오 5개 대응 절차 작성
- W7: GameDay
  - 완료 조건: MTTD 5분, MTTA 10분 이내 달성
- W8~W9: 튜닝/RC
  - 완료 조건: 오탐률 < 5%, 미탐 0건
- W10: 운영 안정화
  - 완료 조건: 주간 운영리포트 자동 생성

## 6) 교차 점검 항목
- E↔A/B/C/D: 공통 로그 필드 미준수 PR merge 금지 규칙 적용.
- E↔C: 예약 실패 사유 코드가 대시보드 분류체계와 일치해야 함.
- E↔D: 알림 실패 이벤트가 1분 이내 집계에 반영돼야 함.

## 7) 정량 목표
- 장애 감지 MTTD ≤ 5분
- 알람 미탐 0건
- 운영 runbook 최신화율 100%
