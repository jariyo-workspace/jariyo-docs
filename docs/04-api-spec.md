# 자리요 API Specification

## 1. 문서 개요

본 문서는 자리요의 고객용 서비스, 매장 운영자용 서비스, 비동기 처리 및 운영 기능에 필요한 API를 정의한다.

자리요는 다음 핵심 흐름을 제공한다.

* 고객의 매장·서비스·직원 조회
* 예약 가능 시간 조회
* 예약 생성·확정·취소
* 예약 대기 신청
* 취소 자리 제안과 수락
* 현장 대기 등록
* 고객 호출과 체크인
* 서비스 시작과 완료
* 매장 운영 정보 관리
* 실패 작업 조회와 재처리

API는 단순한 데이터 CRUD보다 사용자 행동과 도메인 상태 전이를 중심으로 설계한다.

## 1.1 MVP 우선순위 표기

이 문서에서는 섹션 제목에 다음 태그를 붙여 MVP 우선순위를 구분한다.

* `[MVP-P0]` MVP 필수
* `[MVP-P1]` MVP 후속, 필수 흐름을 보강
* `[MVP-P2]` MVP 이후

---

# 2. 기본 규칙

## 2.1 Base URL

```text
/api/v1
```

예:

```text
GET /api/v1/stores
POST /api/v1/reservations
```

---

## 2.2 통신 형식

요청과 응답 본문은 JSON을 사용한다.

```http
Content-Type: application/json
Accept: application/json
```

파일 업로드가 필요한 경우에만 별도 형식을 사용한다.

---

## 2.3 날짜와 시간

날짜:

```text
YYYY-MM-DD
```

예:

```text
2026-07-18
```

시간:

```text
HH:mm
```

예:

```text
14:30
```

일시:

```text
ISO 8601 + timezone offset
```

예:

```text
2026-07-18T14:30:00+09:00
```

서버는 모든 일시 응답에 시간대 정보를 포함해야 한다.

---

## 2.4 식별자

모든 리소스 식별자는 문자열로 표현한다.

```json
{
  "id": "res_01JABCD1234"
}
```

클라이언트는 식별자의 내부 구조를 해석하면 안 된다.

---

## 2.5 인증

인증이 필요한 요청은 다음 헤더를 사용한다.

```http
Authorization: Bearer {accessToken}
```

인증되지 않은 사용자는 공개 매장 조회와 일부 비회원 현장 대기 기능만 이용할 수 있다.

---

## 2.6 사용자 역할

```text
CUSTOMER
STAFF
MANAGER
OWNER
SYSTEM
```

권한 관계:

```text
OWNER > MANAGER > STAFF
```

고객과 매장 멤버 역할은 별도로 판단한다.

한 사용자가 고객이면서 특정 매장의 직원일 수 있다.

---

## 2.7 멀티테넌트 규칙

매장 운영자는 자신이 소속된 매장의 데이터만 조회하거나 변경할 수 있다.

다음과 같은 요청은 허용하지 않는다.

```text
A 매장 직원이 B 매장 예약 조회
A 매장 관리자가 B 매장 정책 변경
A 매장 직원이 B 매장 고객 호출
```

매장 ID는 URL에 포함되더라도 서버가 사용자의 소속과 권한을 반드시 검증해야 한다.

---

## 2.8 멱등성

중복 요청이 데이터 중복 생성으로 이어질 수 있는 API는 다음 헤더를 요구한다.

```http
Idempotency-Key: {uniqueKey}
```

적용 대상:

```text
예약 생성
예약 확정
예약 취소
빈자리 제안 수락
현장 대기 등록
체크인
서비스 시작
서비스 완료
고객 호출
```

같은 사용자와 같은 API에서 동일한 키가 재사용되면 기존 결과를 반환한다.

같은 키에 다른 요청 본문이 들어오면 오류를 반환한다.

```text
IDEMPOTENCY_KEY_REUSED_WITH_DIFFERENT_REQUEST
```

---

## 2.9 공통 응답 구조

단일 리소스 응답:

```json
{
  "data": {
    "id": "res_123"
  }
}
```

목록 응답:

```json
{
  "data": [
    {
      "id": "res_123"
    }
  ],
  "page": {
    "cursor": "next_cursor",
    "hasNext": true
  }
}
```

메타데이터가 필요한 경우:

```json
{
  "data": [],
  "meta": {
    "total": 120
  },
  "page": {
    "cursor": "next_cursor",
    "hasNext": true
  }
}
```

---

## 2.10 오류 응답

```json
{
  "error": {
    "code": "RESERVATION_SLOT_ALREADY_TAKEN",
    "message": "방금 다른 고객이 이 시간을 예약했어요.",
    "details": {
      "staffId": "staff_123",
      "startAt": "2026-07-18T14:00:00+09:00"
    },
    "requestId": "req_123"
  }
}
```

### 공통 HTTP 상태 코드

| 상태 코드 | 의미              |
| ----: | --------------- |
|   200 | 요청 성공           |
|   201 | 리소스 생성 성공       |
|   202 | 비동기 요청 접수       |
|   204 | 본문 없는 성공        |
|   400 | 요청 형식 또는 값 오류   |
|   401 | 인증 필요           |
|   403 | 권한 없음           |
|   404 | 리소스 없음          |
|   409 | 상태 충돌 또는 동시성 충돌 |
|   410 | 이미 만료된 리소스      |
|   422 | 도메인 규칙 위반       |
|   429 | 요청 제한 초과        |
|   500 | 서버 내부 오류        |
|   503 | 일시적인 서비스 사용 불가  |

---

# 3. 공통 데이터 타입

## 3.1 사용자 요약

```json
{
  "id": "usr_123",
  "displayName": "류승엽",
  "phoneNumber": "010-****-1234"
}
```

---

## 3.2 매장 요약

```json
{
  "id": "store_123",
  "name": "자리요 헤어",
  "status": "ACTIVE",
  "address": "대구광역시 ..."
}
```

---

## 3.3 직원 요약

```json
{
  "id": "member_123",
  "displayName": "민지",
  "role": "STAFF",
  "bookingEnabled": true
}
```

---

## 3.4 서비스 요약

```json
{
  "id": "service_123",
  "name": "커트",
  "durationMinutes": 30,
  "cleanupMinutes": 10,
  "status": "ACTIVE"
}
```

---

# 4. 인증 API

## [MVP-P0] 4.1 회원가입

```http
POST /api/v1/auth/sign-up
```

### 요청

```json
{
  "email": "user@example.com",
  "password": "password",
  "displayName": "류승엽",
  "phoneNumber": "01012345678",
  "agreements": {
    "terms": true,
    "privacy": true,
    "marketing": false
  }
}
```

### 응답

```json
{
  "data": {
    "userId": "usr_123",
    "accessToken": "token",
    "refreshToken": "refresh_token"
  }
}
```

### 오류

```text
EMAIL_ALREADY_EXISTS
PHONE_NUMBER_ALREADY_EXISTS
INVALID_PASSWORD_FORMAT
REQUIRED_AGREEMENT_MISSING
```

---

## [MVP-P0] 4.2 로그인

```http
POST /api/v1/auth/sign-in
```

### 요청

```json
{
  "email": "user@example.com",
  "password": "password"
}
```

### 응답

```json
{
  "data": {
    "accessToken": "token",
    "refreshToken": "refresh_token",
    "expiresIn": 3600
  }
}
```

---

## [MVP-P0] 4.3 토큰 재발급

```http
POST /api/v1/auth/refresh
```

### 요청

```json
{
  "refreshToken": "refresh_token"
}
```

---

## [MVP-P0] 4.4 로그아웃

```http
POST /api/v1/auth/sign-out
```

---

## [MVP-P0] 4.5 내 프로필 조회

```http
GET /api/v1/me
```

### 응답

```json
{
  "data": {
    "id": "usr_123",
    "email": "user@example.com",
    "displayName": "류승엽",
    "phoneNumber": "010-****-1234",
    "customerProfile": {
      "notificationConsent": true,
      "marketingConsent": false
    },
    "storeMemberships": [
      {
        "storeId": "store_123",
        "role": "STAFF",
        "status": "ACTIVE"
      }
    ]
  }
}
```

---

# 5. 고객용 매장 조회 API

