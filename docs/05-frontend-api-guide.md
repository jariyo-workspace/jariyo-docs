# 자리요 프론트엔드 연동 명세서

## 1. 문서 목적

본 문서는 자리요 프론트엔드 개발에 필요한 API 호출 규칙, 주요 화면별 데이터, 상태값, 오류 처리 기준을 간단하게 정리한다.

전체 API의 세부 구현보다 다음 항목에 집중한다.

* 화면에서 어떤 API를 호출하는지
* 어떤 데이터를 표시해야 하는지
* 어떤 상태에서 어떤 버튼을 보여주는지
* 요청 성공과 실패를 어떻게 처리하는지
* 실시간 상태 변경을 어떻게 반영하는지

---

# 2. 공통 API 규칙

## Base URL

```text
/api/v1
```

## 인증 헤더

로그인이 필요한 요청에는 다음 헤더를 포함한다.

```http
Authorization: Bearer {accessToken}
```

## 요청 형식

```http
Content-Type: application/json
Accept: application/json
```

## 성공 응답

단일 데이터:

```json
{
  "data": {
    "id": "res_123"
  }
}
```

목록 데이터:

```json
{
  "data": [],
  "page": {
    "cursor": "next_cursor",
    "hasNext": true
  }
}
```

## 오류 응답

```json
{
  "error": {
    "code": "RESERVATION_SLOT_ALREADY_TAKEN",
    "message": "방금 다른 고객이 이 시간을 예약했어요.",
    "details": {},
    "requestId": "req_123"
  }
}
```

프론트엔드는 일반적으로 `error.message`를 사용자에게 표시한다.

`error.code`는 화면 분기와 특수 처리에 사용한다.

---

# 3. 시간 표시 규칙

서버에서 전달되는 일시는 시간대가 포함된 ISO 8601 형식이다.

```text
2026-07-18T14:00:00+09:00
```

화면에는 다음 형식으로 표시한다.

```text
7월 18일 토요일 오후 2:00
```

상대 시간은 보조 정보로만 사용한다.

```text
내일 · 오후 2:00
```

빈자리 제안처럼 만료 시간이 중요한 경우 서버에서 받은 `expiresAt`을 기준으로 남은 시간을 계산한다.

클라이언트 시계만 신뢰하지 말고, 화면 재진입 시 상세 API를 다시 호출한다.

---

# 4. 멱등성 키

다음 요청에는 `Idempotency-Key` 헤더를 포함한다.

* 예약 생성
* 예약 취소
* 예약 대기 신청 및 취소
* 빈자리 제안 수락 및 거절
* 현장 대기 등록 및 취소
* 체크인
* 고객 호출 응답
* 운영자 상태 변경

예:

```http
Idempotency-Key: 0190f7c8-92ef-7a52-a284-2ef34dd84155
```

## 생성 기준

사용자가 버튼을 누르는 시점에 새로운 키를 생성한다.

같은 요청을 네트워크 오류로 다시 전송할 때는 기존 키를 재사용한다.

사용자가 요청 내용을 변경한 뒤 다시 시도할 경우에는 새로운 키를 생성한다.

---

# 5. 인증 상태 처리

## Access Token 만료

다음 오류가 발생하면 토큰 재발급을 시도한다.

```text
ACCESS_TOKEN_EXPIRED
INVALID_ACCESS_TOKEN
```

재발급 성공:

1. 원래 요청 재시도
2. 현재 화면 유지

재발급 실패:

1. 로그인 화면 이동
2. 예약 진행 중이었다면 선택 정보 임시 보존
3. 로그인 후 가능한 경우 이전 흐름 복원

---

# 6. 고객용 화면별 API

## 6.1 고객 홈

### 화면 진입 API

```http
GET /api/v1/me/home
```

### 주요 응답 데이터

```json
{
  "data": {
    "priorityAction": {
      "type": "SLOT_OFFER",
      "referenceId": "offer_123",
      "title": "원하던 시간에 빈자리가 생겼어요",
      "expiresAt": "2026-07-11T10:03:00+09:00"
    },
    "upcomingReservations": [],
    "activeWaitlists": [],
    "activeWalkIns": [],
    "recentStores": []
  }
}
```

### priorityAction 우선순위

다음 순서로 가장 중요한 행동 하나를 강조한다.

1. 현장 고객 호출
2. 빈자리 제안
3. 체크인 가능 예약
4. 오늘 예약
5. 현장 대기
6. 예약 대기

