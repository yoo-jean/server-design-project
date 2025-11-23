
---

## `docs/milestone.md`

```md
# 콘서트 예약 서비스 마일스톤

콘서트 좌석 예약 서비스의 개발 과정을 단계별로 나눈 마일스톤입니다.  
각 마일스톤은 GitHub Milestone과 연계하여 이슈 단위로 관리하는 것을 기준으로 설계되었습니다.

---

## Milestone 1 - 설계 및 기본 구조 세팅

**목표:** 시나리오 분석 및 서버 설계 문서를 작성하고, 기본 프로젝트 구조를 세팅한다.

**범위:**

- 콘서트 예약 서비스 시나리오 분석 및 정리
- 요구사항 기반 도메인 정의 (User, PointHistory, Concert, ConcertSchedule, Seat, Reservation, Payment, QueueToken)
- `docs/api.md` 초안 작성 (필수 5개 API + 선택 대기열 관련 API 정의)
- `docs/erd.md` 작성 (Mermaid ERD 및 테이블 설명)
- `docs/infra.md` 작성 (인프라 구성도 및 구성 요소 설명)
- `docs/sequence.md` 작성 (주요 흐름 시퀀스 다이어그램)
- Spring Boot 프로젝트 생성 및 기본 패키지 구조 설정
  - `controller`, `service`, `domain`, `repository`, `dto`, `exception` 등

---

## Milestone 2 - 유저 & 포인트 도메인 및 API 구현

**목표:** 포인트 충전/조회 기능을 구현하고, 결제에 사용될 기초 도메인을 완성한다.

**연결 요구사항:**  
`필수` 포인트 충전 / 조회 API

**범위:**

- User, PointHistory 엔티티 & 리포지토리 구현
- 포인트 충전 API 구현
  - 사용자 식별자 및 금액을 받아 포인트 충전
  - PointHistory에 충전 이력 기록
- 포인트 조회 API 구현
  - 사용자 식별자를 통한 현재 포인트 조회
- 포인트 관련 단위 테스트 작성
  - 정상 충전/조회
  - 잘못된 유저/금액에 대한 에러 처리
- 동시성 대응 방향 정리
  - 다수 인스턴스 환경에서의 포인트 차감/충전 전략 메모 (DB Lock / Redis Lock 등)

---

## Milestone 3 - 콘서트 조회 (예약 가능 날짜/좌석 API)

**목표:** 예약 가능 날짜/좌석 조회 기능을 구현하여, 좌석 예약 전에 사용 가능한 정보를 제공한다.

**연결 요구사항:**  
1️⃣ `필수` 예약 가능 날짜 / 좌석 API (콘서트 조회)

**범위:**

- Concert, ConcertSchedule, Seat 엔티티 & 리포지토리 구현
- 예약 가능 날짜 조회 API 구현
  - 예약 가능한 Schedule 목록 조회
- 특정 날짜(또는 Schedule)에 대한 예약 가능 좌석 API 구현
  - 좌석 번호 1~50 관리
  - 이미 예약/임시 예약된 좌석 필터링
- 조회 API 단위 테스트 작성
- 인덱스/쿼리 성능 고려 (좌석/스케줄 기준 조회)

---

## Milestone 4 - 좌석 예약 요청 & 결제 API, 동시성 제어

**목표:** 좌석 임시 예약(TEMP_RESERVED)과 결제 로직을 구현하고, 좌석 중복 배정을 방지한다.

**연결 요구사항:**  
`필수` 좌석 예약 요청 API  
 `필수` 결제 API  
KEY POINT: 동시성 이슈 및 다수 인스턴스 고려

**범위:**

- Reservation, Payment 엔티티 & 리포지토리 구현
- 좌석 예약 요청 API 구현
  - Schedule, Seat 정보 기반으로 좌석 예약
  - Reservation status = `TEMP_RESERVED`
  - `temp_reserved_until = now + 5분` 설정 (정책에 따라 시간 조정 가능)
  - 이미 TEMP_RESERVED(유효) 또는 CONFIRMED 상태인 좌석은 예약 불가 처리
- 결제 API 구현
  - Reservation 상태 및 만료 여부(`temp_reserved_until`) 검증
  - 포인트 잔액 검증 및 차감 (PointService 연동)
  - Payment 기록 생성
  - Reservation 상태를 `CONFIRMED`로 변경
- 동시성 제어 적용
  - 하나의 좌석에 대해 중복 예약이 불가능하도록 Lock/제약조건 적용
  - 다수 인스턴스 환경에서도 동일 보장이 가능하도록 전략 선택(DB/Redis 등)
- 좌석 예약/결제 관련 단위 테스트 및 간단한 동시성 테스트 작성

---

## Milestone 5 - 대기열 토큰 기능 & 고도화 및 마무리

**목표:** 대기열 토큰 기능을 구현하고, 전체 기능을 다수 인스턴스 환경에도 견고하도록 보완한다.

**연결 요구사항:**  
 `선택` 유저 대기열 토큰 기능  
 `선택` 대기열 고도화

**범위:**

- QueueToken 엔티티 & 리포지토리 구현
- 대기열 토큰 발급 API 구현
  - 유저 UUID/ID 기반 토큰 생성
  - WAITING, ACTIVE, EXPIRED 상태 관리
  - position(대기 순번) 또는 남은 시간 정보 포함
- 대기 번호 조회 API 구현
  - 폴링 방식으로 자신의 상태/순번 확인
- 활성 사용자 수 제한 정책 구현
  - 한 번에 활성화 가능한 사용자 수(N) 설정
  - 활성 상태(`ACTIVE`) 진입/만료 시 QueueToken 상태 관리
- 비즈니스 로직 전반에 대기열 검증 로직 적용
  - 대기열 토큰이 ACTIVE 상태인 경우에만 예약/결제 API 접근 허용
- 전 API 단위 테스트 누