## [MVP-P0] 5.1 매장 목록 조회

```http
GET /api/v1/stores
```

### 쿼리

```text
query
category
region
openNow
availableToday
cursor
limit
```

### 예시

```http
GET /api/v1/stores?query=헤어&openNow=true&limit=20
```

### 응답

```json
{
  "data": [
    {
      "id": "store_123",
      "name": "자리요 헤어",
      "description": "예약과 현장 대기가 가능한 헤어숍",
      "status": "ACTIVE",
      "address": "대구광역시 ...",
      "openNow": true,
      "todayBusinessHours": {
        "openTime": "10:00",
        "closeTime": "20:00"
      },
      "nextAvailableAt": "2026-07-18T14:00:00+09:00",
      "walkInEnabled": true,
      "waitlistEnabled": true
    }
  ],
  "page": {
    "cursor": "next_cursor",
    "hasNext": false
  }
}
```

---

## [MVP-P0] 5.2 매장 상세 조회

```http
GET /api/v1/stores/{storeId}
```

### 응답

```json
{
  "data": {
    "id": "store_123",
    "name": "자리요 헤어",
    "description": "예약과 현장 대기가 가능한 헤어숍",
    "phoneNumber": "053-123-4567",
    "address": "대구광역시 ...",
    "timezone": "Asia/Seoul",
    "status": "ACTIVE",
    "businessHours": [
      {
        "dayOfWeek": "MONDAY",
        "periods": [
          {
            "openTime": "10:00",
            "closeTime": "20:00"
          }
        ]
      }
    ],
    "policySummary": {
      "bookingOpenDays": 30,
      "minimumBookingNoticeMinutes": 120,
      "cancellationDeadlineMinutes": 1440,
      "waitlistEnabled": true,
      "walkInEnabled": true
    }
  }
}
```

---

## [MVP-P0] 5.3 매장 서비스 목록 조회

```http
GET /api/v1/stores/{storeId}/services
```

### 쿼리

```text
activeOnly=true
```

### 응답

```json
{
  "data": [
    {
      "id": "service_123",
      "name": "커트",
      "description": "기본 커트 서비스",
      "durationMinutes": 30,
      "cleanupMinutes": 10,
      "capacity": 1,
      "status": "ACTIVE",
      "availableStaffCount": 3
    }
  ]
}
```

---

## [MVP-P0] 5.4 서비스 담당 직원 조회

```http
GET /api/v1/stores/{storeId}/services/{serviceId}/staff
```

### 쿼리

```text
date
```

### 응답

```json
{
  "data": [
    {
      "id": "member_123",
      "displayName": "민지",
      "bookingEnabled": true,
      "customDurationMinutes": 40,
      "nextAvailableAt": "2026-07-18T14:00:00+09:00"
    }
  ]
}
```

---

# 6. 예약 가능 시간 API

## [MVP-P0] 6.1 예약 가능 시간 조회

```http
GET /api/v1/stores/{storeId}/availability
```

### 쿼리

```text
serviceId
staffId
from
to
partySize
```

`staffId`는 선택 사항이다.

### 예시

```http
GET /api/v1/stores/store_123/availability
  ?serviceId=service_123
  &staffId=member_123
  &from=2026-07-18
  &to=2026-07-20
  &partySize=1
```

### 응답

```json
{
  "data": {
    "storeId": "store_123",
    "serviceId": "service_123",
    "staffId": "member_123",
    "dates": [
      {
        "date": "2026-07-18",
        "slots": [
          {
            "startAt": "2026-07-18T14:00:00+09:00",
            "serviceEndAt": "2026-07-18T14:30:00+09:00",
            "occupiedUntil": "2026-07-18T14:40:00+09:00",
            "staffId": "member_123",
            "status": "AVAILABLE"
          }
        ]
      }
    ]
  }
}
```

### 상태

```text
AVAILABLE
UNAVAILABLE
WAITLIST_AVAILABLE
```

### 주의

이 API의 결과는 조회 시점의 정보다.

예약 생성 시 서버는 실제 충돌 여부를 다시 확인해야 한다.

---

# 7. 고객 예약 API

## [MVP-P0] 7.1 예약 생성

```http
POST /api/v1/reservations
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

### 응답

```json
{
  "data": {
    "id": "res_123",
    "store": {
      "id": "store_123",
      "name": "자리요 헤어"
    },
    "service": {
      "id": "service_123",
      "name": "커트"
    },
    "staff": {
      "id": "member_123",
      "displayName": "민지"
    },
    "source": "CUSTOMER_BOOKING",
    "status": "CONFIRMED",
    "startAt": "2026-07-18T14:00:00+09:00",
    "serviceEndAt": "2026-07-18T14:30:00+09:00",
    "occupiedUntil": "2026-07-18T14:40:00+09:00",
    "createdAt": "2026-07-11T10:00:00+09:00"
  }
}
```

### 오류

```text
STORE_NOT_ACTIVE
SERVICE_NOT_ACTIVE
STAFF_NOT_AVAILABLE
RESERVATION_SLOT_ALREADY_TAKEN
RESERVATION_OUTSIDE_BOOKING_WINDOW
RESERVATION_TOO_CLOSE_TO_START
CUSTOMER_HAS_OVERLAPPING_RESERVATION
INVALID_PARTY_SIZE
```

---

## [MVP-P1] 7.2 예약 홀드 생성

결제나 추가 확인 단계가 필요한 경우 사용할 수 있다.

```http
POST /api/v1/reservation-holds
Idempotency-Key: {key}
```

### 요청

```json
{
  "storeId": "store_123",
  "serviceId": "service_123",
  "staffId": "member_123",
  "startAt": "2026-07-18T14:00:00+09:00",
  "partySize": 1
}
```

### 응답

```json
{
  "data": {
    "reservationId": "res_123",
    "status": "HELD",
    "holdExpiresAt": "2026-07-11T10:05:00+09:00"
  }
}
```

---

## [MVP-P1] 7.3 예약 홀드 확정

```http
POST /api/v1/reservations/{reservationId}/confirm
Idempotency-Key: {key}
```

### 응답

```json
{
  "data": {
    "id": "res_123",
    "status": "CONFIRMED",
    "confirmedAt": "2026-07-11T10:03:00+09:00"
  }
}
```

### 오류

```text
RESERVATION_HOLD_EXPIRED
RESERVATION_INVALID_STATE
RESERVATION_SLOT_ALREADY_TAKEN
```

---

## [MVP-P0] 7.4 내 예약 목록 조회

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

### 예시

```http
GET /api/v1/me/reservations?status=CONFIRMED&from=2026-07-01
```

### 응답

```json
{
  "data": [
    {
      "id": "res_123",
      "store": {
        "id": "store_123",
        "name": "자리요 헤어"
      },
      "service": {
        "id": "service_123",
        "name": "커트"
      },
      "staff": {
        "id": "member_123",
        "displayName": "민지"
      },
      "status": "CONFIRMED",
      "startAt": "2026-07-18T14:00:00+09:00",
      "serviceEndAt": "2026-07-18T14:30:00+09:00"
    }
  ]
}
```

---

## [MVP-P0] 7.5 예약 상세 조회

```http
GET /api/v1/reservations/{reservationId}
```

### 응답

```json
{
  "data": {
    "id": "res_123",
    "store": {
      "id": "store_123",
      "name": "자리요 헤어",
      "phoneNumber": "053-123-4567",
      "address": "대구광역시 ..."
    },
    "service": {
      "id": "service_123",
      "name": "커트"
    },
    "staff": {
      "id": "member_123",
      "displayName": "민지"
    },
    "source": "CUSTOMER_BOOKING",
    "status": "CONFIRMED",
    "startAt": "2026-07-18T14:00:00+09:00",
    "serviceEndAt": "2026-07-18T14:30:00+09:00",
    "occupiedUntil": "2026-07-18T14:40:00+09:00",
    "partySize": 1,
    "customerNote": "앞머리는 짧지 않게 부탁드립니다.",
    "checkInAvailable": false,
    "canCancel": true,
    "cancellationDeadlineAt": "2026-07-17T14:00:00+09:00",
    "createdAt": "2026-07-11T10:00:00+09:00"
  }
}
```

---

## [MVP-P0] 7.6 예약 취소

```http
POST /api/v1/reservations/{reservationId}/cancel
Idempotency-Key: {key}
```

### 요청

```json
{
  "reasonCode": "CUSTOMER_SCHEDULE_CHANGED",
  "reason": "개인 일정이 생겼습니다."
}
```

### 응답

```json
{
  "data": {
    "id": "res_123",
    "status": "CANCELLED",
    "cancelledAt": "2026-07-15T13:00:00+09:00",
    "cancelledByType": "CUSTOMER"
  }
}
```

### 오류

```text
RESERVATION_ALREADY_CANCELLED
RESERVATION_CANCELLATION_DEADLINE_PASSED
RESERVATION_INVALID_STATE
RESERVATION_NOT_OWNED_BY_USER
```

---

## [MVP-P1] 7.7 예약 상태 이력 조회

```http
GET /api/v1/reservations/{reservationId}/history
```

### 응답

```json
{
  "data": [
    {
      "previousStatus": null,
      "nextStatus": "CONFIRMED",
      "changedByType": "CUSTOMER",
      "reasonCode": "CREATED",
      "occurredAt": "2026-07-11T10:00:00+09:00"
    }
  ]
}
```

---

# 8. 예약 대기 API

## [MVP-P1] 8.1 예약 대기 신청

```http
POST /api/v1/waitlists
Idempotency-Key: {key}
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