### 화면 이동

| type                 | 이동 화면        |
| -------------------- | ------------ |
| `SLOT_OFFER`         | 빈자리 제안 상세    |
| `WALK_IN_CALLED`     | 현장 호출 화면     |
| `CHECK_IN_AVAILABLE` | 예약 상세 또는 체크인 |
| `RESERVATION`        | 예약 상세        |
| `WAITLIST`           | 예약 대기 상세     |
| `WALK_IN`            | 현장 대기 상세     |

---

## 6.2 매장 목록

```http
GET /api/v1/stores
```

### 쿼리 예시

```text
query
openNow
availableToday
cursor
limit
```

### 주요 표시 정보

* 매장명
* 주소
* 영업 여부
* 오늘 영업시간
* 가장 빠른 예약 가능 시간
* 현장 대기 가능 여부
* 예약 대기 가능 여부

### 빈 결과

```text
검색 결과가 없어요.
다른 매장명이나 지역으로 검색해 보세요.
```

---

## 6.3 매장 상세

```http
GET /api/v1/stores/{storeId}
GET /api/v1/stores/{storeId}/services
```

### 주요 표시 정보

* 매장명
* 소개
* 주소
* 전화번호
* 영업 상태
* 영업시간
* 예약 정책
* 서비스 목록
* 예약하기 버튼
* 현장 대기 버튼

### 버튼 표시 조건

| 조건        | 예약 버튼 |  현장 대기 버튼 |
| --------- | ----: | --------: |
| 매장 운영 중   |    표시 | 정책에 따라 표시 |
| 임시 휴무     |   비활성 |       비활성 |
| 현장 대기 미운영 |    표시 |        숨김 |
| 예약 미운영    |    숨김 | 정책에 따라 표시 |

---

## 6.4 서비스 선택

```http
GET /api/v1/stores/{storeId}/services
```

### 카드 데이터

```json
{
  "id": "service_123",
  "name": "커트",
  "description": "기본 커트 서비스",
  "durationMinutes": 30,
  "cleanupMinutes": 10,
  "status": "ACTIVE",
  "availableStaffCount": 3
}
```

`status !== ACTIVE`인 서비스는 선택할 수 없게 표시한다.

---

## 6.5 직원 선택

```http
GET /api/v1/stores/{storeId}/services/{serviceId}/staff
```

### 옵션

* 가능한 직원 누구나
* 특정 직원 선택

### 주요 데이터

```json
{
  "id": "member_123",
  "displayName": "민지",
  "bookingEnabled": true,
  "customDurationMinutes": 40,
  "nextAvailableAt": "2026-07-18T14:00:00+09:00"
}
```

`bookingEnabled === false`인 직원은 선택할 수 없다.

---

## 6.6 예약 가능 시간 선택

```http
GET /api/v1/stores/{storeId}/availability
```

### 필수 쿼리

```text
serviceId
from
to
partySize
```

### 선택 쿼리

```text
staffId
```

### 슬롯 데이터

```json
{
  "startAt": "2026-07-18T14:00:00+09:00",
  "serviceEndAt": "2026-07-18T14:30:00+09:00",
  "occupiedUntil": "2026-07-18T14:40:00+09:00",
  "staffId": "member_123",
  "status": "AVAILABLE"
}
```

### 계산 규칙

* 슬롯 시작 시각은 30분 단위다.
* `staffId` 없이 조회하면 예약 가능한 직원별 슬롯이 함께 내려온다.
* 같은 시작 시각이면 직원명 오름차순으로 정렬된다.
* `occupiedUntil`까지는 같은 직원의 다음 예약이 불가능하다고 본다.

### 슬롯 상태

| 상태                   | 화면 처리       |
| -------------------- | ----------- |
| `AVAILABLE`          | 선택 가능       |
| `UNAVAILABLE`        | 비활성         |
| `WAITLIST_AVAILABLE` | 대기 신청 버튼 제공 |

### 주의

조회 당시 가능했던 시간도 예약 요청 시 다른 고객이 먼저 가져갈 수 있다.

`RESERVATION_SLOT_ALREADY_TAKEN` 오류가 발생하면:

1. 오류 안내
2. 가능 시간 API 재호출
3. 기존 선택 해제
4. 대기 신청 진입 제공

---

## 6.7 예약 확인 및 생성

```http
POST /api/v1/reservations
```

### 헤더

