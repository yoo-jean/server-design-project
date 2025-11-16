# API 명세서 (통합 버전)

콘서트 좌석 예약 서비스의 전체 API 명세서, 에러 코드, Status Enum을 통합한 문서입니다.

## 공통 사항
- Base URL: `/api`
- 모든 응답은 `application/json`
- 시간 형식은 ISO-8601
- (선택) 대기열 기능 사용 시, 모든 API 요청에 `X-QUEUE-TOKEN` 헤더 필요

```http
X-QUEUE-TOKEN: {대기열 토큰 UUID}
```

---

# 1. 공통 HTTP Status 규격

| HTTP Status | 의미 |
|-------------|------|
| 200 OK | 요청 성공 |
| 201 CREATED | 리소스 생성 성공 |
| 400 BAD_REQUEST | 잘못된 요청 / 유효성 오류 |
| 401 UNAUTHORIZED | 인증 실패 |
| 403 FORBIDDEN | 권한 없음 |
| 404 NOT_FOUND | 리소스 없음 |
| 409 CONFLICT | 자원 충돌 (중복 예약 등) |
| 422 UNPROCESSABLE_ENTITY | 예약 만료 / 처리 불가 |
| 429 TOO_MANY_REQUESTS | 요청 과다 |
| 500 INTERNAL_SERVER_ERROR | 서버 내부 오류 |

---

# 2. 공통 Error Code 규격

| Error Code | 설명 |
|------------|------|
| INVALID_REQUEST | 잘못된 요청 데이터 |
| VALIDATION_FAILED | 요청 데이터 검증 실패 |
| UNAUTHORIZED | 인증 실패 |
| FORBIDDEN | 권한 없음 |
| NOT_FOUND | 리소스 없음 |
| CONFLICT | 자원 충돌 |
| EXPIRED_RESERVATION | 임시 예약 만료 |
| TOO_MANY_REQUESTS | 요청 과다 |
| INTERNAL_SERVER_ERROR | 서버 내부 오류 |

---

# 3. API별 에러 코드

## 좌석 예약 API

| HTTP Status | Error Code | 설명 |
|------------|------------|------|
| 400 | INVALID_SEAT_NUMBER | 좌석 번호 1~50 범위 아님 |
| 404 | SCHEDULE_NOT_FOUND | 스케줄 없음 |
| 404 | SEAT_NOT_FOUND | 좌석 없음 |
| 409 | SEAT_ALREADY_BOOKED | 이미 BOOKED 상태 |
| 409 | SEAT_TEMPORARILY_RESERVED | 다른 유저가 임시 예약 중 |
| 422 | TEMP_RESERVATION_EXPIRED | 임시 예약 만료 |

---

## 포인트 API

| HTTP Status | Error Code | 설명 |
|------------|------------|------|
| 400 | INVALID_AMOUNT | 충전 금액 오류 |
| 404 | USER_NOT_FOUND | 유저 없음 |

---

## 결제 API

| HTTP Status | Error Code | 설명 |
|------------|------------|------|
| 400 | INVALID_RESERVATION | 예약 ID 오류 |
| 400 | USER_MISMATCH | 요청한 유저와 예약 유저 불일치 |
| 422 | RESERVATION_EXPIRED | 예약 만료 |
| 402 | INSUFFICIENT_POINTS | 잔액 부족 |
| 409 | PAYMENT_ALREADY_COMPLETED | 이미 결제 완료 상태 |

---

## 대기열 API (선택)

| HTTP Status | Error Code | 설명 |
|------------|------------|------|
| 400 | INVALID_TOKEN | 잘못된 토큰 형식 |
| 404 | TOKEN_NOT_FOUND | 토큰 없음 |
| 403 | QUEUE_NOT_ACTIVE | ACTIVE 상태 아님 |
| 410 | QUEUE_TOKEN_EXPIRED | 토큰 만료됨 |

---

# 4. Status Enum 정리

## SeatStatus
| 값 | 설명 |
|----|------|
| AVAILABLE | 예약 가능 |
| TEMP_RESERVED | 임시 예약됨 |
| BOOKED | 결제 완료 |

