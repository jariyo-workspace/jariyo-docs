# 자리요 백엔드 로컬 부하 테스트 환경 초안

## 1. 문서 목적

본 문서는 자리요 백엔드를 포트폴리오와 사전 검증 용도로 로컬에서 최대한 배포 유사하게 실행하기 위한 초안 환경을 정의한다.

목표는 다음과 같다.

* `bootRun` 단일 프로세스보다 현실적인 병목을 관찰한다.
* PostgreSQL 분리, 멀티 인스턴스, 프록시 경유를 포함한 최소 구조를 재현한다.
* 운영 환경과 다른 점을 명시해 측정 결과를 과장하지 않는다.

## 2. 적용 범위

이 문서는 `dev`와 `stage` 사이의 `staging-lite` 성격의 로컬 환경을 다룬다.

이 환경은 다음 용도에 적합하다.

* 부하 테스트 스크립트 검증
* API 병목 추정
* DB 락 경합과 커넥션 풀 반응 확인
* 멀티 인스턴스 환경에서의 동시성 기초 검증

다음 용도에는 한계가 있다.

* 실제 ALB, ECS, RDS, ElastiCache 지연 특성 검증
* 오토스케일링 정책 검증
* 운영 네트워크 장애와 멀티 AZ 검증

## 3. 구성 요소

기본 구성은 다음과 같다.

* `nginx` 1개: 로컬 진입점과 라운드로빈 프록시
* `app-a` 1개: Spring Boot API 인스턴스
* `app-b` 1개: Spring Boot API 인스턴스
* `postgres` 1개: 기준 저장소

현재 초안에서 제외한 항목은 다음과 같다.

* `worker`: 현재 애플리케이션이 `api/worker` 런타임 분리를 지원하지 않으므로 단일 앱 내부 스케줄러로 대체
* `redis`: 현재 코드베이스에서 실제 사용하지 않으므로 제외

## 4. 자원 제한 기준

포트폴리오용 기본값은 다음과 같이 둔다.

* `app-a`: `1 CPU`, `1024MB RAM`
* `app-b`: `1 CPU`, `1024MB RAM`
* `postgres`: `0.75 CPU`, `768MB RAM`
* `nginx`: `0.25 CPU`, `128MB RAM`

이 값은 운영 절대치가 아니라 로컬 장비에서 과도한 자원 사용 없이 경합을 보기 위한 시작점이다.

## 5. 실행 파일

저장소 루트에 다음 파일을 둔다.

* `Dockerfile`
* `docker-compose.local-load.yml`
* `docker/local-load/.env.example`
* `docker/local-load/generate-local-jwt-env.sh`
* `docker/local-load/nginx/default.conf`
* `docker/local-load/sql/seed-availability-load-test.sql`
* `load-tests/k6/availability-load.js`

## 6. 실행 절차

1. `docker/local-load/generate-local-jwt-env.sh`를 실행해 로컬 JWT 키와 `docker/local-load/.env`를 생성한다.
2. `docker compose -f docker-compose.local-load.yml up --build -d`로 환경을 기동한다.
3. `http://localhost:8080`으로 진입해 공개 조회 API와 인증 API를 점검한다.
4. 준비된 시드 데이터 또는 별도 테스트 데이터를 적재한다.
5. `k6`, `JMeter`, `nGrinder` 중 하나로 시나리오를 실행한다.

### 6.1 예약 가능 시간 조회 부하 테스트 시드 적재

이 저장소에는 이슈 `#54 예약 가능 시간 조회 부하 테스트` 기준 시드를 함께 둔다.

적재 명령:

```bash
docker exec -i jariyo-local-postgres \
  psql -U jariyo -d jariyo \
  < docker/local-load/sql/seed-availability-load-test.sql
```

시드 기준:

* 매장 ID: `00000000-0000-7000-8000-000000000001`
* 기본 서비스 ID: `00000000-0000-7000-8000-000000009201`
* 보조 서비스 ID: `00000000-0000-7000-8000-000000009202`
* 직원 ID:
  * `00000000-0000-7000-8000-000000009301`
  * `00000000-0000-7000-8000-000000009302`
  * `00000000-0000-7000-8000-000000009303`
  * `00000000-0000-7000-8000-000000009304`

시드는 `CURRENT_DATE + 7`일부터 `CURRENT_DATE + 9`일까지 예약을 미리 만들어 인기 시간대가 일부 점유된 상황을 재현한다.