```http
Idempotency-Key: {key}
```

### 요청

```json
{
  "storeId": "store_123",
  "serviceId": "service_123",
  "staffId": "member_123",
  "startAt": "2026-07-18T14:00:00+09:00",
  "partySize": 1,
  "customerNote": "앞머리는 짧지 않게 부탁드립니다."
}
```

### 요청 중 처리

* 예약 버튼 비활성화
* 중복 클릭 방지
* 로딩 문구 표시

```text
예약을 확인하고 있어요.
```

### 성공

예약 완료 화면으로 이동한다.

### 주요 오류

| 오류 코드                                           | 처리                   |
| ----------------------------------------------- | -------------------- |
| `RESERVATION_SLOT_ALREADY_TAKEN`                | 시간 선택 화면으로 이동 후 새로고침 |
| `STAFF_NOT_AVAILABLE`                           | 직원 또는 시간 재선택         |
| `RESERVATION_OUTSIDE_BOOKING_WINDOW`            | 예약 가능 기간 안내          |
| `CUSTOMER_HAS_OVERLAPPING_RESERVATION`          | 겹치는 기존 예약 안내         |
| `IDEMPOTENCY_KEY_REUSED_WITH_DIFFERENT_REQUEST` | 새로운 키 생성 후 재시도 유도    |

### 결과 확인 불가

네트워크 오류로 성공 여부를 판단할 수 없다면 즉시 새 예약을 만들지 않는다.

안내:

```text
예약 결과를 바로 확인하지 못했어요.
내 예약에서 처리 결과를 확인해 주세요.
```

이후 내 예약 API를 조회한다.

---

## 6.8 내 예약 목록

```http
GET /api/v1/me/reservations
```

### 쿼리

```text
status
from
to
cursor
limit
```

### 상태별 분류

| 탭     | 포함 상태                                          |
| ----- | ---------------------------------------------- |
| 예정    | `HELD`, `CONFIRMED`, `CHECKED_IN`              |
| 이용 중  | `IN_SERVICE`                                   |
| 지난 내역 | `COMPLETED`, `CANCELLED`, `NO_SHOW`, `EXPIRED` |

---

## 6.9 예약 상세

```http
GET /api/v1/reservations/{reservationId}
```

### 주요 필드

```json
{
  "id": "res_123",
  "status": "CONFIRMED",
  "startAt": "2026-07-18T14:00:00+09:00",
  "serviceEndAt": "2026-07-18T14:30:00+09:00",
  "checkInAvailable": false,
  "canCancel": true,
  "cancellationDeadlineAt": "2026-07-17T14:00:00+09:00"
}
```

### 상태별 표시

| 상태           | 사용자 문구     | 주요 버튼            |
| ------------ | ---------- | ---------------- |
| `HELD`       | 예약 확인 중    | 예약 확정            |
| `CONFIRMED`  | 예약 확정      | 취소, 체크인 가능 시 체크인 |
| `CHECKED_IN` | 체크인 완료     | 별도 행동 없음         |
| `IN_SERVICE` | 이용 중       | 별도 행동 없음         |
| `COMPLETED`  | 이용 완료      | 다시 예약            |
| `CANCELLED`  | 예약 취소      | 다시 예약            |
| `NO_SHOW`    | 미방문 처리     | 매장 문의            |
| `EXPIRED`    | 예약 시간이 만료됨 | 다시 예약            |

### 취소 버튼

`canCancel === true`일 때만 활성화한다.

---

## 6.10 예약 취소

```http
POST /api/v1/reservations/{reservationId}/cancel
```

### 요청

```json
{
  "reasonCode": "CUSTOMER_SCHEDULE_CHANGED",
  "reason": "개인 일정이 생겼습니다."
}
```

### 성공

* 상세 화면의 상태를 `CANCELLED`로 갱신
* 예정 예약 목록 캐시 무효화
* 홈 데이터 재조회

### 주요 오류

| 오류 코드                                      | 처리                 |
| ------------------------------------------ | ------------------ |
| `RESERVATION_ALREADY_CANCELLED`            | 취소 상태로 화면 동기화      |
| `RESERVATION_CANCELLATION_DEADLINE_PASSED` | 취소 불가 안내와 매장 연락 버튼 |
| `RESERVATION_INVALID_STATE`                | 상세 API 재조회         |

---

## 6.11 예약 대기 신청

```http
POST /api/v1/waitlists
```

### 요청