## ReservationStatus
| 값 | 설명 |
|----|------|
| TEMP_RESERVED | 임시 예약 |
| CONFIRMED | 결제 완료 |
| CANCELLED | 취소 |
| EXPIRED | 만료됨 |

## PaymentStatus
| 값 | 설명 |
|----|------|
| SUCCESS | 결제 성공 |
| FAIL | 결제 실패 |
| REFUND | 환불 |

## QueueTokenStatus (선택)
| 값 | 설명 |
|----|------|
| WAITING | 대기중 |
| ACTIVE | 예약 가능 상태 |
| EXPIRED | 만료 |

---

# 5. 공통 Response 규격

## 성공 Response
```json
{
  "success": true,
  "data": { }
}
```

## 실패 Response
```json
{
  "success": false,
  "error": {
    "status": 400,
    "code": "INVALID_REQUEST",
    "message": "잘못된 요청입니다."
  }
}
```

---

# 6. API 명세

---

# 6-1. 예약 가능 날짜 조회

**GET** `/api/concerts/{concertId}/schedules`

### Request
- Path Variable
    - `concertId`: 콘서트 ID
- Query (선택)
    - `status`: OPEN / CLOSED

### Response
```json
[
  {
    "scheduleId": 10,
    "concertId": 1,
    "concertName": "2025 HH Concert",
    "date": "2025-12-24T20:00:00",
    "status": "OPEN",
    "totalSeats": 50,
    "availableSeats": 32
  }
]
```

---

# 6-2. 좌석 조회

**GET** `/api/concerts/{concertId}/schedules/{scheduleId}/seats`

### Response
```json
[
  {
    "seatId": 1001,
    "seatNumber": 1,
    "status": "AVAILABLE",
    "tempReservedUntil": null
  },
  {
    "seatId": 1002,
    "seatNumber": 2,
    "status": "TEMP_RESERVED",
    "tempReservedUntil": "2025-11-16T12:30:00"
  }
]
```

---

# 6-3. 좌석 예약 요청

**POST** `/api/reservations`

### Request Body
```json
{
  "userId": 1,
  "scheduleId": 10,
  "seatNumber": 5
}
```

### Response
```json
{
  "reservationId": 5001,
  "userId": 1,
  "scheduleId": 10,
  "seatId": 1005,
  "seatNumber": 5,
  "status": "TEMP_RESERVED",
  "tempReservedUntil": "2025-11-16T12:35:00"
}
```

---

# 6-4. 포인트 충전

**POST** `/api/points/charge`

### Request
```json
{
  "userId": 1,
  "amount": 50000
}
```

### Response
```json
{
  "userId": 1,
  "currentBalance": 150000
}
```

---

# 6-5. 포인트 조회

**GET** `/api/points/{userId}`

### Response
```json
{
  "userId": 1,
  "currentBalance": 150000
}
```

---

# 6-6. 결제 API

**POST** `/api/payments`

### Request Body
```json
{
  "userId": 1,
  "reservationId": 5001,
  "amount": 30000
}
```

### Response
```json
{
  "paymentId": 9001,
  "reservationId": 5001,
  "userId": 1,
  "amount": 30000,
  "status": "SUCCESS",
  "paidAt": "2025-11-16T12:31:00",
  "seat": {
    "scheduleId": 10,
    "seatNumber": 5
  }
}
```

---

# 6-7. (선택) 대기열 토큰 발급

**POST** `/api/queue/token`

### Request Body
```json
{
  "userId": 1
}
```

### Response
```json
{
  "token": "7b20b4e3-4a42-4d2c-97b8-8a5b9e9b21c1",
  "userId": 1,
  "status": "WAITING",
  "position": 120,
  "issuedAt": "2025-11-16T12:00:00",
  "estimatedWaitSeconds": 600
}
```

---

# 6-8. (선택) 대기열 상태 조회

**GET** `/api/queue/token/{token}`

### Response
```json
{
  "token": "7b20b4e3-4a42-4d2c-97b8-8a5b9e9b21c1",
  "userId": 1,
  "status": "ACTIVE",
  "position": 0,
  "estimatedWaitSeconds": 0,
  "activeExpiresAt": "2025-11-16T12:15:00"
}
```
---

