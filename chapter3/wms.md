# Chapter 3. 애그리거트 — WMS 적용

> **기준 프로젝트:** WMS (Spring Boot 3.2 / Java 17)

---

## 1. WMS의 애그리거트 구조

WMS의 핵심 도메인은 다음 애그리거트로 구성됩니다.

| 애그리거트 루트 | 포함 객체 | 비즈니스 식별자 |
|----------------|----------|---------------|
| `Inbound` | `InboundItem` (1..N) | `wikey` |
| `Outbound` | `OutboundItem` (1..N), `Orderer` (VO), `Delivery` (VO) | `wokey` |
| `Stock` | `StockCountByStatus` (VO) | `centerId + productItemId` |
| `Reverse` | — | `reverseCode` |
| `Center` | `CenterAddress` (VO) | `centerCode` |

각 루트는 자신에 속한 객체들의 **일관성 유지 책임**을 집니다.

---

## 2. 루트가 일관성을 지키는 방법

### 2-1. 의미 있는 메서드로 상태 변경

WMS의 `Inbound` / `Outbound` 엔티티에는 `setStatusCode()` 같은 setter가 없습니다.  
상태 변경은 반드시 **도메인 의도가 담긴 메서드**를 통해서만 일어납니다.

```java
// ❌ setter — 실제 코드에 없음
inbound.setStatusCode(InboundStatusCode.COMPLETED);

// ✅ 실제 코드 — 검증 + 상태 변경이 하나의 메서드 안에
public void complete(LocalDate completeDay) {
    if (InboundViewAction.COMPLETE.isNotAllowedBy(this.category)) {
        throw new BadRequestException(BAD_REQUEST, CANNOT_COMPLETE_CATEGORY);
    }
    if (this.statusCode != InboundStatusCode.SENT) {
        throw new BadRequestException(BAD_REQUEST, CANNOT_COMPLETE_STATUS);
    }
    this.statusCode = InboundStatusCode.COMPLETED;
    this.completeDay = completeDay;
}
```

`complete()`를 호출하면 **카테고리 검증 → 상태 검증 → 상태 변경**이 원자적으로 일어납니다.  
서비스 레이어에서 이 검증을 직접 할 필요가 없습니다.

### 2-2. 루트를 통한 하위 엔티티 접근

```java
// Inbound.java — 루트가 하위 항목 추가를 제어
public void addItem(InboundItem inboundItem) {
    this.inboundItems.add(inboundItem);
    if (inboundItem.getInbound() != this) {
        inboundItem.setInbound(this); // 연관관계 일관성 유지
    }
}

// 루트가 delete 시 하위 항목도 함께 비활성화
public void delete() {
    verifyDeleted();
    this.activated = ActiveYN.N;
    for (InboundItem item : this.getActivatedItems()) {
        item.delete(); // 루트 통해서만 cascade
    }
}
```

외부에서 `InboundItem`을 직접 삭제하면 `Inbound`가 알 수 없습니다.  
`Inbound.delete()`를 통해서만 삭제가 이루어지기 때문에 **항상 일관된 상태를 유지**합니다.

### 2-3. 입고 완료 시 품목 일관성 검증

```java
// Inbound.java — 완료 요청 품목 수와 실제 품목 수가 일치하는지 루트가 직접 검증
public Map<Long, InboundItem> validateCompleteItems(List<InboundItemCompleteDto> completeItems) {
    Map<Long, InboundItem> inboundItems = this.getInboundItems().stream()
            .filter(InboundItem::isActivated)
            .collect(Collectors.toMap(InboundItem::getId, Function.identity()));

    if (inboundItems.size() != completeItems.size()) {
        throw new BadRequestException(BAD_REQUEST, REQUIRED_ALL_INBOUND_ITEMS);
    }
    return inboundItems;
}
```

"입고 완료 시 모든 품목이 처리되어야 한다"는 도메인 규칙이 서비스가 아닌 **루트 엔티티 안에** 있습니다.

---

## 3. 밸류는 불변으로 — StockCountByStatus

`Stock` 엔티티의 핵심 밸류 타입인 `StockCountByStatus`는 **불변으로 설계**되어 있습니다.  
상태를 바꿀 때는 내부 값을 직접 수정하지 않고, **새 인스턴스를 반환**합니다.

