# E 담당 문서 — 관리자/운영/관측성 (개발~QA)

## 1. 미션
- 출시 후 운영 가능한 수준의 관측성과 관리자 도구 제공.
- 타 도메인의 품질/장애를 조기 탐지 가능하게 만드는 플랫폼 구축.

## 2. 담당 기술 스택
- Admin UI: Next.js + Chart 라이브러리
- Observability: Prometheus/Grafana/Loki/Tempo(OpenTelemetry)
- 감사로그 저장 및 조회 API

## 3. 제공/소비 인터페이스
- 제공: AdminApproval API, AuditLog API, KPI Dashboard API
- 소비: A/B/C/D의 도메인 이벤트/로그/메트릭

## 4. 주차별 상세 작업
- W1: 공통 로그 포맷/trace id 규약 배포
- W2: 관리자 승인 API + 감사로그 수집기 구현
- W3: KPI 집계 배치/실시간 지표 파이프라인 구현
- W4: 대시보드 1차(예약율/대기시간/취소율)
- W5: 도메인별 알람 룰 설정(에러율/지연/큐 적체)
- W6: 온콜 runbook/장애 대응 템플릿 정비
- W7: GameDay(장애 모의훈련) 실행
- W8: 경보 오탐/미탐 튜닝
- W9: RC 기준 운영체크리스트 완성
- W10: Go-live 모니터링 및 초기 안정화 리딩

## 5. 중간 교차 점검 포인트
- W2 금요일: E↔A/B/C/D 공통 로그 필드 강제 여부 점검
- W4 수요일: E↔PO KPI 정의/계산식 검증
- W7 금요일: 전원 대상 장애대응 리허설 결과 리뷰

## 6. 완료 기준
- 핵심 알람 미탐 0건
- 장애 감지 MTTD 5분 이내
- 운영 runbook 최신화 100%