```json
{
  "storeId": "store_123",
  "serviceId": "service_123",
  "preferredStaffId": "member_123",
  "staffPreferenceType": "SPECIFIC_PREFERRED",
  "desiredDate": "2026-07-18",
  "acceptableStartTime": "13:00",
  "acceptableEndTime": "17:00",
  "partySize": 1
}
```

### 직원 선호 타입

| 값                    | 의미                 |
| -------------------- | ------------------ |
| `SPECIFIC_ONLY`      | 선택 직원만 가능          |
| `SPECIFIC_PREFERRED` | 선택 직원 우선, 다른 직원 가능 |
| `ANY_STAFF`          | 직원 무관              |

### 주요 오류

| 오류 코드                         | 처리           |
| ----------------------------- | ------------ |
| `WAITLIST_DUPLICATED`         | 기존 대기 상세로 이동 |
| `WAITLIST_NOT_ENABLED`        | 대기 기능 미지원 안내 |
| `INVALID_WAITLIST_TIME_RANGE` | 시간 입력 오류 표시  |
| `WAITLIST_DATE_OUT_OF_RANGE`  | 날짜 재선택       |

---

## 6.12 예약 대기 상세

```http
GET /api/v1/waitlists/{waitlistId}
```

### 상태별 표시

| 상태          | 표시            |
| ----------- | ------------- |
| `WAITING`   | 빈자리 대기 중      |
| `OFFERED`   | 예약 가능한 자리가 생김 |
| `RESERVED`  | 예약으로 전환됨      |
| `CANCELLED` | 대기 취소         |
| `EXPIRED`   | 대기 기간 종료      |

`activeOffer`가 존재하면 빈자리 제안 화면으로 이동할 수 있는 버튼을 제공한다.

---

## 6.13 예약 대기 취소

```http
POST /api/v1/waitlists/{waitlistId}/cancel
```

성공 후 상태를 `CANCELLED`로 변경한다.

`WAITLIST_INVALID_STATE`가 발생하면 상세 API를 다시 조회한다.

---

## 6.14 빈자리 제안 상세

```http
GET /api/v1/slot-offers/{offerId}
```

### 주요 데이터

```json
{
  "id": "offer_123",
  "status": "PENDING",
  "startAt": "2026-07-18T15:00:00+09:00",
  "expiresAt": "2026-07-11T10:03:00+09:00",
  "remainingSeconds": 125
}
```

### 상태별 처리

| 상태         | 화면                |
| ---------- | ----------------- |
| `PENDING`  | 수락 및 거절 버튼        |
| `ACCEPTED` | 생성된 예약으로 이동       |
| `DECLINED` | 거절 완료 안내          |
| `EXPIRED`  | 제안 만료 안내          |
| `REVOKED`  | 매장에서 제안을 취소했다는 안내 |

남은 시간이 0이 되면 버튼을 비활성화하고 상세 API를 다시 호출한다.

---

## 6.15 빈자리 제안 수락

```http
POST /api/v1/slot-offers/{offerId}/accept
```

### 요청 중

* 수락 버튼 비활성화
* 타이머는 계속 표시
* 중복 클릭 방지

### 성공

응답에 포함된 예약 상세 또는 예약 완료 화면으로 이동한다.

### 주요 오류

| 오류 코드                            | 처리        |
| -------------------------------- | --------- |
| `SLOT_OFFER_EXPIRED`             | 만료 화면 표시  |
| `SLOT_OFFER_ALREADY_ACCEPTED`    | 생성된 예약 조회 |
| `SLOT_OFFER_NO_LONGER_AVAILABLE` | 대기 유지 안내  |
| `RESERVATION_SLOT_ALREADY_TAKEN` | 대기 유지 안내  |
| `WAITLIST_INVALID_STATE`         | 대기 상세 재조회 |

---

## 6.16 빈자리 제안 거절

```http
POST /api/v1/slot-offers/{offerId}/decline
```

### 요청

```json
{
  "keepWaitlistActive": true
}
```

`keepWaitlistActive === true`이면 대기 상세로 이동한다.

---

## 6.17 현장 대기 가능 상태 조회

```http
GET /api/v1/stores/{storeId}/walk-in-status
```

### 표시 정보

* 접수 가능 여부
* 현재 대기 인원
* 예상 대기 시간
* 접수 마감 시간

`acceptingEntries === false`면 등록 버튼을 비활성화한다.

---