```java
// Stock.java — 루트가 밸류를 교체하는 방식
public void registerOutbound(Integer quantity) {
    int stored = this.getStockCountByStatus().getAvailableQuantity();
    if (quantity > stored) {
        throw new BadRequestException(BAD_REQUEST, UNAVAILABLE_OUTBOUND_QUANTITY);
    }
    // 새 인스턴스 반환 — 기존 인스턴스를 변경하지 않음
    this.stockCountByStatus = this.stockCountByStatus.registerOutbound(quantity);
}

public void completeInbound(Integer availableQuantity, Integer receivedQuantity) {
    this.stockCountByStatus = this.stockCountByStatus.completeInbound(availableQuantity, receivedQuantity);
}
```

각 메서드가 `StockCountByStatus`의 새 인스턴스를 만들어 `Stock`에 세팅합니다.  
→ **부작용(side effect)이 없고, 어느 시점의 재고 상태든 추적할 수 있습니다.**

---

## 4. ID 기반 참조 — 다른 애그리거트를 ID로만 참조

WMS에서 `Inbound`와 `Outbound`는 다른 애그리거트(`Center`, `Vendor`)를 **직접 객체 참조하지 않고 ID로만 참조**합니다.

```java
// Inbound.java
@Column(name = "center_id")
private Long centerId;  // Center 엔티티 직접 참조 ❌

@Column(name = "vendor_id")
private Long vendorId;  // Vendor 엔티티 직접 참조 ❌
```

```java
// Outbound.java
@Column(name = "center_id")
private Long centerId;

@Column(name = "vendor_id")
private Long vendorId;

@Column(name = "order_id")
private String orderId; // 앱 주문 원본 ID — 외부 시스템 참조도 ID로
```

**이렇게 하는 이유:**
- `Inbound`를 불러올 때 `Center` 전체가 로딩되지 않습니다.
- `Center`가 바뀌어도 `Inbound` 구조에 영향이 없습니다.
- 나중에 도메인이 MSA로 분리되더라도 참조 방식을 바꾸지 않아도 됩니다.

---

## 5. 도메인 규칙이 Enum에 — InboundViewAction

허용/불가 로직을 `if` 분기로 서비스에 두지 않고, **Enum이 직접 판단**합니다.

```java
public enum InboundViewAction {
    COMPLETE("완료", EnumSet.of(InboundCategory.ADJUST)),          // ADJUST만 완료 가능
    CANCEL("취소",   EnumSet.of(InboundCategory.ADJUST, ...)),
    SEND("전송",     EnumSet.of(InboundCategory.ADJUST, ...));

    private final EnumSet<InboundCategory> unableCategories;

    // 이 카테고리로는 이 액션을 할 수 없는가?
    public boolean isNotAllowedBy(InboundCategory category) {
        return Optional.ofNullable(category)
                .map(this.unableCategories::contains)
                .orElse(true);
    }
}
```

```java
// Inbound.java 내 complete() 메서드
if (InboundViewAction.COMPLETE.isNotAllowedBy(this.category)) {
    throw new BadRequestException(BAD_REQUEST, CANNOT_COMPLETE_CATEGORY);
}
```

"어떤 카테고리에서 어떤 액션이 허용되는가"라는 **도메인 규칙 전체가 Enum 하나에** 응집되어 있습니다.  
서비스나 컨트롤러에 카테고리별 분기가 흩어지지 않습니다.

---

## 6. 일급 컬렉션 — Outbounds

여러 `Outbound`를 다룰 때 `List<Outbound>` 대신 `Outbounds`라는 **일급 컬렉션**을 사용합니다.

```java
public class Outbounds {
    private final Set<Outbound> outbounds;

    public Set<Long> ids() { ... }                         // ID 추출
    public Map<String, Outbound> mappedByReferenceCode() { ... }  // 참조코드로 매핑
    public Set<String> wokeys() { ... }                    // 출고키 추출
    public Set<Outbound> filterNotCanceled() { ... }       // 취소되지 않은 것만 필터링
}
```

