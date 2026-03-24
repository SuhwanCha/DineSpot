# A 담당 문서 — 인증/권한/프로필 (요구사항 분석 + 구현 명세)

## 1) 역할 목표
- 모든 요청은 인증 컨텍스트(`user_id`, `tenant_id`, `role`)를 가진 상태로 downstream에 전달되어야 한다.
- 권한 정책은 코드와 테스트에서 동일한 규칙으로 강제되어야 한다.

## 2) 기능 요구사항 (FR)
- FR-A-01: 이메일/비밀번호 회원가입
- FR-A-02: 로그인(Access/Refresh 발급)
- FR-A-03: 토큰 재발급/로그아웃
- FR-A-04: 프로필 조회/수정
- FR-A-05: RBAC 권한 검증
- FR-A-06: 감사로그용 actor 정보 전달

## 3) API/함수 구현 명세 (Input/Output 포함)

| ID | 함수(예시) | Input | Output | 실패 조건 |
|---|---|---|---|---|
| A-FN-01 | `RegisterUser(ctx, req)` | `email, password, name, role` | `user_id, created_at` | 중복 이메일(`ALREADY_EXISTS`) |
| A-FN-02 | `Login(ctx, req)` | `email, password, device_id` | `access_token, refresh_token, expires_in` | 비밀번호 불일치(`UNAUTHENTICATED`) |
| A-FN-03 | `RefreshToken(ctx, req)` | `refresh_token` | `new_access_token, new_refresh_token` | 폐기 토큰(`TOKEN_REVOKED`) |
| A-FN-04 | `Logout(ctx, req)` | `access_token or refresh_token` | `revoked=true` | 토큰 형식 오류(`INVALID_ARGUMENT`) |
| A-FN-05 | `Authorize(ctx, action, resource)` | `AuthContext + action + resource` | `allow=true/false, reason` | 정책 미정의(`POLICY_NOT_FOUND`) |
| A-FN-06 | `UpdateProfile(ctx, req)` | `name, phone, locale` | `updated_profile` | 유효성 오류(`VALIDATION_FAILED`) |

## 4) 데이터 계약
- JWT claim 필수값
  - `sub`(user_id), `tid`(tenant_id), `rol`(role), `iat`, `exp`, `jti`
- gRPC metadata 필수값
  - `x-user-id`, `x-tenant-id`, `x-role`, `x-request-id`

## 5) 주차별 상세 작업 + 완료 조건
- W1: 토큰/권한 정책 코드화
  - 완료 조건: `Authorize()` 정책 테이블 100% 코드 반영
- W2: Register/Login/Refresh/Logout API
  - 완료 조건: 성공/실패 케이스 단위테스트 25개 이상
- W3: RBAC interceptor 적용
  - 완료 조건: 권한 없는 호출 차단 테스트 100% 통과
- W4: Profile API + FE 연동
  - 완료 조건: FE에서 토큰 만료 시 재발급 후 자동 재시도 성공
- W5: B/C/D 공통 권한 에러코드 정렬
  - 완료 조건: `PERMISSION_DENIED` 사용 규칙 문서/테스트 통일
- W6: 스모크 자동화
  - 완료 조건: CI에서 auth smoke 5분 이내 완료
- W7~W9: 보안/회귀/UAT 대응
  - 완료 조건: 토큰 재사용/위조 테스트 실패율 0%
- W10: 운영 튜닝
  - 완료 조건: 로그인 실패율 대시보드/알람 동작

## 6) 교차 점검 항목 (구체)
- A↔C: `x-user-id` 누락 시 C의 예약 API가 즉시 `UNAUTHENTICATED`를 반환해야 함.
- A↔B: 점주가 타 테넌트 매장 수정 시 `PERMISSION_DENIED` + `tenant_mismatch` reason 반환.
- A↔E: 감사로그에 `actor_id`, `actor_role`, `request_id` 3필드가 반드시 적재.

## 7) 정량 목표 (SLO/품질)
- 인증 API p95 < 120ms
- 로그인 성공률 ≥ 99.5%
- 권한 우회 취약점 0건
- Auth 관련 P1 버그 0건
