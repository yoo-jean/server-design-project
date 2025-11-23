
---

## 3. `docs/infrastructure.md`

```md
# 인프라 구성도

콘서트 예약 서비스의 인프라 구성을 단일 서버 환경과 다수 인스턴스(확장) 환경을 모두 고려하여 설계했습니다.

---

## 1. 인프라 구성 다이어그램 (예시)

Mermaid를 이용한 논리적 인프라 구성도입니다.

```mermaid
flowchart LR

    subgraph Client
        A[Web Browser / Postman]
    end

    A --> B[API Gateway / Load Balancer (Nginx)]

    subgraph AppCluster[Application Layer (Kubernetes / EC2 등)]
        C1[Spring Boot App #1]
        C2[Spring Boot App #2]
        C3[Spring Boot App #N]
    end

    B --> C1
    B --> C2
    B --> C3

    subgraph DataLayer[Data Layer]
        D[(RDB - MySQL / PostgreSQL / H2)]
        E[(Redis - Cache / Queue Token)]
    end

    C1 --> D
    C2 --> D
    C3 --> D

    C1 --> E
    C2 --> E
    C3 --> E
