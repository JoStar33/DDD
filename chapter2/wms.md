# Chapter 2. 아키텍처 개요 — WMS 적용

---

## 1. 네 개의 영역 — WMS 패키지 구조

WMS는 도메인별로 독립된 패키지를 갖고, 그 안에서 4계층으로 분리됩니다.

```
wms.inbound
├── api/            ← 표현 계층 (Controller)
├── application/    ← 응용 계층 (Service, EventListener)
├── domain/         ← 도메인 계층 (Entity, VO, Enum, Domain Service)
├── repository/     ← 인프라 계층 (JPA Repository, Querydsl)
├── usecase/        ← 응용 계층 (UseCase 인터페이스 + Impl)
└── dto/            ← 계층 간 데이터 전달 객체
```

각 도메인(inbound / outbound / stock / reverse / processing / standard)이 모두 같은 구조를 따릅니다.

| 계층 | WMS 구체 클래스 | 설명 |
|------|----------------|------|
| **표현** | `InboundApi`, `OutboundApi` | HTTP 요청 수신, DTO 변환 |
| **응용** | `InboundService`, `OutboundService` | 유스케이스 조율, 트랜잭션 관리 |
| **도메인** | `Inbound`, `Outbound`, `Stock`, `StockCountByStatus` | 핵심 업무 규칙 |
| **인프라** | `InboundRepository`, `OutboundRepository`, Kafka | DB 저장, 메시지 발행 |

---

## 2. DIP — 우리 코드에서의 의존성 역전

### 2-1. UseCase 인터페이스 패턴

WMS에서는 조회 유스케이스를 인터페이스로 분리합니다.

```java
// 인터페이스 — 응용 계층이 이것에만 의존
public interface GetOutboundUseCase {
    OutboundDto invoke(Long outboundId);
}

// 구현체 — 실제 DB 조회 로직
@UseCase
public class GetOutboundUseCaseImpl implements GetOutboundUseCase {
    public OutboundDto invoke(Long outboundId) { ... }
}
```

`@UseCase` 어노테이션은 `@Component` + `@Transactional`을 합친 커스텀 어노테이션입니다.  
Controller는 인터페이스(`GetOutboundUseCase`)에만 의존하므로, 구현체가 바뀌어도 Controller는 영향받지 않습니다.

### 2-2. Repository Custom 인터페이스 패턴

복잡한 조회 쿼리는 Custom 인터페이스로 추상화합니다.

```java
// 인터페이스 (도메인/응용 계층이 의존)
public interface InboundRepositoryCustom {
    List<InboundDto> searchInbounds(InboundSearchCondition condition);
}

// QueryDSL 구현체 (인프라 계층)
public class InboundRepositoryCustomImpl implements InboundRepositoryCustom {
    // QueryDSL로 복잡한 쿼리 구현
}
```

도메인/응용 계층에서는 QueryDSL의 존재를 모릅니다.  
나중에 QueryDSL을 다른 기술로 교체해도 서비스 코드는 수정할 필요가 없습니다.

### 2-3. 도메인 서비스가 인프라를 몰라도 되는 이유

`OutboundStockHandler`는 재고 반영 로직을 가진 도메인 서비스입니다.  
`StockApplier`에만 의존하고, 내부에서 어떤 DB를 쓰는지 알지 못합니다.

```java
// 도메인 서비스 — StockApplier라는 추상화에만 의존
@Component
public class OutboundStockHandler {
    private final StockApplier stockApplier;  // 인터페이스가 아닌 컴포넌트지만 인프라 구현과 분리

    protected void applyStockByDraft(Outbound outbound) {
        List<StockRow> stockRows = outbound.getActivatedItems().stream()
                .map(this::mapToDraft)
                .collect(Collectors.toList());
        stockApplier.applyStockCount(StockApplyDto.from(stockRows));
    }
}
```

---

## 3. 도메인 모델 구성요소

### 3-1. 엔티티

고유 식별자를 가지며 자신의 라이프 사이클(상태 전이)을 갖습니다.

| 엔티티 | 식별자 | 라이프 사이클 |
|--------|--------|-------------|
| `Inbound` | `id`, `wikey` | DRAFT → SENT → COMPLETED / CANCELED |
| `Outbound` | `id`, `wokey` | DRAFT → SENT → COMPLETED / CANCELED |
| `Stock` | `id` + `centerId-productItemId` | 재고 수량의 지속적 변화 |
| `Center` | `id`, `centerCode` | 창고 마스터 |

### 3-2. 밸류 (Value Object)

식별자 없이 개념적으로 하나인 값을 표현합니다. 불변으로 설계되어 새 인스턴스를 반환합니다.

```java
// Invoice — 배송사(Parcel) + 송장번호는 항상 함께 다니는 개념
@Embeddable
public class Invoice {
    private Parcel parcel;  // 배송사 enum
    private String number;  // 송장번호

    public boolean isNotEmpty() {
        return parcel != null && StringUtils.hasText(number);
    }
}

// StockCountByStatus — 재고는 단순 숫자가 아닌 5가지 상태의 묶음
@Embeddable
public class StockCountByStatus {
    private Integer availableQuantity;   // 가용
    private Integer allocationQuantity;  // 할당
    private Integer stopQuantity;        // 중지
    private Integer damageQuantity;      // 손상
    private Integer standbyQuantity;     // 대기

    // 새 인스턴스 반환 — 불변
    public StockCountByStatus registerOutbound(Integer qty) {
        return StockCountByStatus.builder()
                .availableQuantity(this.availableQuantity - qty)
                .allocationQuantity(this.allocationQuantity + qty)
                .build();
    }
}
```