### 6.2 로그인 비교용 access token 준비

예약 가능 시간 조회 API는 비로그인으로도 호출할 수 있다. 이슈 `#54`의 비교 기준을 맞추려면 같은 시나리오를 비로그인과 로그인 사용자로 각각 실행한다.

로그인 사용자 토큰이 필요하면 아래 순서 중 하나를 사용한다.

1. 테스트 고객을 직접 회원가입한다.
2. 이미 생성한 테스트 고객으로 `POST /api/v1/auth/sign-in`을 호출한다.
3. 응답 본문의 `data.accessToken` 값을 `K6_AUTH_TOKEN` 환경 변수로 전달한다.

예시 요청:

```http
POST /api/v1/auth/sign-up
Content-Type: application/json

{
  "email": "load-member@example.com",
  "password": "load-test-password",
  "displayName": "부하 테스트 고객",
  "phoneNumber": "01012345678",
  "agreements": {
    "terms": true,
    "privacy": true,
    "marketing": false
  }
}
```

주의:

* 현재 비밀번호 정책은 15자 이상이다.
* 회원가입 응답 또는 로그인 응답에서 access token을 추출해야 한다.

### 6.3 k6 예약 가능 시간 조회 시나리오

`load-tests/k6/availability-load.js`는 이슈 `#54`의 `base`, `stressed` 두 프로필을 그대로 반영한다.

프로필 정의:

* `base`
  * 총 사용자 수 `30`
  * 사용자당 요청 수 `3`
  * 목표 총 RPS 약 `9`
  * 허용 기준 `p95 <= 700ms`, `p99 <= 1200ms`, 오류율 `<= 1%`
* `stressed`
  * 총 사용자 수 `120`
  * 사용자당 요청 수 `5`
  * 목표 총 RPS 약 `60`
  * 허용 기준 `p95 <= 1500ms`, `p99 <= 2500ms`, 오류율 `<= 3%`

기본 실행 예시:

```bash
k6 run \
  -e K6_PROFILE=base \
  -e K6_BASE_URL=http://localhost:8080 \
  load-tests/k6/availability-load.js
```

로그인/비로그인 비교 실행 예시:

```bash
K6_AUTH_TOKEN="{access-token}" \
k6 run \
  -e K6_PROFILE=stressed \
  -e K6_BASE_URL=http://localhost:8080 \
  load-tests/k6/availability-load.js
```

선택 환경 변수:

* `K6_PROFILE`: `base` 또는 `stressed`
* `K6_BASE_URL`: 기본값 `http://localhost:8080`
* `K6_AUTH_TOKEN`: 전달하면 로그인 사용자 시나리오를 함께 실행
* `K6_AVAILABILITY_CASES`: 케이스 배열 JSON 문자열
* `K6_RANDOM_SEED`: 인기 시간대 편향 선택용 정수 시드

측정 결과는 최소 아래 항목으로 정리한다.

* `guest`와 `member` 각각의 `p95`, `p99`
* 전체 오류율
* 인기 직원 필터 사용 시와 미사용 시 응답 시간 차이
* 병목 추정 구간

## 7. 검증 관점

이 환경에서는 다음을 우선 본다.

* `p95`, `p99`, 오류율
* 예약 생성 시 DB 락 경합 여부
* 다중 인스턴스에서 동일 슬롯 충돌 방지 유지 여부
* Flyway migration 이후 기동 안정성
* 스케줄러 포함 상태에서 요청 처리 지연 여부

## 8. 해석 원칙

로컬 측정값은 운영 절대 성능 수치가 아니라 비교 지표로 사용한다.

따라서 다음처럼 해석한다.

* 변경 전후 상대 비교에는 사용 가능
* 병목 위치 추정에는 사용 가능
* 운영 수용량 확정 근거로는 사용 불가

## 9. 후속 작업

다음 항목은 실제 부하 테스트 착수 전에 보완 대상이다.

* `#55 일반 예약 생성 부하 테스트`와 `#56 현장 대기 순번 조회 부하 테스트` 시나리오 추가
* `guest`, `member` 분리 결과를 자동 저장하는 리포트 출력 형식 정리
* `worker` 런타임 분리 후 별도 컨테이너 반영
* Redis 도입 시 캐시 포함 구성으로 확장
