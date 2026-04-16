# Chapter 3. 애그리거트
---

## 목차

1. [애그리거트](#31-애그리거트)
2. [애그리거트 루트](#32-애그리거트-루트)
3. [애그리거트 루트의 기능 구현](#33-애그리거트-루트의-기능-구현)
4. [트랜잭션과 애그리거트](#34-트랜잭션과-애그리거트)
5. [애그리거트 간 참조](#35-애그리거트-간-참조)
6. [애그리거트를 팩토리로 사용하기](#36-애그리거트를-팩토리로-사용하기)

---

## 3.1 애그리거트

도메인 모델이 복잡해질수록 전체 구조를 파악하기 어려워집니다.  
**애그리거트**는 연관된 객체를 하나의 군으로 묶어, 개별 객체가 아닌 **상위 수준의 관계**를 볼 수 있게 합니다.

```
주문 도메인
├── Order         ← 애그리거트 루트
│   ├── OrderLine (1..N)
│   └── ShippingInfo
├── Product       ← 별도 애그리거트
└── Member        ← 별도 애그리거트
```

> 개별 테이블 단위로 모델을 보는 것과, 애그리거트 단위로 보는 것은 **이해의 깊이가 다릅니다.**

### 경계 설정 기준

- **함께 생성되고 함께 변경되는** 객체들을 하나의 애그리거트로 묶습니다.
- **불변 조건(invariant)** 을 함께 유지해야 하는 객체들이 한 애그리거트입니다.
- 연관관계가 있다고 해서 반드시 같은 애그리거트가 되는 것은 아닙니다.

```
// ❌ 잘못된 경계 설정
주문
└── 주문한 회원  ← 주문과 함께 생성/변경되지 않음

// ✅ 올바른 경계 설정
주문 (Order)          회원 (Member) — 별도 애그리거트
├── OrderLine
└── ShippingInfo
```

---

## 3.2 애그리거트 루트

애그리거트는 **루트 엔티티** 하나를 갖습니다.  
루트는 애그리거트의 **입구**이자 **일관성 유지의 책임자**입니다.

### 루트만 직접 접근 가능

```java
// ❌ 외부에서 하위 엔티티에 직접 접근
orderLine.setQuantity(2);

// ✅ 루트를 통해 변경
order.changeQuantity(productId, 2); // 루트 메서드 안에서 검증 후 OrderLine 변경
```

외부에서 `OrderLine`을 직접 수정하면 `Order`가 자신의 일관성을 보장할 수 없습니다.

### setter를 없애고 의미 있는 메서드로

```java
// ❌ setter — 도메인 의도가 사라지고, 검증 로직이 흩어짐
order.setStatus(OrderStatus.CANCELED);

// ✅ 의미 있는 메서드 — 도메인 규칙이 메서드 안에 함께
public void cancel() {
    verifyNotYetShipped();
    this.status = OrderStatus.CANCELED;
}
```

---

## 3.3 애그리거트 루트의 기능 구현

### 내부 객체들을 조율하는 역할

루트는 자신이 포함한 다른 객체들에게 기능 실행을 위임합니다.

```java
public class Order {
    private List<OrderLine> orderLines;

    // 루트가 총 금액 계산을 OrderLine에 위임
    public Money calculateTotalAmounts() {
        return orderLines.stream()
            .map(OrderLine::getAmounts)
            .reduce(Money.ZERO, Money::add);
    }

    // 루트를 통해서만 항목 추가 가능 — 일관성 보장
    public void addOrderLine(OrderLine line) {
        verifyNotYetShipped();
        orderLines.add(line);
    }
}
```

### 밸류는 불변으로 설계

밸류를 변경할 때는 새 인스턴스를 생성해 교체합니다.

```java
// ❌ 밸류 내부를 직접 수정
order.getShippingInfo().setAddress("새 주소");

// ✅ 새 밸류 인스턴스 생성 후 교체
ShippingInfo newShipping = ShippingInfo.of("새 주소", zipcode);
order.changeShipping(newShipping);
```

---

## 3.4 트랜잭션과 애그리거트

> **한 트랜잭션에서는 한 애그리거트만 수정한다.**

한 트랜잭션에서 여러 애그리거트를 수정하면:
- **잠금 범위**가 넓어져 동시성 문제가 생깁니다.
- 트랜잭션 실패 시 **롤백 범위**가 예측하기 어려워집니다.

```java
// ❌ 한 트랜잭션에서 두 애그리거트 수정
public void processOrder(Long orderId, Long memberId) {
    Order order = orderRepo.findById(orderId);
    Member member = memberRepo.findById(memberId);

    order.complete();
    member.addPoints(order.getTotalAmount()); // ← 다른 애그리거트 수정
}

// ✅ 이벤트로 분리
public void processOrder(Long orderId) {
    Order order = orderRepo.findById(orderId);
    order.complete(); // Order만 수정
    // → OrderCompletedEvent 발행 → 별도 트랜잭션에서 포인트 적립
}
```

> 부득이하게 두 개 이상을 수정해야 한다면,  
> 응용 서비스에서 직접 처리하되 **명시적으로 의도를 드러내는** 것이 낫습니다.

---

## 3.5 애그리거트 간 참조

### 직접 참조의 문제

```java
// ❌ 직접 참조 — Order가 Member 객체 전체를 들고 있음
public class Order {
    private Member orderer; // 직접 객체 참조
}
```

- `Order`를 로딩할 때 `Member`도 함께 로딩됩니다. (성능)
- `order.getOrderer().getEmail()` 처럼 다른 애그리거트를 쉽게 수정할 수 있게 됩니다. (무결성)
- 두 애그리거트의 결합도가 높아져 확장이 어렵습니다.

### ID 기반 참조

```java
// ✅ ID로만 참조
public class Order {
    private Long ordererId;  // Member의 ID만 보관
}
```

- 필요할 때 별도로 조회합니다. (N+1 문제는 별도의 조회 서비스나 CQRS로 해결)
- 애그리거트 간 결합도가 낮아집니다.
- 서로 다른 서버에 배포(MSA)하더라도 참조 구조를 유지할 수 있습니다.

---

## 3.6 애그리거트를 팩토리로 사용하기

애그리거트가 **다른 애그리거트를 생성하는 팩토리** 역할을 할 수 있습니다.

```java
// 상품 생성 권한이 있는지 Store가 직접 판단
public class Store {
    public Product createProduct(ProductId productId, ...) {
        if (isBlocked()) throw new StoreBlockedException();
        return new Product(productId, getId(), ...);
    }
}
```

```java
// ❌ 응용 서비스에서 판단 — 도메인 규칙이 서비스로 흘러나옴
if (store.isBlocked()) throw new StoreBlockedException();
Product product = new Product(...);

// ✅ 팩토리 메서드 — 생성 권한 검증이 도메인 안에
Product product = store.createProduct(...);
```

생성 로직과 제약 조건이 도메인 객체 안에 남아 있어 응집도가 높아집니다.

---

## 토론 주제

### 💬 Topic 1. 애그리거트 경계를 잘못 잡으면 어떤 일이 생길까?

책은 경계를 "불변 조건을 함께 유지해야 하는 것"으로 잡으라고 합니다.  
그런데 실무에서는 초기에 경계를 잘못 잡는 일이 흔합니다.

- 너무 크게 잡으면: 잠금 범위가 넓어져 동시성 문제, 불필요한 데이터 로딩
- 너무 작게 잡으면: 일관성 보장을 어디서 할지 모호해짐, 서비스 레이어가 비대해짐

**생각해볼 것:**
- 경계를 잘못 잡았다는 것을 어떤 신호로 알아차릴 수 있을까?
- 이미 운영 중인 시스템에서 애그리거트 경계를 바꾸는 것은 어떤 비용이 드는가?
- "한 트랜잭션 = 한 애그리거트" 원칙을 깨야 할 때는 언제인가?

---

### 💬 Topic 2. ID 참조는 항상 옳은가? — 직접 참조 vs ID 참조 트레이드오프

ID 참조는 결합도를 낮추지만, 조회 성능 비용이 생깁니다.

```java
// ID 참조 — 결합도 낮음, 조회 시 추가 쿼리 발생
Long centerId;

// 직접 참조 — JOIN 한 방, 하지만 애그리거트 경계가 흐려짐
Center center;
```

**생각해볼 것:**
- ID 참조를 쓰면 N+1 문제를 어떻게 해결하고 있는가? (별도 조회 서비스? CQRS?)
- 직접 참조를 유지하면서도 경계를 지킬 수 있는 방법이 있을까?
- MSA 환경에서 다른 서비스의 데이터를 참조할 때와 같은 도메인 안 참조를 동일하게 볼 수 있는가?
