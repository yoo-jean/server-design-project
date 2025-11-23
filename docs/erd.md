
---

## 2. `docs/erd.md`

```md
# ERD

콘서트 좌석 예약 서비스에서 사용되는 주요 테이블(엔티티) ERD 설계입니다.

---

## 1. 개요

주요 도메인:

- `User`: 서비스 이용자
- `PointHistory`: 포인트 충전/사용 이력
- `Concert`: 콘서트 정보
- `ConcertSchedule`: 콘서트별 회차/날짜 정보
- `Seat`: 각 회차별 좌석 정보
- `Reservation`: 좌석 예약(임시/확정/취소) 정보
- `Payment`: 결제 정보
- `QueueToken` (선택): 대기열 토큰 정보

---

## 2. ERD (Mermaid)

GitHub에서 렌더링이 가능한 Mermaid ERD 예시입니다.

```mermaid
erDiagram

    USER ||--o{ POINT_HISTORY : has
    USER ||--o{ RESERVATION : makes
    USER ||--o{ QUEUE_TOKEN : holds

    CONCERT ||--o{ CONCERT_SCHEDULE : has
    CONCERT_SCHEDULE ||--o{ SEAT : has
    SEAT ||--o{ RESERVATION : reserved_for
    RESERVATION ||--o{ PAYMENT : paid_by

    USER {
        bigint id PK
        varchar username
        bigint point_balance
        timestamp created_at
        timestamp updated_at
    }

    POINT_HISTORY {
        bigint id PK
        bigint user_id FK
        bigint amount
        varchar type       // CHARGE, PAYMENT, REFUND
        timestamp created_at
    }

    CONCERT {
        bigint id PK
        varchar name
        varchar description
        timestamp created_at
        timestamp updated_at
    }

    CONCERT_SCHEDULE {
        bigint id PK
        bigint concert_id FK
        timestamp start_at
        varchar status      // OPEN, CLOSED
        timestamp created_at
        timestamp updated_at
    }

    SEAT {
        bigint id PK
        bigint schedule_id FK
        int seat_number     // 1 ~ 50
        timestamp created_at
        timestamp updated_at
    }

    RESERVATION {
        bigint id PK
        bigint user_id FK
        bigint seat_id FK
        varchar status      // TEMP_RESERVED, CONFIRMED, CANCELLED, EXPIRED
        timestamp temp_reserved_until
        timestamp created_at
        timestamp updated_at
    }

    PAYMENT {
        bigint id PK
        bigint reservation_id FK
        bigint amount
        varchar status      // SUCCESS, FAIL, REFUND
        timestamp paid_at
        timestamp created_at
    }

    QUEUE_TOKEN {
        uuid id PK
        bigint user_id FK
        varchar status      // WAITING, ACTIVE, EXPIRED
        int position
        timestamp issued_at
        timestamp activated_at
        timestamp expires_at
    }
