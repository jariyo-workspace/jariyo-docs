# 자리요 기획·설계·코드 온보딩 가이드

## 1. 문서 목적

본 문서는 자리요를 새로 보거나, 오랜만에 다시 보는 사람이 짧은 시간 안에 현재 상태를 따라잡을 수 있게 돕는 온보딩 문서다.

대상은 다음과 같다.

* 새로 투입된 백엔드 개발자
* 기존 참여자 중 문서와 코드를 다시 빠르게 훑어야 하는 사람
* 기획, 설계, 구현 사이 연결 지점을 먼저 이해해야 하는 사람

이 문서는 상세 스펙 자체를 대체하지 않는다. 대신 무엇부터 읽고, 어떤 문서를 기준으로 판단하고, 코드에서 어디를 보면 되는지 순서를 잡아준다.

## 2. 15분 요약 경로

시간이 부족하면 아래 순서만 먼저 본다.

1. `01-plan.md`
2. `08-backend-domain-reference.md`
3. `09-backend-system-architecture.md`
4. `10-backend-implementation-guide-v2.md`
5. 루트 저장소의 `src/main/java/com/example/jariyo_backend/domain`

이 순서로 보면 다음 질문에 답할 수 있다.

* 자리요가 어떤 문제를 푸는 서비스인가
* 현재 MVP에서 절대 바꾸면 안 되는 것은 무엇인가
* AWS 기준 권장 운영 구조는 무엇인가
* 구현에서 어떤 계층과 도메인을 먼저 봐야 하는가

## 3. 문서 읽는 순서

### 3.1 기획을 먼저 이해하고 싶을 때

다음 순서로 읽는다.

1. `01-plan.md`
2. `03-screen-plan.md`
3. `05-frontend-api-guide.md`

이 경로는 고객과 운영자가 어떤 화면과 흐름으로 기능을 쓰는지 이해할 때 적합하다.

### 3.2 백엔드 설계를 먼저 이해하고 싶을 때

다음 순서로 읽는다.

1. `08-backend-domain-reference.md`
2. `02-data-model.md`
3. `04-api-spec.md`
4. `09-backend-system-architecture.md`
5. `10-backend-implementation-guide-v2.md`
6. `11-backend-operations-guide.md`

이 경로는 도메인 경계, 상태 전이, API 계약, 인프라 판단 기준을 한 번에 잡을 때 적합하다.

### 3.3 오랜만에 다시 볼 때

다음 순서가 가장 빠르다.

1. `08-backend-domain-reference.md`
2. `09-backend-system-architecture.md`
3. `10-backend-implementation-guide-v2.md`
4. 최근 작업 report
5. 필요한 도메인의 코드와 테스트

즉, 예전 기억을 되살릴 때는 예전 워크스트림 문서보다 새 기준 문서 3개를 먼저 보는 편이 빠르다.

## 4. 지금 기준으로 기억해야 하는 핵심

### 4.1 서비스 핵심

자리요는 예약, 예약 대기, 현장 대기, 운영자 처리를 하나의 흐름으로 묶는 서비스다.

현재 MVP에서 중요한 도메인은 다음 다섯 가지다.

* 인증과 권한
* 매장과 기준 데이터
* 예약
* 예약 대기와 빈자리 제안
* 현장 대기와 운영자 처리

### 4.2 절대 바꾸면 안 되는 것

현재 구현된 MVP 기준에서 다음은 불변이다.

* 공개 API 요청과 응답 계약
* 상태 전이 규칙
* 고객과 운영자 권한 경계
* 멱등성 적용 규칙
* 동시성 충돌을 DB 기준으로 막는 현재 방침

새 설계나 리팩터링을 하더라도 이 기준부터 확인해야 한다.

### 4.3 권장 시스템 구조

현재 권장 백엔드 운영 구조는 다음이다.

* `ECS Fargate`
* 단일 리전 멀티 AZ
* `ALB -> App -> RDS PostgreSQL + Redis`
* `DB Outbox`
* `Secrets Manager + SSM`

즉, 지금은 기능 재설계보다 운영 단순성과 성장 여유를 같이 가져가는 균형안이 기준이다.

## 5. 코드 읽는 순서

### 5.1 엔트리와 공통 기반

먼저 아래를 본다.

* `JariyoBackendApplication`
* `common/api`
* `common/error`
* `common/config`
* `common/idempotency`
* `common/async`

여기서 애플리케이션의 공통 응답, 예외, 보안, 멱등성, 비동기 기록 방식이 정리된다.

### 5.2 도메인 패키지 지도