### 응답

```json
{
  "data": {
    "id": "wait_123",
    "storeId": "store_123",
    "serviceId": "service_123",
    "preferredStaffId": "member_123",
    "staffPreferenceType": "SPECIFIC_PREFERRED",
    "desiredDate": "2026-07-18",
    "acceptableStartTime": "13:00",
    "acceptableEndTime": "17:00",
    "status": "WAITING",
    "sequenceNumber": 15,
    "createdAt": "2026-07-11T10:00:00+09:00"
  }
}
```

### 오류

```text
WAITLIST_NOT_ENABLED
WAITLIST_DUPLICATED
WAITLIST_DATE_OUT_OF_RANGE
INVALID_WAITLIST_TIME_RANGE
SERVICE_NOT_ACTIVE
STAFF_NOT_AVAILABLE
```

---

## [MVP-P1] 8.2 내 예약 대기 목록 조회

```http
GET /api/v1/me/waitlists
```

### 쿼리

```text
status
from
to
cursor
limit
```

---

## [MVP-P1] 8.3 예약 대기 상세 조회

```http
GET /api/v1/waitlists/{waitlistId}
```

### 응답

```json
{
  "data": {
    "id": "wait_123",
    "store": {
      "id": "store_123",
      "name": "자리요 헤어"
    },
    "service": {
      "id": "service_123",
      "name": "커트"
    },
    "preferredStaff": {
      "id": "member_123",
      "displayName": "민지"
    },
    "staffPreferenceType": "SPECIFIC_PREFERRED",
    "desiredDate": "2026-07-18",
    "acceptableStartTime": "13:00",
    "acceptableEndTime": "17:00",
    "status": "WAITING",
    "sequenceNumber": 15,
    "activeOffer": null,
    "createdAt": "2026-07-11T10:00:00+09:00"
  }
}
```

---

## [MVP-P2] 8.4 예약 대기 조건 변경

```http
PUT /api/v1/waitlists/{waitlistId}
```

### 요청

```json
{
  "preferredStaffId": null,
  "staffPreferenceType": "ANY_STAFF",
  "acceptableStartTime": "12:00",
  "acceptableEndTime": "18:00"
}
```

### 조건

다음 상태에서만 변경 가능하다.

```text
WAITING
```

### 오류

```text
WAITLIST_NOT_EDITABLE
WAITLIST_NOT_OWNED_BY_USER
INVALID_WAITLIST_TIME_RANGE
```

---

## [MVP-P1] 8.5 예약 대기 취소

```http
POST /api/v1/waitlists/{waitlistId}/cancel
Idempotency-Key: {key}
```

### 응답

```json
{
  "data": {
    "id": "wait_123",
    "status": "CANCELLED",
    "cancelledAt": "2026-07-12T12:00:00+09:00"
  }
}
```

---

# 9. 빈자리 제안 API

## [MVP-P1] 9.1 내 활성 빈자리 제안 목록

```http
GET /api/v1/me/slot-offers
```

### 쿼리

```text
status=PENDING
```

---

## [MVP-P1] 9.2 빈자리 제안 상세 조회

```http
GET /api/v1/slot-offers/{offerId}
```

### 응답

```json
{
  "data": {
    "id": "offer_123",
    "waitlistId": "wait_123",
    "store": {
      "id": "store_123",
      "name": "자리요 헤어"
    },
    "service": {
      "id": "service_123",
      "name": "커트"
    },
    "staff": {
      "id": "member_123",
      "displayName": "민지"
    },
    "startAt": "2026-07-18T15:00:00+09:00",
    "serviceEndAt": "2026-07-18T15:30:00+09:00",
    "status": "PENDING",
    "expiresAt": "2026-07-11T10:03:00+09:00",
    "remainingSeconds": 125
  }
}
```

---

## [MVP-P1] 9.3 빈자리 제안 수락

```http
POST /api/v1/slot-offers/{offerId}/accept
Idempotency-Key: {key}
```

### 응답

```json
{
  "data": {
    "offer": {
      "id": "offer_123",
      "status": "ACCEPTED",
      "acceptedAt": "2026-07-11T10:02:00+09:00"
    },
    "reservation": {
      "id": "res_456",
      "source": "WAITLIST_OFFER",
      "status": "CONFIRMED",
      "startAt": "2026-07-18T15:00:00+09:00"
    },
    "waitlist": {
      "id": "wait_123",
      "status": "RESERVED"
    }
  }
}
```

### 하나의 트랜잭션으로 보장할 변경

```text
SlotOffer PENDING → ACCEPTED
WaitlistEntry OFFERED → RESERVED
Reservation 생성
만료 작업 취소
상태 이력 기록
후속 이벤트 기록
```

### 오류

```text
SLOT_OFFER_EXPIRED
SLOT_OFFER_ALREADY_ACCEPTED
SLOT_OFFER_ALREADY_DECLINED
SLOT_OFFER_NO_LONGER_AVAILABLE
WAITLIST_INVALID_STATE
RESERVATION_SLOT_ALREADY_TAKEN
```

---

## [MVP-P2] 9.4 빈자리 제안 거절

```http
POST /api/v1/slot-offers/{offerId}/decline
Idempotency-Key: {key}
```

### 요청

```json
{
  "keepWaitlistActive": true
}
```

### 응답

```json
{
  "data": {
    "offer": {
      "id": "offer_123",
      "status": "DECLINED"
    },
    "waitlist": {
      "id": "wait_123",
      "status": "WAITING"
    }
  }
}
```

---

# 10. 현장 대기 API

## [MVP-P1] 10.1 현장 대기 정보 조회

```http
GET /api/v1/stores/{storeId}/walk-in-status
```

### 쿼리

```text
serviceId
staffId
```

### 응답

```json
{
  "data": {
    "storeId": "store_123",
    "walkInEnabled": true,
    "acceptingEntries": true,
    "waitingCount": 5,
    "estimatedWaitMinutes": 35,
    "lastEntryAt": "19:00"
  }
}
```

---

## [MVP-P1] 10.2 현장 대기 등록

```http
POST /api/v1/walk-ins
Idempotency-Key: {key}
```

### 로그인 고객 요청

```json
{
  "storeId": "store_123",
  "serviceId": "service_123",
  "preferredStaffId": null,
  "partySize": 1
}
```

### 비회원 요청

```json
{
  "storeId": "store_123",
  "serviceId": "service_123",
  "preferredStaffId": null,
  "partySize": 1,
  "guest": {
    "name": "류승엽",
    "phoneNumber": "01012345678"
  }
}
```

### 응답

```json
{
  "data": {
    "id": "walk_123",
    "storeId": "store_123",
    "serviceId": "service_123",
    "queueNumber": 12,
    "status": "WAITING",
    "waitingAhead": 4,
    "estimatedWaitMinutes": 30,
    "createdAt": "2026-07-11T14:00:00+09:00"
  }
}
```

### 오류