## 6.18 현장 대기 등록

```http
POST /api/v1/walk-ins
```

### 요청

```json
{
  "storeId": "store_123",
  "serviceId": "service_123",
  "preferredStaffId": null,
  "partySize": 1
}
```

### 성공

현장 대기 상세로 이동한다.

### 주요 오류

| 오류 코드                         | 처리              |
| ----------------------------- | --------------- |
| `WALK_IN_ALREADY_REGISTERED`  | 기존 현장 대기 상세로 이동 |
| `WALK_IN_REGISTRATION_CLOSED` | 접수 마감 안내        |
| `WALK_IN_NOT_ENABLED`         | 기능 미지원 안내       |

---

## 6.19 현장 대기 상세

```http
GET /api/v1/walk-ins/{walkInId}
```

### 주요 필드

```json
{
  "queueNumber": 12,
  "status": "WAITING",
  "waitingAhead": 4,
  "estimatedWaitMinutes": 30,
  "calledAt": null,
  "callExpiresAt": null
}
```

### 상태별 화면

| 상태           | 표시       | 주요 버튼      |
| ------------ | -------- | ---------- |
| `WAITING`    | 대기 중     | 대기 취소      |
| `CALLED`     | 입장 요청    | 입장, 지연, 취소 |
| `CHECKED_IN` | 입장 확인됨   | 없음         |
| `IN_SERVICE` | 이용 중     | 없음         |
| `COMPLETED`  | 이용 완료    | 없음         |
| `SKIPPED`    | 순서 보류    | 매장 문의      |
| `CANCELLED`  | 대기 취소    | 없음         |
| `NO_SHOW`    | 응답 없음 처리 | 매장 문의      |

### 갱신

실시간 이벤트 또는 주기적인 재조회로 상태를 갱신한다.

실시간 연결 실패 시에는 일정 간격으로 상세 API를 재호출한다.

---

## 6.20 호출 응답

```http
POST /api/v1/walk-ins/{walkInId}/respond-call
```

### 요청

```json
{
  "response": "ENTERING"
}
```

### response 값

| 값          | 화면 행동    |
| ---------- | -------- |
| `ENTERING` | 입장 확인 처리 |
| `DELAYED`  | 지연 의사 전달 |
| `CANCEL`   | 대기 취소    |

### 주요 오류

| 오류 코드                   | 처리                |
| ----------------------- | ----------------- |
| `WALK_IN_CALL_EXPIRED`  | 상세 재조회 후 만료 상태 표시 |
| `WALK_IN_INVALID_STATE` | 상세 API 재조회        |

---

## 6.21 예약 체크인

```http
GET /api/v1/reservations/{reservationId}/check-in-availability
POST /api/v1/reservations/{reservationId}/check-in
```

### 체크인 가능 여부

```json
{
  "available": true,
  "availableFrom": "2026-07-18T13:40:00+09:00",
  "availableUntil": "2026-07-18T14:10:00+09:00",
  "alreadyCheckedIn": false
}
```

### 상태별 버튼

| 조건                          | 처리         |
| --------------------------- | ---------- |
| `available === true`        | 체크인 버튼 활성화 |
| 아직 체크인 시간 전                 | 활성화 시각 안내  |
| 체크인 시간 종료                   | 직원 문의 안내   |
| `alreadyCheckedIn === true` | 체크인 완료 표시  |

---

# 7. 알림 화면 API

## 알림 목록

```http
GET /api/v1/me/notifications
```

## 알림 읽음 처리

```http
POST /api/v1/notifications/{notificationId}/read
```

## 전체 읽음

```http
POST /api/v1/me/notifications/read-all
```

### 알림 클릭 이동

| referenceType    | 이동       |
| ---------------- | -------- |
| `RESERVATION`    | 예약 상세    |
| `WAITLIST_ENTRY` | 예약 대기 상세 |
| `SLOT_OFFER`     | 빈자리 제안   |
| `WALK_IN_ENTRY`  | 현장 대기 상세 |

이미 만료된 알림이어도 연결된 상세 API를 호출해 최신 상태를 보여준다.

---

# 8. 운영자 화면별 API

## 8.1 오늘의 운영 홈

```http
GET /api/v1/admin/stores/{storeId}/dashboard/today
```

### 표시 항목

* 오늘 예약 수
* 현장 대기 수
* 체크인 고객
* 서비스 진행 중
* 노쇼 후보
* 응답 대기 중인 빈자리 제안
* 운영 경고
* 시간순 일정

