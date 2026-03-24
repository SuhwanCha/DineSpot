# A 담당 문서 — 인증/권한/프로필 (개발~QA)

## 1. 미션
- 모든 도메인이 신뢰할 수 있는 인증/인가 기반 제공.
- 고객/점주/관리자 권한 경계를 API 레벨에서 강제.

## 2. 담당 기술 스택
- Go gRPC interceptor, JWT(Access/Refresh), Redis(세션 블랙리스트)
- 비밀번호 해시: Argon2id
- FE: Next.js middleware + route guard

## 3. 제공/소비 인터페이스
- 제공: `AuthService`, `UserService`, `AuthContext` metadata
- 소비: E의 감사로그 수집 엔드포인트

## 4. 주차별 상세 작업
- W1: 토큰 구조/만료정책 구현, 에러코드 표준 적용
- W2: 회원가입/로그인/재발급 API + 단위테스트
- W3: RBAC 미들웨어, role matrix 적용
- W4: 프로필/계정관리 API + FE 연동
- W5: C/B/D 도메인에 권한 컨텍스트 전달 검증
- W6: 인증 스모크 자동화, 실패시나리오 보강
- W7: 보안 점검(토큰 위조/탈취/재사용)
- W8: 회귀 테스트 및 결함 수정
- W9: UAT 지원/권한 이슈 대응
- W10: 운영 알람(로그인 실패율, 토큰 오류율) 튜닝

## 5. 중간 교차 점검 포인트
- W2 수요일: A↔C 인증 subject 필드 일치 점검
- W3 금요일: A↔E 감사로그 필드(`actor_id`, `role`) 정합성 점검
- W5 수요일: A↔B/D 권한 에러코드 통일 점검

## 6. 완료 기준
- 인증 API 오류율 < 0.5%
- 권한 우회 재현 케이스 0건
- Auth 관련 P1 버그 0건
