# 콘서트 좌석 예약 시퀀스 다이어그램

콘서트 좌석 예약 서비스에서 **대기열 → 좌석 임시예약 → 결제 → 좌석 확정**까지의 흐름을 표현한 시퀀스 다이어그램입니다.

## 1. 주요 등장 객체

- `User` : 서비스 이용자
- `QueueToken API` : 대기열 토큰 발급 및 상태 조회 API
- `Reservation API` : 좌석 예약 요청 API
- `Payment API` : 결제 처리 API
- `ReservationService` : 좌석 예약/상태 변경 비즈니스 로직
- `PointService` : 포인트 충전/조회/차감을 담당하는 비즈니스 로직
- `DB` : User, PointHistory, Concert, Schedule, Seat, Reservation, Payment, QueueToken 저장소

---

## 2. 전체 흐름

1. 사용자가 **대기열 토큰**을 발급받는다.
2. 대기열 상태가 `ACTIVE`가 되면, 예약 가능한 상태가 된다.
3. 사용자는 특정 날짜/회차의 **좌석 예약 요청**을 보낸다.
4. 서버는 좌석 중복 여부 및 임시 예약 가능 여부를 검증한 후, 좌석을 `TEMP_RESERVED` 상태로 5분간 임시 배정한다.
5. 사용자는 임시 배정된 좌석에 대해 **결제 API**를 호출한다.
6. 결제에 성공하면:
    - 포인트 잔액 차감
    - Reservation 상태를 `CONFIRMED`로 변경
    - QueueToken 상태를 `EXPIRED`로 변경한다.

---

## 3. Mermaid 시퀀스 다이어그램

```mermaid
sequenceDiagram
    participant U as User
    participant Q as QueueToken API
    participant R as Reservation API
    participant P as Payment API
    participant RS as ReservationService
    participant PS as PointService
    participant DB as DB

    %% 1. 대기열 토큰 발급
    U->>Q: 대기열 토큰 발급 요청
    Q->>DB: QueueToken 생성 (status=WAITING)
    DB-->>Q: 생성 완료
    Q-->>U: 토큰 응답 (id, position, status=WAITING)

    %% 2. 대기열 상태 폴링
    loop until ACTIVE
        U->>Q: 대기열 상태 조회 (토큰 id)
        Q->>DB: 토큰 상태 조회
        DB-->>Q: 현재 상태 반환 (WAITING 또는 ACTIVE)
        Q-->>U: 상태 응답
    end

    %% 3. 좌석 예약 요청
    U->>R: 좌석 예약 요청 (tokenId, scheduleId, seatNo)
    R->>DB: QueueToken 상태 확인 (ACTIVE 여부)
    DB-->>R: status=ACTIVE

    R->>RS: 예약 서비스 호출 (userId, scheduleId, seatNo)
    RS->>DB: 해당 좌석 예약/임시예약 존재 여부 조회
    alt 이미 CONFIRMED 또는 TEMP_RESERVED(유효시간 내)
        RS-->>R: 예약 불가 에러 반환
        R-->>U: 예약 실패 응답
    else 예약 가능
        RS->>DB: Reservation 생성 (status=TEMP_RESERVED, temp_reserved_until=now+5분)
        DB-->>RS: 생성 완료
        RS-->>R: 예약 정보 반환
        R-->>U: 예약 성공 응답 (reservationId 포함)
    end

    %% 4. 결제 요청
    U->>P: 결제 요청 (reservationId, tokenId)
    P->>DB: QueueToken 상태 확인 (ACTIVE 여부)
    DB-->>P: status=ACTIVE

    P->>DB: Reservation 조회
    DB-->>P: Reservation (status=TEMP_RESERVED, temp_reserved_until)

    alt 임시 예약 만료됨 (now > temp_reserved_until)
        P->>DB: Reservation 상태 EXPIRED로 변경
        P-->>U: 결제 실패 응답 (임시 배정 만료)
    else 유효한 임시 예약
        P->>PS: 포인트 차감 요청 (userId, amount)
        PS->>DB: User 포인트 조회 및 잔액 검증
        alt 잔액 부족
            PS-->>P: 실패 반환
            P-->>U: 결제 실패 응답 (잔액 부족)
        else 잔액 충분
            PS->>DB: 포인트 차감 및 PointHistory 기록
            DB-->>PS: 완료
            PS-->>P: 성공 반환

            P->>DB: Reservation 상태 CONFIRMED로 변경
            P->>DB: Payment 기록 생성
            P->>DB: QueueToken 상태 EXPIRED로 변경
            DB-->>P: 완료

            P-->>U: 결제 성공 응답 (좌석 확정)
        end
    end