```text
WALK_IN_NOT_ENABLED
WALK_IN_REGISTRATION_CLOSED
WALK_IN_ALREADY_REGISTERED
SERVICE_NOT_ACTIVE
INVALID_PARTY_SIZE
```

---

## [MVP-P1] 10.3 내 현장 대기 목록 조회

```http
GET /api/v1/me/walk-ins
```

### 쿼리

```text
status
date
```

---

## [MVP-P1] 10.4 현장 대기 상세 조회

```http
GET /api/v1/walk-ins/{walkInId}
```

### 응답

```json
{
  "data": {
    "id": "walk_123",
    "store": {
      "id": "store_123",
      "name": "자리요 헤어"
    },
    "service": {
      "id": "service_123",
      "name": "커트"
    },
    "queueNumber": 12,
    "status": "WAITING",
    "waitingAhead": 4,
    "estimatedWaitMinutes": 30,
    "calledAt": null,
    "callExpiresAt": null,
    "createdAt": "2026-07-11T14:00:00+09:00"
  }
}
```

---

## [MVP-P1] 10.5 현장 대기 취소

```http
POST /api/v1/walk-ins/{walkInId}/cancel
Idempotency-Key: {key}
```

### 요청

```json
{
  "reason": "다른 일정이 생겼습니다."
}
```

---

## [MVP-P1] 10.6 호출 응답

```http
POST /api/v1/walk-ins/{walkInId}/respond-call
Idempotency-Key: {key}
```

### 요청

```json
{
  "response": "ENTERING"
}
```

### response

```text
ENTERING
DELAYED
CANCEL
```

### 응답

```json
{
  "data": {
    "id": "walk_123",
    "status": "CHECKED_IN"
  }
}
```

---

# 11. 체크인 API

## [MVP-P1] 11.1 예약 체크인 가능 여부 조회

```http
GET /api/v1/reservations/{reservationId}/check-in-availability
```

### 응답

```json
{
  "data": {
    "available": true,
    "availableFrom": "2026-07-18T13:40:00+09:00",
    "availableUntil": "2026-07-18T14:10:00+09:00",
    "alreadyCheckedIn": false
  }
}
```

---

## [MVP-P1] 11.2 예약 체크인

```http
POST /api/v1/reservations/{reservationId}/check-in
Idempotency-Key: {key}
```

### 요청

```json
{
  "method": "CUSTOMER_APP"
}
```

### 응답

```json
{
  "data": {
    "checkInId": "checkin_123",
    "reservationId": "res_123",
    "status": "VALID",
    "checkedInAt": "2026-07-18T13:50:00+09:00"
  }
}
```

### 오류

```text
CHECK_IN_NOT_AVAILABLE_YET
CHECK_IN_WINDOW_EXPIRED
CHECK_IN_ALREADY_COMPLETED
RESERVATION_INVALID_STATE
```

---

## [MVP-P2] 11.3 QR 체크인

```http
POST /api/v1/check-ins/qr
Idempotency-Key: {key}
```

### 요청

```json
{
  "token": "raw_qr_token"
}
```

### 오류

```text
CHECK_IN_TOKEN_INVALID
CHECK_IN_TOKEN_EXPIRED
CHECK_IN_TOKEN_ALREADY_USED
```

---

# 12. 고객 통합 홈 API

## [MVP-P2] 12.1 고객 홈 상태 조회

```http
GET /api/v1/me/home
```

### 응답

```json
{
  "data": {
    "priorityAction": {
      "type": "SLOT_OFFER",
      "referenceId": "offer_123",
      "title": "원하던 시간에 빈자리가 생겼어요",
      "expiresAt": "2026-07-11T10:03:00+09:00"
    },
    "upcomingReservations": [
      {
        "id": "res_123",
        "storeName": "자리요 헤어",
        "startAt": "2026-07-18T14:00:00+09:00",
        "status": "CONFIRMED"
      }
    ],
    "activeWaitlists": [],
    "activeWalkIns": [],
    "recentStores": []
  }
}
```

---

# 13. 알림 API

## [MVP-P2] 13.1 내 알림 목록

```http
GET /api/v1/me/notifications
```

### 쿼리

```text
read
type
cursor
limit
```

### 응답

```json
{
  "data": [
    {
      "id": "noti_123",
      "type": "SLOT_OFFER_CREATED",
      "title": "원하던 시간에 빈자리가 생겼어요",
      "body": "오늘 오후 3시 30분 예약이 가능합니다.",
      "referenceType": "SLOT_OFFER",
      "referenceId": "offer_123",
      "read": false,
      "createdAt": "2026-07-11T10:00:00+09:00"
    }
  ]
}
```

---

## [MVP-P2] 13.2 알림 읽음 처리

```http
POST /api/v1/notifications/{notificationId}/read
```

---

## [MVP-P2] 13.3 전체 알림 읽음 처리

```http
POST /api/v1/me/notifications/read-all
```

---

## [MVP-P2] 13.4 알림 설정 조회

```http
GET /api/v1/me/notification-settings
```

---

## [MVP-P2] 13.5 알림 설정 변경

```http
PUT /api/v1/me/notification-settings
```

### 요청

```json
{
  "reservationNotifications": true,
  "waitlistNotifications": true,
  "walkInNotifications": true,
  "marketingNotifications": false
}
```

---

# 14. 운영자 매장 홈 API

## [MVP-P1] 14.1 오늘 운영 현황 조회

```http
GET /api/v1/admin/stores/{storeId}/dashboard/today
```

### 응답

```json
{
  "data": {
    "date": "2026-07-11",
    "summary": {
      "reservationCount": 24,
      "waitingWalkInCount": 5,
      "checkedInCount": 3,
      "inServiceCount": 2,
      "noShowCandidateCount": 1,
      "pendingSlotOfferCount": 2
    },
    "alerts": [
      {
        "type": "NO_SHOW_CANDIDATE",
        "referenceId": "res_123",
        "message": "예약 시간이 15분 지났지만 체크인되지 않았어요."
      }
    ],
    "timeline": []
  }
}
```

---

# 15. 운영자 예약 관리 API