경고 카드 클릭 시 관련 상세 화면으로 이동한다.

---

## 8.2 예약 캘린더

```http
GET /api/v1/admin/stores/{storeId}/reservations
```

### 쿼리

```text
from
to
staffId
serviceId
status
```

### 예약 블록 필수 정보

* 고객명
* 서비스명
* 담당 직원
* 시작 시간
* 종료 시간
* 예약 상태
* 체크인 상태

상태 변경 후에는 전체 캘린더를 무조건 재호출하기보다 해당 예약 상세과 관련 기간 목록을 갱신한다.

---

## 8.3 운영자 예약 상세

```http
GET /api/v1/admin/stores/{storeId}/reservations/{reservationId}
```

### 상태별 행동

| 예약 상태        | 가능한 행동                     |
| ------------ | -------------------------- |
| `CONFIRMED`  | 체크인, 시간 변경, 담당자 변경, 취소, 노쇼 |
| `CHECKED_IN` | 서비스 시작, 취소                 |
| `IN_SERVICE` | 서비스 완료                     |
| `COMPLETED`  | 조회만 가능                     |
| `CANCELLED`  | 조회만 가능                     |
| `NO_SHOW`    | 조회만 가능                     |

프론트엔드 버튼 노출과 별개로 서버가 최종 권한과 상태를 검증한다.

---

## 8.4 수동 예약 등록

```http
POST /api/v1/admin/stores/{storeId}/reservations
```

충돌 오류가 발생하면 저장하지 않고 가능한 시간 재선택을 유도한다.

```text
RESERVATION_SLOT_ALREADY_TAKEN
STAFF_NOT_AVAILABLE
```

---

## 8.5 예약 시간 변경

```http
POST /api/v1/admin/stores/{storeId}/reservations/{reservationId}/reschedule
```

### 성공 후 갱신

* 예약 상세
* 캘린더
* 오늘 운영 홈
* 관련 직원 일정

---

## 8.6 운영자 현장 대기열

```http
GET /api/v1/admin/stores/{storeId}/walk-ins
```

### 목록 상태 그룹

* 호출 필요
* 대기 중
* 호출 중
* 보류
* 체크인 완료
* 서비스 진행 중

### 카드 주요 정보

* 대기 번호
* 고객명
* 서비스
* 희망 직원
* 등록 시각
* 대기 시간
* 현재 상태

---

## 8.7 현장 고객 호출

```http
POST /api/v1/admin/stores/{storeId}/walk-ins/{walkInId}/call
```

### 요청

```json
{
  "responseTimeoutMinutes": 3,
  "sendNotification": true
}
```

### 성공

상태를 `CALLED`로 변경하고 타이머를 표시한다.

### 주요 오류

| 오류 코드                    | 처리           |
| ------------------------ | ------------ |
| `WALK_IN_ALREADY_CALLED` | 상세 또는 목록 재조회 |
| `WALK_IN_INVALID_STATE`  | 최신 상태 재조회    |

---

## 8.8 현장 대기 보류 및 복귀

보류:

```http
POST /api/v1/admin/stores/{storeId}/walk-ins/{walkInId}/skip
```

복귀:

```http
POST /api/v1/admin/stores/{storeId}/walk-ins/{walkInId}/restore
```

상태 변경 후 대기열 순서 전체를 다시 불러오는 것이 안전하다.

---

## 8.9 서비스 시작

예약 고객:

```http
POST /api/v1/admin/stores/{storeId}/reservations/{reservationId}/start-service
```

현장 고객:

```http
POST /api/v1/admin/stores/{storeId}/walk-ins/{walkInId}/start-service
```

성공 응답의 `serviceSessionId`를 보관한다.

---

## 8.10 서비스 완료

```http
POST /api/v1/admin/stores/{storeId}/service-sessions/{sessionId}/complete
```

### 요청

```json
{
  "completionNote": "정상 완료"
}
```

성공 후 다음 데이터를 갱신한다.

* 예약 또는 현장 대기 상태
* 오늘 운영 홈
* 운영 목록
* 캘린더

---

## 8.11 서비스 관리

목록:

```http
GET /api/v1/admin/stores/{storeId}/services
```

생성:

```http
POST /api/v1/admin/stores/{storeId}/services
```

수정:

```http
PUT /api/v1/admin/stores/{storeId}/services/{serviceId}
```