### 3-3. 도메인 서비스

특정 엔티티에 속하지 않는 도메인 로직은 **도메인 서비스**로 분리합니다.

WMS에서는 `domain/service` 패키지 아래에 도메인 서비스들이 위치합니다.

| 도메인 서비스 | 역할 |
|--------------|------|
| `OutboundCreator` | 출고 생성 + 재고 차감을 하나의 단위로 처리 |
| `OutboundStockHandler` | 출고 상태 변화에 따른 재고 반영 로직 |
| `OutboundTopicPublisher` | 출고 이벤트를 Kafka 토픽으로 발행 |
| `StockApplier` | 재고 수량 증감 로직 (여러 도메인에서 공통 사용) |

```java
// OutboundCreator — 출고 생성과 재고 반영은 항상 함께 일어나야 하는 도메인 규칙
@Component
public class OutboundCreator {
    @Transactional
    public Outbound create(OutboundDomainDto outboundDomainDto) {
        final String wokey = generateUtil.generateWokey();
        final Outbound outbound = outboundDomainDto.toEntity(wokey);
        outboundRepository.save(outbound);
        outboundStockHandler.applyStockByDraft(outbound); // 재고 즉시 반영
        return outbound;
    }
}
```

이 로직은 `Outbound` 엔티티에도, `Stock` 엔티티에도 속하지 않습니다.  
"출고가 생성되면 재고를 차감해야 한다"는 규칙이 도메인 서비스로 표현된 것입니다.

### 3-4. 리포지토리

WMS는 **애그리거트 단위**로 리포지토리가 존재합니다.

```java
// ✅ 애그리거트 루트 단위로 존재
InboundRepository       // Inbound 애그리거트 루트
OutboundRepository      // Outbound 애그리거트 루트
StockRepository         // Stock 애그리거트 루트

// ❌ 하위 엔티티용 리포지토리는 없음 (직접 접근 금지)
// InboundItemRepository는 InboundItem 개별 저장 용도가 아닌 조회 보조 목적
```

---

## 4. 애그리거트 — Outbound 군집

WMS에서 `Outbound`는 `OutboundItem`들을 포함하는 애그리거트입니다.

```java
@Entity
public class Outbound {
    @Id
    private Long id;
    private String wokey;                  // 비즈니스 식별자

    @Embedded
    private Orderer orderer;               // 밸류 — 주문자 정보
    @Embedded
    private Delivery delivery;             // 밸류 — 배송지 정보

    @OneToMany(mappedBy = "outbound", cascade = CascadeType.ALL)
    private List<OutboundItem> outboundItems; // 출고 상품 목록
}
```

외부에서는 `Outbound`(루트)를 통해서만 `OutboundItem`에 접근합니다.  
`OutboundStockHandler`도 `outbound.getActivatedItems()`를 통해 내부 아이템에 접근합니다.

```java
// 루트를 통해서만 내부 접근
outbound.getActivatedItems().stream()
    .map(this::mapToDraft)
    ...
```

---

## 5. 이벤트 기반 도메인 간 협력

WMS는 도메인 간 직접 의존 대신 **이벤트**를 통해 협력합니다.

```
[출고 서비스]  →  outbound.complete() 실행
                     ↓
            OutboundCompletedEvent 발행
                     ↓
[재고 서비스]  ←  StockEventListener.onOutboundCompleted()
                     ↓
            stock.completeOutbound()  재고 차감 확정
```

이 구조 덕분에:
- `OutboundService`는 `StockService`를 직접 호출하지 않습니다.
- 재고 로직이 변경되어도 출고 서비스는 영향을 받지 않습니다.

Kafka를 통해 외부 서비스로도 이벤트를 전파합니다.

```java
// OutboundTopicPublisher — Kafka로 출고 이벤트 발행
public void publishOutboundTopic(Long outboundId, OutboundAction outboundAction) {
    // 트랜잭션 커밋 후 발행 (MessageQueueAfterCommitSendEvent)
    eventPublisher.publishEvent(new MessageQueueAfterCommitBulkSendEvent(topics));
}
```

`MessageQueueAfterCommitSendEvent`를 사용해 **트랜잭션이 커밋된 이후에만** Kafka 메시지를 발행합니다.  
DB 저장과 메시지 발행의 일관성을 보장하기 위한 설계입니다.

---

## 핵심 요약

| DDD 개념 | WMS 구현 |
|----------|---------|
| **4계층 분리** | `api / application / domain / repository` 패키지 |
| **DIP** | `GetOutboundUseCase` 인터페이스, `RepositoryCustom` 인터페이스 |
| **도메인 서비스** | `OutboundCreator`, `OutboundStockHandler`, `StockApplier` |
| **애그리거트** | `Outbound` + `OutboundItem` / `Inbound` + `InboundItem` |
| **리포지토리** | 애그리거트 루트 단위 (`OutboundRepository`, `StockRepository`) |
| **이벤트** | Spring Event + Kafka로 도메인 간 느슨한 결합 |

---
