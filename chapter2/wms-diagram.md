# WMS — Chapter 2 아키텍처 다이어그램

---

## 1. 4계층 아키텍처 — WMS 전체 구조

```mermaid
graph TD
    subgraph "표현 계층 (Presentation)"
        API1["InboundApi"]
        API2["OutboundApi"]
        API3["OutboundAppApi"]
    end

    subgraph "응용 계층 (Application)"
        SVC1["InboundService"]
        SVC2["OutboundService"]
        SVC3["StockInboundService\nStockOutboundService"]
        UC1["GetOutboundUseCase\n(interface)"]
        UC2["GetOutboundUseCaseImpl"]
        EL1["StockEventListener"]
    end

    subgraph "도메인 계층 (Domain)"
        E1["Inbound\n(Entity)"]
        E2["Outbound\n(Entity)"]
        E3["Stock\n(Entity)"]
        DS1["OutboundCreator\n(Domain Service)"]
        DS2["OutboundStockHandler\n(Domain Service)"]
        DS3["StockApplier\n(Domain Service)"]
        DS4["OutboundTopicPublisher\n(Domain Service)"]
    end

    subgraph "인프라 계층 (Infrastructure)"
        R1["InboundRepository\n(JPA + Querydsl)"]
        R2["OutboundRepository\n(JPA + Querydsl)"]
        R3["StockRepository\n(JPA)"]
        MQ["Kafka\n메시지 큐"]
        EXT["CJ / TeamFresh\n외부 배송 API"]
    end

    API1 --> SVC1
    API2 --> SVC2
    API3 --> UC1
    UC1 -.->|implements| UC2

    SVC1 --> E1
    SVC2 --> DS1
    DS1 --> E2
    DS1 --> DS2
    DS2 --> DS3
    DS3 --> E3
    SVC2 --> DS4
    DS4 --> MQ

    EL1 --> DS3

    SVC1 --> R1
    SVC2 --> R2
    DS3 --> R3
    SVC2 --> EXT
```

---

## 2. DIP — UseCase 패턴

```mermaid
graph LR
    API["OutboundAppApi\n(Controller)"]
    IF["<<interface>>\nGetOutboundUseCase"]
    IMPL["GetOutboundUseCaseImpl\n(응용 계층)"]
    REPO["OutboundRepository\n(인프라 계층)"]

    API -->|"의존"| IF
    IF -.->|"구현"| IMPL
    IMPL -->|"의존"| REPO

    style IF fill:#fff3cd,stroke:#856404
```

Controller는 인터페이스만 알고, 구현체가 바뀌어도 영향받지 않습니다.

---

## 3. DIP — Repository Custom 패턴

```mermaid
graph LR
    SVC["InboundService\n(응용 계층)"]
    IF["<<interface>>\nInboundRepositoryCustom"]
    IMPL["InboundRepositoryCustomImpl\n(QueryDSL 구현)"]
    DB[("MySQL DB")]

    SVC -->|"의존"| IF
    IF -.->|"구현"| IMPL
    IMPL -->|"쿼리"| DB

    style IF fill:#fff3cd,stroke:#856404
```

---

## 4. 이벤트 기반 도메인 간 협력

```mermaid
sequenceDiagram
    participant OB as OutboundService
    participant EP as ApplicationEventPublisher
    participant EL as StockEventListener
    participant SA as StockApplier
    participant ST as Stock (Entity)
    participant KAFKA as Kafka
    participant EXT as 외부 서비스

    OB->>OB: outbound.complete()
    OB->>EP: OutboundCompletedEvent 발행
    EP->>EL: onOutboundCompleted()
    EL->>SA: applyStockCount()
    SA->>ST: completeOutbound() 재고 차감 확정

    OB->>EP: MessageQueueAfterCommitSendEvent 발행
    Note over EP,KAFKA: 트랜잭션 커밋 이후
    EP->>KAFKA: 출고 토픽 발행
    KAFKA->>EXT: 외부 서비스 수신
```

---

## 5. Outbound 애그리거트 구조

```mermaid
classDiagram
    direction TB

    class Outbound {
        <<Aggregate Root>>
        +Long id
        +String wokey
        +OutboundStatusCode statusCode
        +OutboundCategory category
        +Orderer orderer
        +Delivery delivery
        +getActivatedItems() List~OutboundItem~
        +send()
        +complete()
        +cancel()
    }

    class OutboundItem {
        +Long id
        +Long productItemId
        +Integer availableQuantity
        +Invoice invoice
    }

    class Orderer {
        <<Value Object>>
        +String recipient
        +String name
        +String mainMobileNo
    }

    class Delivery {
        <<Value Object>>
        +String zipcode
        +String address
        +String detailAddress
        +String enterMethod
    }

    class Invoice {
        <<Value Object>>
        +Parcel parcel
        +String number
        +isNotEmpty() bool
    }

    Outbound "1" *-- "N" OutboundItem : 포함
    Outbound *-- Orderer : 내장
    Outbound *-- Delivery : 내장
    OutboundItem *-- Invoice : 내장

    note for Outbound "외부에서는 Outbound를 통해서만\nOutboundItem에 접근"
```

---

## 6. 출고 생성 흐름 — OutboundCreator 도메인 서비스

```mermaid
sequenceDiagram
    participant SVC as OutboundService (응용)
    participant CREATOR as OutboundCreator (도메인 서비스)
    participant OB as Outbound (엔티티)
    participant HANDLER as OutboundStockHandler (도메인 서비스)
    participant APPLIER as StockApplier (도메인 서비스)
    participant ST as Stock (엔티티)

    SVC->>CREATOR: create(outboundDomainDto)
    CREATOR->>OB: toEntity(wokey) — 출고 엔티티 생성
    CREATOR->>CREATOR: outboundRepository.save(outbound)
    CREATOR->>HANDLER: applyStockByDraft(outbound)
    HANDLER->>OB: getActivatedItems()
    HANDLER->>APPLIER: applyStockCount(stockApplyDto)
    APPLIER->>ST: accumulate(stockCountByStatus)
    Note over ST: available - qty\nallocation + qty
```

---

## 7. 패키지 구조 — 계층별 의존 방향

```mermaid
graph TD
    subgraph "outbound 도메인"
        direction TB
        A["api/\nOutboundApi\nOutboundAppApi"]
        B["usecase/\n<<interface>> GetOutboundUseCase\nGetOutboundUseCaseImpl"]
        C["application/\nOutboundService\nOutboundEventListener"]
        D["domain/\nOutbound, OutboundItem\nOrderer, Delivery, Invoice\nOutboundCreator\nOutboundStockHandler\nOutboundTopicPublisher"]
        E["repository/\nOutboundRepository\nOutboundRepositoryCustom\nOutboundRepositoryCustomImpl"]
    end

    A -->|"사용"| B
    A -->|"사용"| C
    C -->|"조합"| D
    C -->|"저장/조회"| E
    D -->|"저장/조회"| E

    style D fill:#d4edda,stroke:#155724
    style E fill:#cce5ff,stroke:#004085
```

> 도메인 계층(녹색)이 핵심. 인프라 계층(파란색)은 도메인을 지원하는 역할.
