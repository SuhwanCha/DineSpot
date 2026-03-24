# B 담당 문서 — 레스토랑/메뉴/운영정책 (요구사항 분석 + 구현 명세)

## 1) 역할 목표
- 예약 엔진(C)이 신뢰 가능한 매장 운영 데이터(좌석/영업시간/휴무)를 조회할 수 있게 한다.
- 점주 UI와 API 데이터가 완전히 동일한 규칙으로 검증되어야 한다.

## 2) 기능 요구사항 (FR)
- FR-B-01: 레스토랑 CRUD
- FR-B-02: 메뉴 CRUD + 품절 상태
- FR-B-03: 운영시간/브레이크타임/휴무일 관리
- FR-B-04: 좌석 정책(인원/테이블 타입) 관리
- FR-B-05: 테넌트 격리

## 3) API/함수 구현 명세

| ID | 함수(예시) | Input | Output | 실패 조건 |
|---|---|---|---|---|
| B-FN-01 | `CreateRestaurant(ctx, req)` | `tenant_id, name, address, timezone` | `restaurant_id` | 중복 매장(`ALREADY_EXISTS`) |
| B-FN-02 | `UpdateOperatingHours(ctx, req)` | `restaurant_id, day_of_week, open, close, break_ranges[]` | `version` | 시간 역전(`VALIDATION_FAILED`) |
| B-FN-03 | `UpsertSeatPolicy(ctx, req)` | `restaurant_id, table_type, min_party, max_party, count` | `policy_id` | 범위 오류(`INVALID_ARGUMENT`) |
| B-FN-04 | `UpsertMenuItem(ctx, req)` | `restaurant_id, name, price, status` | `menu_item_id` | 음수 가격(`VALIDATION_FAILED`) |
| B-FN-05 | `GetReservationReferenceData(ctx, req)` | `restaurant_id, target_date` | `hours, seat_policy, holiday_flag` | 타 테넌트 접근(`PERMISSION_DENIED`) |

## 4) 데이터/스키마 요구사항
- `restaurant` 테이블: `tenant_id` 인덱스 필수
- `menu_item` 테이블: `(restaurant_id, status)` 복합 인덱스
- `seat_policy` 테이블: `(restaurant_id, table_type)` unique

## 5) 주차별 상세 작업 + 완료 조건
- W1: 스키마/마이그레이션
  - 완료 조건: 롤백 가능한 migration up/down 준비
- W2: Restaurant/Menu CRUD
  - 완료 조건: 생성/수정/삭제 API 통합테스트 20건
- W3: 운영시간/휴무/좌석정책 API
  - 완료 조건: C에서 참조 가능한 gRPC 응답 규약 고정
- W4: 점주 관리 UI 연동
  - 완료 조건: FE 폼 검증과 BE 검증 규칙 1:1 일치
- W5: C 연동
  - 완료 조건: C 예약 계산 시 B 참조 데이터 mismatch 0건
- W6: 인덱스 튜닝
  - 완료 조건: 주요 조회 p95 < 150ms
- W7~W9: 테넌시/회귀/UAT
  - 완료 조건: 테넌트 격리 위반 0건
- W10: 운영 대응
  - 완료 조건: 관리자 수동 보정 SQL 없이 운영 가능

## 6) 교차 점검 항목
- B↔C: `GetReservationReferenceData()`의 `holiday_flag=true`일 때 C는 예약 생성 금지.
- B↔A: 점주가 본인 tenant 외 restaurant_id 조회 시 403 고정.
- B↔E: KPI 집계를 위해 `restaurant_status` 변경 이벤트 발행.

## 7) 정량 목표
- 매장/메뉴 CRUD 성공률 ≥ 99.9%
- 참조데이터 응답 정확도 100%
- 테넌시 위반 0건