## [MVP-P1] 15.1 매장 예약 목록 조회

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
customerQuery
cursor
limit
```

---

## [MVP-P1] 15.2 운영자 예약 상세 조회

```http
GET /api/v1/admin/stores/{storeId}/reservations/{reservationId}
```

---

## [MVP-P2] 15.3 수동 예약 생성

```http
POST /api/v1/admin/stores/{storeId}/reservations
Idempotency-Key: {key}
```

### 요청

```json
{
  "customer": {
    "userId": "usr_123"
  },
  "serviceId": "service_123",
  "staffId": "member_123",
  "startAt": "2026-07-18T14:00:00+09:00",
  "partySize": 1,
  "storeNote": "전화 예약",
  "sendNotification": true
}
```

비회원 고객:

```json
{
  "customer": {
    "guestName": "홍길동",
    "guestPhoneNumber": "01012345678"
  }
}
```

---

## [MVP-P2] 15.4 예약 시간 변경

```http
POST /api/v1/admin/stores/{storeId}/reservations/{reservationId}/reschedule
Idempotency-Key: {key}
```

### 요청

```json
{
  "staffId": "member_456",
  "startAt": "2026-07-18T15:00:00+09:00",
  "reason": "고객 요청",
  "sendNotification": true
}
```

### 오류

```text
RESERVATION_NOT_RESCHEDULABLE
RESERVATION_SLOT_ALREADY_TAKEN
STAFF_NOT_AVAILABLE
```

---

## [MVP-P2] 15.5 담당 직원 변경

```http
POST /api/v1/admin/stores/{storeId}/reservations/{reservationId}/assign-staff
Idempotency-Key: {key}
```

### 요청

```json
{
  "staffId": "member_456",
  "reason": "근무 일정 변경"
}
```

---

## [MVP-P1] 15.6 운영자 예약 취소

```http
POST /api/v1/admin/stores/{storeId}/reservations/{reservationId}/cancel
Idempotency-Key: {key}
```

### 요청

```json
{
  "reasonCode": "STORE_SCHEDULE_CHANGED",
  "reason": "매장 일정 변경",
  "sendNotification": true
}
```

---

## [MVP-P1] 15.7 노쇼 처리

```http
POST /api/v1/admin/stores/{storeId}/reservations/{reservationId}/mark-no-show
Idempotency-Key: {key}
```

### 요청

```json
{
  "reason": "예약 시간 이후 15분 동안 미방문"
}
```

### 오류

```text
RESERVATION_NOT_NO_SHOW_CANDIDATE
RESERVATION_INVALID_STATE
```

---

## [MVP-P1] 15.8 직원 수동 체크인

```http
POST /api/v1/admin/stores/{storeId}/reservations/{reservationId}/check-in
Idempotency-Key: {key}
```

---

# 16. 운영자 현장 대기 API

## [MVP-P1] 16.1 현장 대기열 조회

```http
GET /api/v1/admin/stores/{storeId}/walk-ins
```

### 쿼리

```text
status
serviceId
staffId
date
```

### 응답

```json
{
  "data": [
    {
      "id": "walk_123",
      "queueNumber": 12,
      "customerName": "류승엽",
      "serviceName": "커트",
      "preferredStaffName": null,
      "status": "WAITING",
      "waitingMinutes": 20,
      "estimatedWaitMinutes": 15
    }
  ]
}
```

---

## [MVP-P1] 16.2 운영자 현장 대기 등록

```http
POST /api/v1/admin/stores/{storeId}/walk-ins
Idempotency-Key: {key}
```

### 요청

```json
{
  "customer": {
    "guestName": "류승엽",
    "guestPhoneNumber": "01012345678"
  },
  "serviceId": "service_123",
  "preferredStaffId": null,
  "partySize": 1
}
```

---

## [MVP-P1] 16.3 고객 호출

```http
POST /api/v1/admin/stores/{storeId}/walk-ins/{walkInId}/call
Idempotency-Key: {key}
```

### 요청

```json
{
  "responseTimeoutMinutes": 3,
  "sendNotification": true
}
```

### 응답

```json
{
  "data": {
    "id": "walk_123",
    "status": "CALLED",
    "calledAt": "2026-07-11T14:20:00+09:00",
    "callExpiresAt": "2026-07-11T14:23:00+09:00"
  }
}
```

### 오류

```text
WALK_IN_INVALID_STATE
WALK_IN_ALREADY_CALLED
```

---

## [MVP-P2] 16.4 재호출

```http
POST /api/v1/admin/stores/{storeId}/walk-ins/{walkInId}/recall
Idempotency-Key: {key}
```

---

## [MVP-P2] 16.5 대기 보류

```http
POST /api/v1/admin/stores/{storeId}/walk-ins/{walkInId}/skip
Idempotency-Key: {key}
```

### 요청

```json
{
  "reason": "고객이 잠시 자리를 비움"
}
```

---

## [MVP-P2] 16.6 대기열 복귀

```http
POST /api/v1/admin/stores/{storeId}/walk-ins/{walkInId}/restore
Idempotency-Key: {key}
```

---

## [MVP-P2] 16.7 현장 고객 노쇼 처리

```http
POST /api/v1/admin/stores/{storeId}/walk-ins/{walkInId}/mark-no-show
Idempotency-Key: {key}
```

---

## [MVP-P2] 16.8 대기 순서 변경

```http
POST /api/v1/admin/stores/{storeId}/walk-ins/{walkInId}/reorder
Idempotency-Key: {key}
```

### 요청

```json
{
  "beforeWalkInId": "walk_456",
  "reasonCode": "CUSTOMER_REQUEST",
  "reason": "고객 요청으로 한 순서 뒤로 이동"
}
```

### 주의

순서 변경은 반드시 감사 로그에 기록한다.

---

# 17. 서비스 진행 API

## [MVP-P1] 17.1 예약 서비스 시작

```http
POST /api/v1/admin/stores/{storeId}/reservations/{reservationId}/start-service
Idempotency-Key: {key}
```

### 요청

```json
{
  "staffId": "member_123"
}
```

### 응답

```json
{
  "data": {
    "serviceSessionId": "session_123",
    "reservationId": "res_123",
    "status": "IN_PROGRESS",
    "actualStartAt": "2026-07-18T14:05:00+09:00"
  }
}
```

---

## [MVP-P1] 17.2 현장 고객 서비스 시작

```http
POST /api/v1/admin/stores/{storeId}/walk-ins/{walkInId}/start-service
Idempotency-Key: {key}
```

---

## [MVP-P1] 17.3 서비스 완료

```http
POST /api/v1/admin/stores/{storeId}/service-sessions/{sessionId}/complete
Idempotency-Key: {key}
```

### 요청

```json
{
  "completionNote": "정상 완료"
}
```

### 응답

```json
{
  "data": {
    "id": "session_123",
    "status": "COMPLETED",
    "actualStartAt": "2026-07-18T14:05:00+09:00",
    "actualEndAt": "2026-07-18T14:37:00+09:00",
    "actualDurationMinutes": 32
  }
}
```

---

## [MVP-P2] 17.4 서비스 세션 취소

```http
POST /api/v1/admin/stores/{storeId}/service-sessions/{sessionId}/cancel
Idempotency-Key: {key}
```

---

# 18. 운영자 예약 대기 API

## [MVP-P1] 18.1 매장 예약 대기 목록 조회

```http
GET /api/v1/admin/stores/{storeId}/waitlists
```

### 쿼리

```text
date
serviceId
staffId
status
sort
cursor
limit
```

### sort

```text
SEQUENCE_ASC
CREATED_AT_ASC
PRIORITY_DESC
```

---

## [MVP-P1] 18.2 예약 대기 상세 조회

```http
GET /api/v1/admin/stores/{storeId}/waitlists/{waitlistId}
```

---

## [MVP-P2] 18.3 수동 빈자리 제안 생성

```http
POST /api/v1/admin/stores/{storeId}/waitlists/{waitlistId}/offers
Idempotency-Key: {key}
```

### 요청

```json
{
  "staffId": "member_123",
  "startAt": "2026-07-18T15:00:00+09:00",
  "expiresInMinutes": 3,
  "sourceReservationId": "res_cancelled_123"
}
```

### 오류

```text
WAITLIST_INVALID_STATE
WAITLIST_CONDITION_NOT_MATCHED
SLOT_OFFER_ALREADY_ACTIVE
RESERVATION_SLOT_ALREADY_TAKEN
```

---

## [MVP-P2] 18.4 빈자리 제안 목록 조회

```http
GET /api/v1/admin/stores/{storeId}/slot-offers
```

### 쿼리

```text
status
date
customerQuery
```

---

## [MVP-P2] 18.5 빈자리 제안 취소

```http
POST /api/v1/admin/stores/{storeId}/slot-offers/{offerId}/revoke
Idempotency-Key: {key}
```

### 요청

```json
{
  "reason": "슬롯이 더 이상 제공 가능하지 않음"
}
```

---

## [MVP-P2] 18.6 다음 대기자에게 제안

```http
POST /api/v1/admin/stores/{storeId}/slot-offers/{offerId}/offer-next
Idempotency-Key: {key}
```

---

# 19. 매장 서비스 관리 API

## 19.1 서비스 목록 조회

```http
GET /api/v1/admin/stores/{storeId}/services
```

---

## 19.2 서비스 생성

```http
POST /api/v1/admin/stores/{storeId}/services
```

### 요청

```json
{
  "name": "커트",
  "description": "기본 커트",
  "durationMinutes": 30,
  "cleanupMinutes": 10,
  "capacity": 1
}
```

---

## 19.3 서비스 상세 조회

```http
GET /api/v1/admin/stores/{storeId}/services/{serviceId}
```

---

## 19.4 서비스 수정

```http
PUT /api/v1/admin/stores/{storeId}/services/{serviceId}
```

### 요청

```json
{
  "name": "커트",
  "description": "기본 커트 서비스",
  "durationMinutes": 40,
  "cleanupMinutes": 10,
  "capacity": 1
}
```

---

## 19.5 서비스 비활성화

```http
POST /api/v1/admin/stores/{storeId}/services/{serviceId}/deactivate
```

---

## 19.6 서비스 활성화

```http
POST /api/v1/admin/stores/{storeId}/services/{serviceId}/activate
```

---

# 20. 직원 관리 API

## 20.1 직원 목록 조회

```http
GET /api/v1/admin/stores/{storeId}/staff
```

### 쿼리

```text
status
role
bookingEnabled
```

---

## 20.2 직원 초대

```http
POST /api/v1/admin/stores/{storeId}/staff/invitations
```

### 요청

```json
{
  "email": "staff@example.com",
  "role": "STAFF",
  "displayName": "민지"
}
```

---

## 20.3 직원 상세 조회

```http
GET /api/v1/admin/stores/{storeId}/staff/{staffId}
```

---

## 20.4 직원 정보 수정

```http
PUT /api/v1/admin/stores/{storeId}/staff/{staffId}
```

### 요청

```json
{
  "displayName": "민지",
  "role": "STAFF",
  "bookingEnabled": true,
  "status": "ACTIVE"
}
```

---

## 20.5 직원 담당 서비스 설정

```http
PUT /api/v1/admin/stores/{storeId}/staff/{staffId}/services
```

### 요청

```json
{
  "services": [
    {
      "serviceId": "service_123",
      "active": true,
      "customDurationMinutes": 40
    }
  ]
}
```

---

## 20.6 직원 비활성화

```http
POST /api/v1/admin/stores/{storeId}/staff/{staffId}/deactivate
```

### 오류

```text
STAFF_HAS_FUTURE_RESERVATIONS
STAFF_ALREADY_INACTIVE
```

---

# 21. 근무 일정 API

## 21.1 직원 반복 근무시간 조회

```http
GET /api/v1/admin/stores/{storeId}/staff/{staffId}/schedules
```

---

## 21.2 직원 반복 근무시간 설정

```http
PUT /api/v1/admin/stores/{storeId}/staff/{staffId}/schedules
```

### 요청

```json
{
  "schedules": [
    {
      "dayOfWeek": "MONDAY",
      "periods": [
        {
          "startTime": "09:00",
          "endTime": "12:00"
        },
        {
          "startTime": "13:00",
          "endTime": "18:00"
        }
      ],
      "validFrom": "2026-07-01",
      "validUntil": null
    }
  ]
}
```

### 응답

충돌 예약이 있는 경우 바로 변경하지 않고 경고 응답을 줄 수 있다.

```json
{
  "data": {
    "updated": false,
    "conflicts": [
      {
        "reservationId": "res_123",
        "startAt": "2026-07-18T17:30:00+09:00"
      }
    ]
  }
}
```

---

## 21.3 직원 일정 예외 목록

```http
GET /api/v1/admin/stores/{storeId}/staff/{staffId}/schedule-exceptions
```

---

## 21.4 직원 일정 예외 생성

```http
POST /api/v1/admin/stores/{storeId}/staff/{staffId}/schedule-exceptions
```

### 요청

```json
{
  "targetDate": "2026-07-20",
  "type": "DAY_OFF",
  "reason": "개인 휴가"
}
```

---

## 21.5 직원 일정 예외 삭제

```http
DELETE /api/v1/admin/stores/{storeId}/staff/{staffId}/schedule-exceptions/{exceptionId}
```

---

# 22. 매장 영업시간과 예외 일정 API

## 22.1 매장 영업시간 조회

```http
GET /api/v1/admin/stores/{storeId}/business-hours
```

---

## 22.2 매장 영업시간 설정

```http
PUT /api/v1/admin/stores/{storeId}/business-hours
```

### 요청

```json
{
  "businessHours": [
    {
      "dayOfWeek": "MONDAY",
      "isClosed": false,
      "periods": [
        {
          "openTime": "10:00",
          "closeTime": "20:00"
        }
      ]
    }
  ]
}
```

---

## 22.3 매장 일정 예외 조회

```http
GET /api/v1/admin/stores/{storeId}/schedule-exceptions
```

---

## 22.4 매장 일정 예외 생성

```http
POST /api/v1/admin/stores/{storeId}/schedule-exceptions
```

### 요청

```json
{
  "targetDate": "2026-08-15",
  "type": "CLOSED_ALL_DAY",
  "reason": "임시 휴무"
}
```

---

## 22.5 매장 일정 예외 삭제

```http
DELETE /api/v1/admin/stores/{storeId}/schedule-exceptions/{exceptionId}
```

---

# 23. 매장 정책 API

## 23.1 매장 정책 조회

```http
GET /api/v1/admin/stores/{storeId}/policy
```

---

## 23.2 매장 정책 수정

```http
PUT /api/v1/admin/stores/{storeId}/policy
```

### 요청

```json
{
  "bookingOpenDays": 30,
  "minimumBookingNoticeMinutes": 120,
  "cancellationDeadlineMinutes": 1440,
  "checkInOpenBeforeMinutes": 20,
  "lateToleranceMinutes": 10,
  "noShowAfterMinutes": 15,
  "reservationHoldMinutes": 5,
  "slotOfferExpirationMinutes": 3,
  "walkInCallTimeoutMinutes": 3,
  "waitlistEnabled": true,
  "walkInEnabled": true,
  "autoNoShowEnabled": false
}
```

---

# 24. 매장 정보 API

## 24.1 매장 정보 조회

```http
GET /api/v1/admin/stores/{storeId}
```

---

## 24.2 매장 정보 수정

```http
PUT /api/v1/admin/stores/{storeId}
```

### 요청

```json
{
  "name": "자리요 헤어",
  "description": "예약과 현장 대기가 가능한 헤어숍",
  "phoneNumber": "0531234567",
  "address": "대구광역시 ...",
  "timezone": "Asia/Seoul"
}
```

---

# 25. 통계 API

## 25.1 운영 요약 통계

```http
GET /api/v1/admin/stores/{storeId}/analytics/summary
```

### 쿼리

```text
from
to
```

### 응답

```json
{
  "data": {
    "reservationCount": 320,
    "completionRate": 0.84,
    "cancellationRate": 0.09,
    "noShowRate": 0.07,
    "waitlistOfferAcceptanceRate": 0.52,
    "cancelledSlotRefillRate": 0.41,
    "averageWalkInWaitMinutes": 24,
    "walkInAbandonmentRate": 0.12
  }
}
```

---

## 25.2 일별 예약 통계

```http
GET /api/v1/admin/stores/{storeId}/analytics/reservations/daily
```

---

## 25.3 직원별 운영 통계

```http
GET /api/v1/admin/stores/{storeId}/analytics/staff
```

---

## 25.4 서비스별 실제 소요 시간

```http
GET /api/v1/admin/stores/{storeId}/analytics/services/duration
```

---

# 26. 감사 로그 API

## 26.1 감사 로그 조회

```http
GET /api/v1/admin/stores/{storeId}/audit-logs
```

### 쿼리

```text
actorId
action
targetType
targetId
from
to
cursor
limit
```

### 응답

```json
{
  "data": [
    {
      "id": "audit_123",
      "actor": {
        "type": "STORE_MEMBER",
        "id": "member_123",
        "displayName": "민지"
      },
      "action": "WALK_IN_REORDERED",
      "targetType": "WALK_IN_ENTRY",
      "targetId": "walk_123",
      "reason": "고객 요청",
      "occurredAt": "2026-07-11T14:00:00+09:00"
    }
  ]
}
```

---

# 27. 처리 실패 및 재처리 API

## 27.1 실패 작업 목록 조회

```http
GET /api/v1/admin/stores/{storeId}/failed-jobs
```

### 쿼리

```text
type
status
from
to
cursor
limit
```

### 응답

```json
{
  "data": [
    {
      "id": "job_123",
      "type": "SEND_NOTIFICATION",
      "referenceType": "RESERVATION",
      "referenceId": "res_123",
      "status": "FAILED",
      "attemptCount": 5,
      "lastError": {
        "code": "PROVIDER_TIMEOUT",
        "message": "알림 제공자 응답 시간이 초과되었습니다."
      },
      "failedAt": "2026-07-11T14:00:00+09:00"
    }
  ]
}
```

---

## 27.2 실패 작업 상세 조회

```http
GET /api/v1/admin/stores/{storeId}/failed-jobs/{jobId}
```

---

## 27.3 실패 작업 재처리

```http
POST /api/v1/admin/stores/{storeId}/failed-jobs/{jobId}/retry
Idempotency-Key: {key}
```

### 응답

```json
{
  "data": {
    "jobId": "job_123",
    "status": "PENDING"
  }
}
```

---

## 27.4 실패 작업 무시 처리

```http
POST /api/v1/admin/stores/{storeId}/failed-jobs/{jobId}/ignore
```

### 요청

```json
{
  "reason": "고객에게 전화로 직접 안내 완료"
}
```

---

# 28. 실시간 상태 전달

예약 대기, 현장 대기, 빈자리 제안과 같은 상태는 실시간 갱신이 필요할 수 있다.

## 28.1 고객 실시간 이벤트 스트림

```http
GET /api/v1/me/events/stream
```

### 이벤트 예시

```text
RESERVATION_UPDATED
WAITLIST_UPDATED
SLOT_OFFER_CREATED
SLOT_OFFER_EXPIRED
WALK_IN_UPDATED
WALK_IN_CALLED
NOTIFICATION_CREATED
```

### 데이터 예시

```json
{
  "eventId": "evt_123",
  "type": "WALK_IN_CALLED",
  "occurredAt": "2026-07-11T14:20:00+09:00",
  "data": {
    "walkInId": "walk_123",
    "callExpiresAt": "2026-07-11T14:23:00+09:00"
  }
}
```

---

## 28.2 운영자 매장 이벤트 스트림

```http
GET /api/v1/admin/stores/{storeId}/events/stream
```

### 이벤트 예시

```text
RESERVATION_CREATED
RESERVATION_CANCELLED
CUSTOMER_CHECKED_IN
WALK_IN_CREATED
WALK_IN_UPDATED
SERVICE_STARTED
SERVICE_COMPLETED
FAILED_JOB_CREATED
```

실시간 연결이 끊긴 경우 클라이언트는 목록 API를 다시 호출해 현재 상태를 복구해야 한다.

실시간 이벤트 자체를 데이터의 진실로 사용하면 안 된다.

---

# 29. 내부 이벤트 스키마

외부 REST API와 별도로 내부 비동기 처리를 위한 이벤트 형식을 정의한다.

```json
{
  "eventId": "evt_123",
  "eventType": "RESERVATION_CONFIRMED",
  "aggregateType": "RESERVATION",
  "aggregateId": "res_123",
  "aggregateVersion": 2,
  "storeId": "store_123",
  "occurredAt": "2026-07-11T10:00:00+09:00",
  "traceId": "trace_123",
  "schemaVersion": 1,
  "payload": {}
}
```

## 주요 이벤트

```text
RESERVATION_HELD
RESERVATION_CONFIRMED
RESERVATION_CANCELLED
RESERVATION_EXPIRED
RESERVATION_CHECKED_IN
RESERVATION_NO_SHOWED