활성화:

```http
POST /api/v1/admin/stores/{storeId}/services/{serviceId}/activate
```

비활성화:

```http
POST /api/v1/admin/stores/{storeId}/services/{serviceId}/deactivate
```

기존 예약이 있는 서비스는 삭제하지 않고 비활성화한다.

---

## 8.12 직원 관리

목록:

```http
GET /api/v1/admin/stores/{storeId}/staff
```

상세:

```http
GET /api/v1/admin/stores/{storeId}/staff/{staffId}
```

수정:

```http
PUT /api/v1/admin/stores/{storeId}/staff/{staffId}
```

담당 서비스 설정:

```http
PUT /api/v1/admin/stores/{storeId}/staff/{staffId}/services
```

비활성화:

```http
POST /api/v1/admin/stores/{storeId}/staff/{staffId}/deactivate
```

`STAFF_HAS_FUTURE_RESERVATIONS` 오류 시 예정 예약 목록을 보여준다.

---

## 8.13 근무 일정 관리

조회:

```http
GET /api/v1/admin/stores/{storeId}/staff/{staffId}/schedules
```

설정:

```http
PUT /api/v1/admin/stores/{storeId}/staff/{staffId}/schedules
```

일정 예외 생성:

```http
POST /api/v1/admin/stores/{storeId}/staff/{staffId}/schedule-exceptions
```

충돌 예약이 반환되면 변경을 완료한 것으로 간주하지 않는다.

---

# 9. 주요 상태값 정리

## ReservationStatus

```text
HELD
CONFIRMED
CHECKED_IN
IN_SERVICE
COMPLETED
CANCELLED
NO_SHOW
EXPIRED
```

## WaitlistStatus

```text
WAITING
OFFERED
RESERVED
CANCELLED
EXPIRED
```

## SlotOfferStatus

```text
PENDING
ACCEPTED
DECLINED
EXPIRED
REVOKED
```

## WalkInStatus

```text
WAITING
CALLED
CHECKED_IN
IN_SERVICE
COMPLETED
SKIPPED
CANCELLED
NO_SHOW
```

## ServiceSessionStatus

```text
READY
IN_PROGRESS
COMPLETED
CANCELLED
```

---

# 10. 사용자 표시 문구

| 서버 상태        | 고객 화면 문구 |
| ------------ | -------- |
| `HELD`       | 예약 확인 중  |
| `CONFIRMED`  | 예약 확정    |
| `CHECKED_IN` | 체크인 완료   |
| `IN_SERVICE` | 이용 중     |
| `COMPLETED`  | 이용 완료    |
| `CANCELLED`  | 취소됨      |
| `NO_SHOW`    | 미방문      |
| `EXPIRED`    | 만료됨      |
| `WAITING`    | 대기 중     |
| `OFFERED`    | 예약 가능    |
| `CALLED`     | 입장 요청    |
| `SKIPPED`    | 순서 보류    |

상태 문구는 화면마다 임의로 다르게 만들지 않고 공통 매핑을 사용한다.

---

# 11. 로딩 상태 기준

## 전체 화면 로딩

첫 진입 시 화면 전체 데이터가 필요한 경우 사용한다.

예:

* 고객 홈
* 매장 상세
* 예약 상세
* 운영자 대시보드

## 부분 로딩

화면 일부 데이터만 갱신하는 경우 사용한다.

예:

* 날짜 변경 후 예약 가능 시간
* 대기 상태 갱신
* 알림 읽음 처리

## 버튼 로딩

사용자 명령 요청 중 버튼에 적용한다.

예:

* 예약 확정
* 예약 취소
* 빈자리 수락
* 체크인
* 현장 호출
* 서비스 시작 및 완료

요청 중 동일한 행동을 다시 수행할 수 없도록 한다.

---

# 12. 오류 처리 기준

## 입력 오류

필드 단위 오류를 해당 입력창 아래에 표시한다.

```text
INVALID_FIELD_FORMAT
MISSING_REQUIRED_FIELD
INVALID_DATE_RANGE
INVALID_PARTY_SIZE
```

## 상태 충돌

현재 화면이 오래된 상태이므로 관련 상세 API를 다시 호출한다.

```text
RESERVATION_INVALID_STATE
WAITLIST_INVALID_STATE
WALK_IN_INVALID_STATE
SERVICE_SESSION_INVALID_STATE
```

## 동시성 충돌

