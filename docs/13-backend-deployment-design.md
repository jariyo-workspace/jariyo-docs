# 자리요 백엔드 배포 설계서

## 1. 문서 목적

본 문서는 자리요 백엔드를 AWS에 실제로 배포하기 위한 상세 설계서다.

이 문서의 목표는 다음 두 가지다.

* 배포 리소스를 구현자가 더 이상 추상적으로 해석하지 않게 한다.
* 이후 Terraform, ECS 설정, GitHub Actions 구현 작업을 바로 쪼갤 수 있게 한다.

## 2. 최종 배포 구조

최종 권장 구조는 다음과 같다.

* `Route 53`
* `ALB`
* `ECS Cluster`
* `ECS Service: API`
* `ECS Service: Worker`
* `RDS PostgreSQL Multi-AZ`
* `ElastiCache Redis`
* `Secrets Manager`
* `SSM Parameter Store`
* `CloudWatch Logs`
* `CloudWatch Metrics`

현재 단계에서는 아래를 필수로 두지 않는다.

* `WAF`
* `SQS`
* `Blue/Green`

## 3. 리소스 책임

### 3.1 ALB

* 모바일 앱 백엔드의 공개 진입점
* health check 수행
* TLS 종료 지점

### 3.2 ECS API 서비스

* 고객과 운영자 요청 처리
* 동기 상태 변경
* DB 트랜잭션 종료

### 3.3 ECS Worker 서비스

* Outbox 소비
* 비동기 후처리
* 실패 작업 재시도
* 스케줄성 배치성 작업

### 3.4 RDS PostgreSQL

* 기준 상태 저장소
* 이력, 감사 로그, 멱등성, Outbox 저장

### 3.5 ElastiCache Redis

* 읽기 캐시
* 임시성 보조 상태
* DB 보호용 단기 TTL 저장

## 4. 네트워크 설계

기본 설계:

* 단일 리전
* 멀티 AZ
* public subnet: ALB
* private subnet: ECS API, ECS Worker, RDS, Redis

보안 그룹 원칙:

* ALB -> API만 허용
* API/Worker -> RDS 허용
* API/Worker -> Redis 허용
* 외부 -> RDS, Redis 직접 접근 금지

## 5. 애플리케이션 런타임 설계

기본 원칙은 단일 이미지다.

같은 이미지에서 다음 실행 모드만 다르게 둔다.

* `api`
* `worker`

필요한 차이:

* command
* CPU/메모리
* autoscaling 기준
* health check

## 6. 설정값 구조

필수 환경값 범주는 다음과 같다.

### 6.1 공통

* 애플리케이션 이름
* active profile
* timezone

### 6.2 DB

* `DB_URL`
* `DB_USERNAME`
* `DB_PASSWORD`

### 6.3 JWT

* `JWT_ISSUER`
* `JWT_AUDIENCE`
* `JWT_PUBLIC_KEY`
* `JWT_PRIVATE_KEY`

### 6.4 Redis

* host
* port
* auth 또는 연결 관련 값

### 6.5 실행 모드

* `APP_RUNTIME_MODE=api|worker`

## 7. 배포 순서

기본 배포 순서는 다음과 같다.

1. 테스트 실행
2. 컨테이너 이미지 빌드
3. 레지스트리 푸시
4. DB migration 검증
5. API 서비스 배포
6. ALB health check 확인
7. Worker 서비스 배포
8. Outbox 소비 및 핵심 후처리 확인

## 8. 배포 후 검증 시나리오

최소 검증 항목:

* health check 통과
* 공개 API smoke test
* 인증 흐름 확인
* 예약 생성/조회
* 예약 취소와 후처리 확인
* 운영자 권한 흐름 확인
* Worker 로그와 적체 확인

## 9. 구현 작업계획

### 9.1 인프라 레이어

* VPC와 subnet
* security group
* ALB와 target group
* ECS cluster, service, task definition
* RDS
* Redis
* Secrets Manager, SSM
* CloudWatch 로그 그룹과 알람

### 9.2 애플리케이션 레이어

* API/Worker 실행 모드 분리
* Outbox 소비기 실행 위치 확정
* health endpoint 점검
* 환경 변수 매핑 정리

### 9.3 배포 자동화

* GitHub Actions workflow
* 이미지 태깅 전략
* ECS update 단계
* smoke test 단계
* 롤백 기준 정리

## 10. 후속 확장 조건

다음이 보이면 후속 리디자인을 시작한다.

* Redis 없이 감당 가능한 수준을 넘어설 때
* Worker 적체가 지속될 때
* WAF가 필요한 비정상 호출이 늘어날 때
* Rolling 배포 리스크가 커질 때

그때 검토할 항목:

* `WAF`
* `SQS`
* `Blue/Green`
* 읽기 복제본