WAITLIST_CREATED
WAITLIST_CANCELLED
SLOT_OFFER_CREATED
SLOT_OFFER_ACCEPTED
SLOT_OFFER_DECLINED
SLOT_OFFER_EXPIRED

WALK_IN_CREATED
WALK_IN_CALLED
WALK_IN_SKIPPED
WALK_IN_CHECKED_IN
WALK_IN_NO_SHOWED

SERVICE_STARTED
SERVICE_COMPLETED

NOTIFICATION_REQUESTED
NOTIFICATION_SENT
NOTIFICATION_FAILED
```

---

# 30. 상태 전이 규칙

## 30.1 예약

```text
HELD → CONFIRMED
HELD → EXPIRED
HELD → CANCELLED

CONFIRMED → CHECKED_IN
CONFIRMED → CANCELLED
CONFIRMED → NO_SHOW

CHECKED_IN → IN_SERVICE
CHECKED_IN → CANCELLED

IN_SERVICE → COMPLETED
```

허용하지 않는 예:

```text
CANCELLED → CONFIRMED
COMPLETED → CANCELLED
NO_SHOW → IN_SERVICE
EXPIRED → CONFIRMED
```

---

## 30.2 예약 대기

```text
WAITING → OFFERED
WAITING → CANCELLED
WAITING → EXPIRED