주요 패키지는 다음처럼 나뉜다.

* `domain/auth`: 회원가입, 로그인, refresh token
* `domain/user`: 내 정보, 매장 멤버 권한
* `domain/store`: 매장, 서비스, 직원, 정책, 운영 설정 조회
* `domain/availability`: 예약 가능 시간 계산
* `domain/reservation`: 예약 생성, 조회, 취소, 운영자 예약 처리
* `domain/waitlist`: 예약 대기, 빈자리 제안
* `domain/walkin`: 현장 대기, 호출, 체크인, 서비스 처리
* `domain/admin`: 운영자 조회, 감사 로그, 서비스 완료

코드를 볼 때는 `controller -> service -> repository/entity` 순서보다, 먼저 `service`를 읽고 필요한 API와 저장소를 거꾸로 따라가는 편이 빠르다.

### 5.3 지금 가장 많이 보게 되는 코드 흐름

자주 건드리는 흐름은 아래와 같다.

* 인증 수정: `auth`, `user`, `common/config`
* 예약 로직 수정: `availability`, `reservation`, `store`
* 예약 대기 수정: `waitlist`, `reservation`
* 현장 대기 수정: `walkin`, `admin`
* 운영자 화면 수정: `admin`, `reservation`, `waitlist`, `walkin`

## 6. 테스트 읽는 순서

### 6.1 빠른 확인

`src/test/java`는 단위 테스트와 컨트롤러 테스트 중심이다.

이 레이어에서 먼저 보는 항목:

* 상태 전이 규칙
* 권한 검증
* 서비스별 예외 처리

### 6.2 흐름 확인

`src/integrationTest/java`는 도메인 흐름을 끝까지 검증한다.

현재 주요 통합 테스트 범위는 다음과 같다.

* `auth`: 인증 흐름
* `reservation`: 예약 생성, 조회, 취소, 충돌
* `waitlist`: 예약 대기와 빈자리 제안
* `walkin`: 현장 대기 전체 흐름
* `admin`: 운영자 예약 체크인, 서비스 시작, 완료
* `store`: 매장 기준 데이터와 설정 관련 흐름

새 기능을 이해할 때는 문서보다 해당 도메인의 통합 테스트를 먼저 보는 편이 더 빠를 때가 많다.

## 7. 로컬 실행과 검증

기본 실행 명령은 다음을 기억한다.

* `./gradlew test`
* `./gradlew integrationTest`
* `./gradlew bootRun`

프로젝트는 다음 전제를 가진다.

* Java 17
* Spring Boot 4.x
* PostgreSQL
* Flyway
* JWT 기반 인증
* `integrationTest`는 Testcontainers PostgreSQL 사용

기본 환경값은 `application.properties`에서 확인할 수 있다.

## 8. 새 작업을 시작할 때 체크리스트

작업 전에 아래를 확인한다.

1. 관련 도메인 기준 문서를 읽었는가
2. API 계약이 이미 존재하는가
3. 상태 전이와 권한 규칙을 바꾸는가
4. 멱등성 적용 대상인가
5. 통합 테스트로 기존 흐름을 다시 확인했는가
6. 문서 수정이 같이 필요한가

문서를 먼저 읽고 코드를 본 뒤, 마지막에 테스트를 확인하는 순서를 권장한다.

## 9. 헷갈리기 쉬운 포인트

* 고객과 운영자는 같은 API 경계에 있지만, 같은 권한 모델이 아니다.
* Redis나 큐가 생겨도 최종 기준 데이터는 PostgreSQL이다.
* 운영자 조회는 별도 진실의 원천이 아니라 읽기용 조합 계층이다.
* 예전 `06`, `07` 문서는 legacy이고, 현재 판단 기준은 `08`부터 `11`이다.
* 문서 리디자인은 있었지만 MVP 동작 자체를 바꾸는 작업은 아니었다.

## 10. 추천 온보딩 루트

### 기획 중심

* `01-plan.md`
* `03-screen-plan.md`
* `05-frontend-api-guide.md`

### 설계 중심

* `08-backend-domain-reference.md`
* `02-data-model.md`
* `04-api-spec.md`
* `09-backend-system-architecture.md`

### 코드 중심

* `10-backend-implementation-guide-v2.md`
* `src/main/java/com/example/jariyo_backend/common`
* `src/main/java/com/example/jariyo_backend/domain`
* `src/integrationTest/java/com/example/jariyo_backend`

이 세 경로를 모두 본 뒤에는 대부분의 변경 작업에 착수할 수 있다.