**장점:**
- 컬렉션 관련 로직(필터, 변환, 추출)이 `Outbounds` 안에 응집됩니다.
- 서비스에서 매번 `stream().filter().collect()` 하지 않아도 됩니다.
- 컬렉션을 다루는 의도가 메서드 이름으로 명확하게 드러납니다.

---

## 7. 재고조정이 루트를 통해 하위 항목을 일괄 처리

```java
// Inbound.java — 재고 조정 완료 시 루트가 하위 InboundItem들도 함께 처리
public void completeByStockAdjustment(LocalDate currentDay) {
    this.completeDay = currentDay;
    this.statusCode = InboundStatusCode.COMPLETED;
    for (InboundItem item : this.getActivatedItems()) {
        item.completeByStockAdjustment();  // 루트가 하위에 위임
    }
}
```

외부에서 `InboundItem`을 직접 순회하며 상태를 바꾸지 않습니다.  
`Inbound`가 자신의 책임 범위인 `InboundItem`들을 직접 조율합니다.

---

## 핵심 요약

| DDD 개념 | WMS 구현 |
|----------|---------|
| **애그리거트 루트** | `Inbound`(wikey), `Outbound`(wokey), `Stock`(centerId+productItemId) |
| **setter 금지** | `complete()`, `send()`, `cancel()` 등 의미 있는 메서드로만 상태 변경 |
| **불변 밸류** | `StockCountByStatus` — 각 메서드가 새 인스턴스 반환 |
| **루트를 통한 접근** | `addItem()`, `delete()`, `validateCompleteItems()` |
| **ID 기반 참조** | `centerId: Long`, `vendorId: Long` — 직접 객체 참조 없음 |
| **Enum 도메인 규칙** | `InboundViewAction.isNotAllowedBy()` — 허용 액션을 Enum이 판단 |
| **일급 컬렉션** | `Outbounds` — 컬렉션 관련 로직 응집 |

---

## 토론 주제

### 💬 Topic 1. `Inbound.setCenterId()`는 왜 setter인데도 존재할까?

WMS 코드를 보면 대부분의 setter가 없지만, `setCenterId()`는 예외적으로 public setter로 남아 있습니다.

```java
/**
 * 입고(Inbound)와 센터(Center)가 서로 다른 도메인 엔티티이기 때문에,
 * 이벤트 발행을 통하여 centerId를 update 한다.
 */
public void setCenterId(Long centerId) {
    this.centerId = centerId;
}
```

주석을 보면 **이벤트를 통해 다른 애그리거트의 ID를 받아올 때** 사용하는 목적입니다.  
즉, 애그리거트 간 ID 참조를 유지하기 위해 어쩔 수 없이 허용된 setter입니다.

**생각해볼 것:**
- `setCenterId()`를 `updateCenter(Long centerId)` 같은 의도 있는 이름으로 바꾸는 것이 나을까?
- 이벤트로 다른 애그리거트의 데이터를 동기화할 때, setter를 허용하지 않으면서 처리하는 방법은?
- 이런 예외적인 setter가 늘어나면 어떤 문제가 생기는가?

---

### 💬 Topic 2. `Outbounds` 일급 컬렉션의 경계 — 어디까지 메서드를 넣어야 할까?

`Outbounds`는 컬렉션 관련 로직을 응집시키는 좋은 패턴입니다.  
하지만 어디까지 메서드를 넣어야 할지 기준이 모호해지는 순간이 옵니다.

```java
// 현재 Outbounds에 있는 것
public Set<Long> ids() { ... }
public Set<Outbound> filterNotCanceled() { ... }

// 이건 넣어야 할까?
public Money calculateTotalAmount() { ... }    // 총 금액 계산
public boolean hasAnyCompleted() { ... }       // 완료된 것이 있는지
public void cancelAll() { ... }                // 전체 취소 — 상태 변경까지?
```

**생각해볼 것:**
- 일급 컬렉션이 상태 변경(`cancelAll()`)을 직접 수행하면 책임이 너무 커지는가?
- 조회(필터/변환/추출)만 허용하고 변경은 루트 엔티티에 두는 것이 나은 방향인가?
- 일급 컬렉션을 만드는 기준은 무엇인가? 단순 `List`를 일급 컬렉션으로 언제 격상시킬까?