OFFERED → WAITING
OFFERED → RESERVED
OFFERED → CANCELLED
OFFERED → EXPIRED
```

---

## 30.3 빈자리 제안

```text
PENDING → ACCEPTED
PENDING → DECLINED
PENDING → EXPIRED
PENDING → REVOKED
```

---

## 30.4 현장 대기

```text
WAITING → CALLED
WAITING → CANCELLED

CALLED → CHECKED_IN
CALLED → SKIPPED
CALLED → NO_SHOW
CALLED → CANCELLED

SKIPPED → WAITING
SKIPPED → CALLED
SKIPPED → CANCELLED

CHECKED_IN → IN_SERVICE
CHECKED_IN → CANCELLED

IN_SERVICE → COMPLETED
```

---

# 31. 권한 매트릭스

| 기능          |  고객 |  직원 | 관리자 | 소유자 |    시스템 |
| ----------- | --: | --: | --: | --: | -----: |
| 공개 매장 조회    |   O |   O |   O |   O |      O |
| 본인 예약 생성    |   O |   O |   O |   O |      X |
| 본인 예약 취소    |   O |   O |   O |   O |    조건부 |
| 매장 예약 수동 생성 |   X |   O |   O |   O |      X |
| 예약 시간 변경    |   X | 제한적 |   O |   O |      X |
| 노쇼 처리       |   X |   O |   O |   O | 정책에 따라 |
| 예약 대기 신청    |   O |   O |   O |   O |      X |
| 빈자리 제안 수락   | 본인만 |   X |   X |   X |      X |
| 수동 빈자리 제안   |   X | 제한적 |   O |   O |      O |
| 현장 대기 등록    |   O |   O |   O |   O |      X |
| 고객 호출       |   X |   O |   O |   O |    조건부 |
| 대기 순서 변경    |   X | 제한적 |   O |   O |      X |
| 서비스 시작·완료   |   X |   O |   O |   O |      X |
| 서비스 관리      |   X |   X |   O |   O |      X |
| 직원 관리       |   X |   X | 제한적 |   O |      X |
| 정책 변경       |   X |   X |   O |   O |      X |
| 실패 작업 재처리   |   X |   X |   O |   O |      O |
| 감사 로그 조회    |   X | 제한적 |   O |   O |      X |

---

# 32. 오류 코드 목록

## 인증 및 권한

```text
AUTHENTICATION_REQUIRED
INVALID_ACCESS_TOKEN
ACCESS_TOKEN_EXPIRED
ACCESS_DENIED
STORE_ACCESS_DENIED
RESOURCE_NOT_OWNED_BY_USER
```

## 사용자

```text
EMAIL_ALREADY_EXISTS
PHONE_NUMBER_ALREADY_EXISTS
USER_NOT_FOUND
USER_SUSPENDED
```

## 매장

```text
STORE_NOT_FOUND
STORE_NOT_ACTIVE
STORE_TEMPORARILY_CLOSED
STORE_OPERATION_CLOSED
```

## 서비스와 직원

```text
SERVICE_NOT_FOUND
SERVICE_NOT_ACTIVE
STAFF_NOT_FOUND
STAFF_NOT_ACTIVE
STAFF_NOT_AVAILABLE
STAFF_NOT_ASSIGNED_TO_SERVICE
STAFF_HAS_FUTURE_RESERVATIONS
```

## 예약

```text
RESERVATION_NOT_FOUND
RESERVATION_SLOT_ALREADY_TAKEN
RESERVATION_INVALID_STATE
RESERVATION_ALREADY_CANCELLED
RESERVATION_HOLD_EXPIRED
RESERVATION_OUTSIDE_BOOKING_WINDOW
RESERVATION_TOO_CLOSE_TO_START
RESERVATION_CANCELLATION_DEADLINE_PASSED
RESERVATION_NOT_RESCHEDULABLE
CUSTOMER_HAS_OVERLAPPING_RESERVATION
```

## 예약 대기

```text
WAITLIST_NOT_FOUND
WAITLIST_NOT_ENABLED
WAITLIST_DUPLICATED
WAITLIST_INVALID_STATE
WAITLIST_NOT_EDITABLE
WAITLIST_DATE_OUT_OF_RANGE
WAITLIST_CONDITION_NOT_MATCHED
INVALID_WAITLIST_TIME_RANGE
```

## 빈자리 제안

```text
SLOT_OFFER_NOT_FOUND
SLOT_OFFER_EXPIRED
SLOT_OFFER_ALREADY_ACCEPTED
SLOT_OFFER_ALREADY_DECLINED
SLOT_OFFER_ALREADY_ACTIVE
SLOT_OFFER_NO_LONGER_AVAILABLE
```

## 현장 대기

```text
WALK_IN_NOT_FOUND
WALK_IN_NOT_ENABLED
WALK_IN_REGISTRATION_CLOSED
WALK_IN_ALREADY_REGISTERED
WALK_IN_INVALID_STATE
WALK_IN_ALREADY_CALLED
WALK_IN_CALL_EXPIRED
```

## 체크인

```text
CHECK_IN_NOT_AVAILABLE_YET
CHECK_IN_WINDOW_EXPIRED
CHECK_IN_ALREADY_COMPLETED
CHECK_IN_TOKEN_INVALID
CHECK_IN_TOKEN_EXPIRED
CHECK_IN_TOKEN_ALREADY_USED
```

## 서비스 세션

```text
SERVICE_SESSION_NOT_FOUND
SERVICE_SESSION_ALREADY_STARTED
SERVICE_SESSION_ALREADY_COMPLETED
SERVICE_SESSION_INVALID_STATE
```

## 요청

```text
INVALID_REQUEST
MISSING_REQUIRED_FIELD
INVALID_FIELD_FORMAT
INVALID_DATE_RANGE
INVALID_PARTY_SIZE
IDEMPOTENCY_KEY_REQUIRED
IDEMPOTENCY_KEY_REUSED_WITH_DIFFERENT_REQUEST
RATE_LIMIT_EXCEEDED
```

## 시스템

```text
INTERNAL_SERVER_ERROR
TEMPORARY_UNAVAILABLE
DEPENDENCY_UNAVAILABLE
OPERATION_TIMEOUT
```

---

# 33. 페이지네이션

목록 API는 cursor 기반 페이지네이션을 기본으로 한다.

### 요청

```http
GET /api/v1/me/reservations?limit=20&cursor=abc
```

### 응답

```json
{
  "data": [],
  "page": {
    "cursor": "next_cursor",
    "hasNext": true
  }
}
```

`limit` 기본값은 20, 최대값은 100으로 제한한다.

---

# 34. 정렬 규칙

정렬이 필요한 목록은 명시적인 `sort` 값을 사용한다.

예:

```text
CREATED_AT_DESC
START_AT_ASC
SEQUENCE_ASC
PRIORITY_DESC
```

임의의 필드명을 클라이언트가 전달하는 방식은 허용하지 않는다.

---

# 35. 동시성 제어가 필요한 API

다음 API는 상태 조건과 버전 또는 최종 무결성 검증이 필요하다.

```text
POST /reservations
POST /reservations/{id}/confirm
POST /reservations/{id}/cancel
POST /slot-offers/{id}/accept
POST /slot-offers/{id}/decline
POST /walk-ins
POST /walk-ins/{id}/respond-call
POST /admin/.../walk-ins/{id}/call
POST /admin/.../reservations/{id}/start-service
POST /admin/.../service-sessions/{id}/complete
```

예를 들어 빈자리 제안 수락은 다음 조건을 한 번에 확인해야 한다.

```text
현재 제안 상태가 PENDING
제안 만료 전
대기 상태가 OFFERED
슬롯이 아직 비어 있음
동일 제안으로 예약이 생성되지 않음
```

---

# 36. API별 트랜잭션 경계

## 예약 생성

```text
중복 요청 확인
예약 충돌 확인
예약 생성
상태 이력 생성
도메인 이벤트 기록
멱등 요청 결과 기록
```

## 예약 취소

```text
예약 상태 변경
취소 정보 기록
상태 이력 생성
취소 자리 이벤트 기록
기존 예약 알림 작업 취소 요청 기록
```

## 빈자리 제안 수락

```text
제안 상태 변경
대기 상태 변경
예약 생성
제안과 예약 연결
만료 작업 취소
상태 이력 기록
이벤트 기록
```

## 현장 고객 호출

```text
현장 대기 상태 변경
호출 시각과 만료 시각 기록
호출 이력 생성
알림 이벤트 기록
```

## 서비스 시작

```text
예약 또는 현장 대기 상태 변경
서비스 세션 생성
실제 시작 시각 기록
이벤트 기록
```

## 서비스 완료

```text
서비스 세션 완료
예약 또는 현장 대기 완료
실제 종료 시각 기록
이벤트 기록
통계 갱신 이벤트 기록
```

---

# 37. API 구현 우선순위

## 1차 고객 핵심 API

```text
GET /stores/{storeId}
GET /stores/{storeId}/services
GET /stores/{storeId}/services/{serviceId}/staff
GET /stores/{storeId}/availability
POST /reservations
GET /me/reservations
GET /reservations/{reservationId}
POST /reservations/{reservationId}/cancel
```

## 2차 예약 대기 API

```text
POST /waitlists
GET /me/waitlists
GET /waitlists/{waitlistId}
POST /waitlists/{waitlistId}/cancel
GET /slot-offers/{offerId}
POST /slot-offers/{offerId}/accept
POST /slot-offers/{offerId}/decline
```

## 3차 현장 대기 API

```text
GET /stores/{storeId}/walk-in-status
POST /walk-ins
GET /walk-ins/{walkInId}
POST /walk-ins/{walkInId}/cancel
POST /walk-ins/{walkInId}/respond-call
POST /reservations/{reservationId}/check-in
```

## 4차 운영자 API

```text
GET /admin/stores/{storeId}/dashboard/today
GET /admin/stores/{storeId}/reservations
POST /admin/stores/{storeId}/reservations
GET /admin/stores/{storeId}/walk-ins
POST /admin/stores/{storeId}/walk-ins/{walkInId}/call
POST /admin/stores/{storeId}/reservations/{reservationId}/start-service
POST /admin/stores/{storeId}/service-sessions/{sessionId}/complete
```

## 5차 매장 설정 API

```text
서비스 관리
직원 관리
근무 일정
휴무 일정
매장 정책
감사 로그
실패 작업
통계
```

---

# 38. MVP에서 제외해도 되는 API

초기 MVP에서는 다음 API를 나중으로 미룰 수 있다.

```text
QR 체크인
실시간 이벤트 스트림
고급 통계
직원 초대 이메일
실패 작업 관리 화면
빈자리 제안 수동 생성
대기 순서 직접 변경
서비스별 실제 시간 분석
```

다만 데이터 모델과 URL 구조는 이후 확장을 막지 않도록 유지한다.

---

# 39. OpenAPI 문서 구성 권장안

```text
openapi/
├─ openapi.yaml
├─ paths/
│  ├─ auth.yaml
│  ├─ stores.yaml
│  ├─ availability.yaml
│  ├─ reservations.yaml
│  ├─ waitlists.yaml
│  ├─ slot-offers.yaml
│  ├─ walk-ins.yaml
│  ├─ check-ins.yaml
│  ├─ notifications.yaml
│  └─ admin/
│     ├─ dashboard.yaml
│     ├─ reservations.yaml
│     ├─ walk-ins.yaml
│     ├─ services.yaml
│     ├─ staff.yaml
│     ├─ schedules.yaml
│     ├─ policies.yaml
│     ├─ analytics.yaml
│     └─ operations.yaml
└─ components/
   ├─ schemas.yaml
   ├─ errors.yaml
   ├─ parameters.yaml
   ├─ responses.yaml
   └─ security.yaml
```

---

# 40. API 설계 핵심 원칙

자리요 API는 다음 원칙을 따른다.

1. 클라이언트가 상태 값을 직접 덮어쓰지 않는다.
2. 예약 취소, 체크인, 호출처럼 의미 있는 행동은 별도 명령 API로 표현한다.
3. 조회 결과와 최종 예약 성공 여부를 구분한다.
4. 중복 요청이 발생해도 같은 결과를 반환한다.
5. 알림 실패가 예약 성공을 되돌리지 않는다.
6. 실시간 이벤트는 상태 알림 수단이며 진실 공급원이 아니다.
7. 운영자 권한은 매장 소속과 역할을 함께 검증한다.
8. 복구하기 어려운 변경은 감사 로그를 남긴다.
9. 상태 전이는 허용된 경로로만 수행한다.
10. 동시 요청은 정상적인 상황으로 간주하고 충돌 응답을 명확히 정의한다.
