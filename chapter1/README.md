# Chapter 1. 도메인 모델 시작하기
---

## 목차

1. [도메인이란?](#11-도메인이란)
2. [도메인 전문가와 개발자 간 지식 공유](#12-도메인-전문가와-개발자-간-지식-공유)
3. [도메인 모델](#13-도메인-모델)
4. [도메인 모델 패턴](#14-도메인-모델-패턴)
5. [도메인 모델 도출](#15-도메인-모델-도출)
6. [엔티티와 밸류](#16-엔티티와-밸류)
7. [도메인 용어와 유비쿼터스 언어](#17-도메인-용어와-유비쿼터스-언어)

---

## 1.1 도메인이란?

**소프트웨어로 해결하고자 하는 문제 영역**

하나의 도메인은 여러 **하위 도메인**으로 나눌 수 있습니다.

```
온라인 서점 (도메인)
├── 상품    ← 상품 정보 관리
├── 주문    ← 주문 처리
├── 회원    ← 회원 관리
├── 결제    ← 결제 처리
├── 배송    ← 외부 업체 연동 가능
└── 혜택    ← 쿠폰/포인트
```

모든 하위 도메인을 직접 구현할 필요는 없어요. 배송처럼 외부 시스템과 연동하는 방식도 있습니다.

---

## 1.2 도메인 전문가와 개발자 간 지식 공유

```
도메인 전문가 → (요구사항 전달) → 개발자 → (분석·개발) → 배포
```

**요구사항을 올바르게 이해하는 것이 중요하다.**

- 도메인 전문가가 개발자에게 직접적으로 지식을 전달해야 합니다.
- 개발자도 도메인 지식을 깊이 이해해야 올바른 모델을 만들 수 있습니다.
- 잘못 전달된 요구사항은 잘못된 구현으로 이어집니다. → **"쓰레기가 들어오면 쓰레기가 나오죠."**

---

## 1.3 도메인 모델

**특정 도메인을 개념적으로 표현한 것**

표현 방식은 다양합니다. (객체 다이어그램, 그래프, 수학 공식 등). 중요한 것은 도메인을 이해하는 데 도움이 되는 형태여야 합니다.

**개념 모델 vs 구현 모델**

| | 개념 모델 | 구현 모델 |
|--|---------|---------|
| 목적 | 순수하게 문제를 분석한 결과물 | DB, 성능, 기술 고려사항 포함 |
| 특징 | 이상적 | 현실적 |

> 처음부터 완벽한 개념 모델을 만들 필요는 없습니다.  
> 개발하면서 도메인에 대한 이해가 깊어질수록 모델도 점진적으로 발전합니다.

---

## 1.4 도메인 모델 패턴

애플리케이션 아키텍처는 **4개의 계층**으로 구성됩니다.

| 계층 | 역할 |
|------|------|
| **표현 (Presentation)** | 사용자 요청 처리 및 응답 반환 |
| **응용 (Application)** | 기능 실행 조율 — 도메인 계층을 조합할 뿐입니다. |
| **도메인 (Domain)** | **핵심 업무 규칙 구현** |
| **인프라 (Infrastructure)** | DB, 외부 시스템 연동 |

핵심 규칙은 반드시 **도메인 계층** 안에 있어야 합니다.

```java
// 배송지 변경 가능 여부 — 도메인 규칙이 도메인 모델 안에 있다
public class Order {
    private boolean isShippingChangeable() {
        return state == OrderState.PAYMENT_WAITING ||
               state == OrderState.PREPARING;
    }

    public void changeShippingInfo(ShippingInfo newShippingInfo) {
        if (!isShippingChangeable())
            throw new IllegalStateException("can't change shipping in " + state);
        this.shippingInfo = newShippingInfo;
    }
}
```

> 규칙이 도메인 안에 있으면, 규칙이 바뀌어도 영향 범위가 도메인 계층으로 한정된다.

---

## 1.5 도메인 모델 도출

요구사항을 분석해 **핵심 구성요소·규칙·기능**을 도출하는 것이 출발점입니다.

**요구사항 예시 (주문 도메인)**

```
- 최소 한 종류 이상의 상품을 주문해야 한다          → OrderLine 최소 1개 필수
- 총 주문 금액 = 각 상품의 (가격 × 수량)의 합       → calculateTotalAmounts()
- 주문 시 배송지 정보를 반드시 지정해야 한다         → ShippingInfo 필수
- 출고 후에는 배송지를 변경할 수 없다               → verifyNotYetShipped()
- 출고 전에는 주문을 취소할 수 있다                 → cancel() + verifyNotYetShipped()
```

```java
public class Order {
    private OrderState state;
    private List<OrderLine> orderLines; // 최소 1개 이상
    private ShippingInfo shippingInfo;  // 필수

    public void changeShippingInfo(ShippingInfo newShipping) {
        verifyNotYetShipped();
        this.shippingInfo = newShipping;
    }

    public void cancel() {
        verifyNotYetShipped();
        this.state = OrderState.CANCELED;
    }

    private void verifyNotYetShipped() {
        if (state != OrderState.PAYMENT_WAITING && state != OrderState.PREPARING)
            throw new IllegalStateException("already shipped");
    }
}

public class OrderLine {
    private Product product;
    private int price;
    private int quantity;

    private int calculateAmounts() {
        return price * quantity;
    }
}
```

> **명사 → 클래스/필드 / 동사 → 메서드 / 제약 조건 → 검증 로직**

---

## 1.6 엔티티와 밸류

### 엔티티 (Entity)

**고유 식별자(ID)를 가지는 객체**

- 식별자가 같으면 같은 엔티티로 판단합니다.
- 속성이 바뀌어도 식별자는 변하지 않습니다.
- `equals()` / `hashCode()`를 식별자 기반으로 구현합니다.

```java
public class Order {
    private String orderNumber; // 식별자

    @Override
    public boolean equals(Object obj) {
        if (!(obj instanceof Order)) return false;
        return this.orderNumber.equals(((Order) obj).orderNumber);
    }
}
```

**식별자 생성 방식:** 규칙 기반 | UUID | 직접 입력 | DB auto increment

---

### 밸류 (Value)

**개념적으로 완전한 하나의 값을 표현하는 객체**

```java
// Before: 의미 단위가 흩어져 있음
class ShippingInfo {
    private String receiverName;
    private String receiverPhoneNumber;
    private String shippingAddress1;
    private String shippingAddress2;
    private String shippingZipcode;
}

// After: 개념 단위로 묶기
class ShippingInfo {
    private Receiver receiver; // 이름 + 전화번호
    private Address address;   // 주소1 + 주소2 + 우편번호
}
```

단일 필드도 밸류 타입으로 만들면 **의미 명확화 + 도메인 기능 추가** 가 가능합니다.

```java
public class Money {
    private int value;

    public Money add(Money money) {
        return new Money(this.value + money.value); // 새 객체 반환 → 불변
    }
    public Money multiply(int multiplier) {
        return new Money(value * multiplier);
    }
}
// int price 보다 Money price 가 의미가 명확합니다.
// 개인적으로 이렇게 작성하는 방법이 상당히 유용하겠다라는 생각을 했어요.
```

> 식별자도 밸류 타입으로 만들 수 있습니다.  
> `OrderNo id` 라고 쓰면 필드명이 `id`여도 타입만 보고 "주문번호"임을 알 수 있습니다.

### set 메서드를 도메인 모델에 넣지 않기

| ❌ set 메서드 | ✅ 의미 있는 메서드 |
|---|---|
| `setShippingInfo()` | `changeShippingInfo()` — "배송지 변경" |
| `setOrderState()` | `completePayment()` — "결제 완료" |

set 메서드는 단순 값 설정 → **도메인 의도가 코드에서 사라진다**

생성자로 완전한 객체를 만들고, 내부 검증은 private 메서드로 처리합니다.

```java
// 생성 시점에 필수 데이터 전달 + 검증
Order order = new Order(orderer, lines, shippingInfo, OrderState.PREPARING);

private void setOrderer(Orderer orderer) {
    if (orderer == null) throw new IllegalArgumentException("no orderer");
    this.orderer = orderer;
}
```

---

## 1.7 도메인 용어와 유비쿼터스 언어

**유비쿼터스 언어(Ubiquitous Language)** = 도메인 전문가·개발자·관계자가 함께 사용하는 공통 언어

활용 범위: **대화 · 문서 · 도메인 모델 · 코드 · 테스트**

```java
// ❌ 기술 중심 표현
public enum OrderStatus { STEP1, STEP2, STEP3 }

// ✅ 도메인 용어 사용
public enum OrderState {
    PAYMENT_WAITING,  // 결제 대기
    PREPARING,        // 상품 준비 중
    SHIPPED,          // 출고 완료
    DELIVERING,       // 배송 중
    DELIVERY_COMPLETED // 배송 완료
}
```

> 코드에 도메인 용어를 그대로 사용하면, 코드만 봐도 업무 흐름을 이해할 수 있습니다.  
> 용어가 달라지면 해석 비용이 생기고 소통이 어려워집니다.