사용자에게 상황을 설명하고 최신 데이터를 다시 조회한다.

```text
RESERVATION_SLOT_ALREADY_TAKEN
SLOT_OFFER_NO_LONGER_AVAILABLE
```

## 네트워크 오류

쓰기 요청의 결과가 불명확한 경우 바로 같은 작업을 새로운 멱등성 키로 재시도하지 않는다.

기존 키로 재시도하거나 관련 상세·목록을 조회해 결과를 확인한다.

## 5xx 및 일시 장애

```text
잠시 연결이 원활하지 않아요.
현재 예약과 대기 정보는 유지되고 있어요.
```

가능한 경우 재시도 버튼을 제공한다.

---

# 13. 캐시 및 데이터 갱신 기준

## 예약 생성 후 무효화

* 예약 가능 시간
* 내 예약 목록
* 고객 홈
* 관련 매장 상세 요약

## 예약 취소 후 무효화

* 예약 상세
* 내 예약 목록
* 예약 가능 시간
* 고객 홈
* 운영자 예약 목록

## 빈자리 제안 수락 후 무효화

* 빈자리 제안
* 예약 대기 상세
* 내 예약 목록
* 고객 홈
* 예약 가능 시간

## 현장 대기 상태 변경 후 무효화

* 현장 대기 상세
* 고객 홈
* 운영자 대기열
* 운영자 오늘 홈

## 서비스 시작·완료 후 무효화

* 관련 예약 또는 현장 대기 상세
* 운영자 대시보드
* 운영 목록
* 예약 캘린더

---

# 14. 실시간 이벤트 처리

고객 이벤트 스트림:

```http
GET /api/v1/me/events/stream
```

운영자 이벤트 스트림:

```http
GET /api/v1/admin/stores/{storeId}/events/stream
```

## 고객 주요 이벤트

```text
RESERVATION_UPDATED
WAITLIST_UPDATED
SLOT_OFFER_CREATED
SLOT_OFFER_EXPIRED
WALK_IN_UPDATED
WALK_IN_CALLED
NOTIFICATION_CREATED
```

## 운영자 주요 이벤트

```text
RESERVATION_CREATED
RESERVATION_CANCELLED
CUSTOMER_CHECKED_IN
WALK_IN_CREATED
WALK_IN_UPDATED
SERVICE_STARTED
SERVICE_COMPLETED
```

## 처리 원칙

이벤트를 받으면 이벤트 데이터만으로 화면 상태를 확정하지 않는다.

관련 상세 또는 목록 API를 무효화하고 최신 데이터를 다시 조회한다.

실시간 연결이 끊겨도 핵심 기능이 동작해야 한다.

---

# 15. 프론트엔드 구현 우선순위

## 1차

* 로그인
* 매장 상세
* 서비스 선택
* 직원 선택
* 예약 가능 시간
* 예약 생성
* 내 예약 목록
* 예약 상세
* 예약 취소

## 2차

* 예약 대기 신청
* 예약 대기 상세
* 빈자리 제안
* 빈자리 수락 및 거절

## 3차

* 현장 대기 등록
* 현장 대기 상세
* 고객 호출 화면
* 체크인

## 4차

* 운영자 오늘 홈
* 예약 캘린더
* 예약 상세
* 현장 대기열
* 고객 호출
* 서비스 시작 및 완료

## 5차

* 서비스 관리
* 직원 관리
* 근무 일정
* 매장 정책
* 알림함
* 실시간 갱신

---

# 16. 프론트엔드 핵심 주의사항

1. 예약 가능 시간 조회 결과를 예약 성공으로 간주하지 않는다.
2. 쓰기 요청 버튼은 처리 중 중복 클릭을 막는다.
3. 멱등성 키는 요청 재시도 여부를 고려해 관리한다.
4. 상태 충돌 오류가 발생하면 최신 상세를 다시 조회한다.
5. 빈자리 제안 타이머가 끝나면 서버 상태를 다시 확인한다.
6. 실시간 이벤트는 알림 용도로 사용하고 상세 API를 진실 공급원으로 사용한다.
7. 내부 상태 코드를 사용자에게 그대로 노출하지 않는다.
8. 요청 성공 여부가 불명확할 때 새 요청을 만들기 전에 기존 결과를 확인한다.
9. 운영자 화면의 버튼 노출과 서버 권한 검증은 별개다.
10. 예약·대기·현장 상태 매핑은 공통 상수로 관리한다.
