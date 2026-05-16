# Chapter 7. 도메인 서비스
---
---

## 7.1 여러 애그리거트가 필요한 기능

도메인 기능을 구현하다 보면 한 애그리거트만으로 처리하기 애매한 경우가 있습니다. 대표적인 예가 **결제 금액 계산**입니다.

결제 금액을 계산하려면 여러 애그리거트의 정보가 필요합니다.

| 애그리거트 | 필요한 정보 |
|------------|-------------|
| **상품** | 가격, 배송비 |
| **주문** | 상품별 구매 개수 |
| **쿠폰** | 할인 금액, 할인율, 중복 사용 여부, 적용 조건 |
| **회원** | 등급별 추가 할인 |

이 계산을 `Order` 애그리거트에 억지로 넣으면 주문이 할인 정책까지 떠안게 됩니다.

```java
public class Order {
    private List<OrderLine> orderLines;
    private List<Coupon> usedCoupons;
    private Orderer orderer;

    private Money calculatePayAmounts() {
        Money totalAmounts = calculateTotalAmounts();
        Money couponDiscounts = calculateCouponDiscounts();
        Money membershipDiscount = calculateMembershipDiscount(orderer.getGrade());

        return totalAmounts.minus(couponDiscounts).minus(membershipDiscount);
    }
}
```

특별 할인 정책이 추가될 때마다 `Order`를 수정해야 한다면, 애그리거트가 자신의 책임 범위를 넘어선 것입니다. 이렇게 한 애그리거트에 넣기 어색한 도메인 개념은 별도 도메인 서비스로 드러내는 편이 좋습니다.

---

## 7.2 도메인 서비스

**도메인 서비스**는 도메인 영역에 위치하며, 특정 애그리거트나 밸류에 넣기 어려운 도메인 로직을 표현합니다.

주로 다음 경우에 사용합니다.

- 여러 애그리거트가 필요한 계산 로직
- 한 애그리거트에 넣으면 책임이 어색해지는 도메인 규칙
- 외부 시스템 연동이 필요하지만 도메인 의미를 갖는 기능

도메인 서비스는 엔티티나 밸류와 달리 **상태 없이 로직만 구현**합니다. 필요한 값은 필드로 들고 있기보다 메서드 파라미터로 전달받습니다.

```java
public class DiscountCalculationService {
    public Money calculateDiscountAmounts(
            List<OrderLine> orderLines,
            List<Coupon> coupons,
            MemberGrade grade
    ) {
        Money couponDiscounts = coupons.stream()
                .map(coupon -> calculateDiscount(coupon, orderLines))
                .reduce(Money.ZERO, Money::add);

        Money membershipDiscount = calculateDiscount(grade);
        return couponDiscounts.add(membershipDiscount);
    }

    private Money calculateDiscount(Coupon coupon, List<OrderLine> orderLines) {
        ...
    }

    private Money calculateDiscount(MemberGrade grade) {
        ...
    }
}
```

도메인 서비스 이름은 기술이 아니라 도메인 개념을 드러내야 합니다. `DiscountCalculationService`는 단순 계산 유틸이 아니라 "할인 금액 계산"이라는 도메인 기능을 표현합니다.

### 응용 서비스와 도메인 서비스 비교

| 구분 | 응용 서비스 | 도메인 서비스 |
|------|-------------|---------------|
| **위치** | 응용 영역 | 도메인 영역 |
| **역할** | 유스케이스 흐름 제어 | 특정 애그리거트에 넣기 어려운 도메인 로직 |
| **주요 책임** | 조회, 권한 확인, 트랜잭션, 저장 | 계산, 판단, 애그리거트 상태 변경 |
| **도메인 규칙** | 직접 구현하지 않음 | 직접 구현함 |
| **예시** | 주문하기, 설문 생성하기 | 할인 금액 계산, 계좌 이체 |

응용 서비스는 "무엇을 어떤 순서로 실행할지"를 다루고, 도메인 서비스는 "도메인 규칙 자체"를 다룹니다.

---

## 7.3 도메인 서비스 사용 방식

도메인 서비스는 애그리거트가 사용할 수도 있고, 응용 서비스가 직접 사용할 수도 있습니다.

애그리거트가 도메인 서비스를 사용한다면, 응용 서비스가 메서드 실행 시점에 도메인 서비스를 전달합니다.

```java
public class Order {
    public void calculateAmounts(
            DiscountCalculationService discountService,
            MemberGrade grade
    ) {
        Money totalAmounts = getTotalAmounts();
        Money discountAmounts = discountService.calculateDiscountAmounts(
                orderLines, usedCoupons, grade
        );

        this.paymentAmounts = totalAmounts.minus(discountAmounts);
    }
}
```

```java
@Transactional
public OrderNo placeOrder(OrderRequest req) {
    Member member = findMember(req.getOrdererId());
    Order order = new Order(req.getOrderLines(), req.getCoupons());

    order.calculateAmounts(discountCalculationService, member.getGrade());
    orderRepository.save(order);

    return order.getNumber();
}
```

여기서 주의할 점은 **도메인 서비스를 애그리거트 필드로 주입하지 않는 것**입니다. 도메인 객체의 필드는 모델의 상태를 표현해야 합니다. `DiscountCalculationService`는 주문의 상태도 아니고 저장 대상도 아니며, 일부 기능에서만 필요합니다. 따라서 생성자 주입이나 필드 주입보다 메서드 파라미터로 전달하는 방식이 더 자연스럽습니다.

반대로 도메인 서비스가 애그리거트를 인자로 받아 기능을 실행할 수도 있습니다.

```java
public class TransferService {
    public void transfer(Account from, Account to, Money amounts) {
        from.withdraw(amounts);
        to.credit(amounts);
    }
}
```

계좌 이체는 두 계좌 애그리거트가 함께 관여하는 도메인 기능입니다. 이때 응용 서비스는 두 계좌를 조회하고, 트랜잭션 안에서 `TransferService`를 실행합니다.

```java
@Transactional
public void transfer(TransferRequest req) {
    Account from = accountRepository.findById(req.getFromAccountId());
    Account to = accountRepository.findById(req.getToAccountId());

    transferService.transfer(from, to, req.getAmounts());
}
```

도메인 서비스는 도메인 로직을 수행하지만 트랜잭션을 관리하지는 않습니다. 트랜잭션 처리는 응용 서비스의 책임이기 때문이죠.

---

## 7.4 외부 시스템 연동과 도메인 서비스

외부 시스템이나 다른 도메인과의 연동도 도메인 서비스가 될 수 있습니다.

예를 들어 설문 조사 시스템이 사용자 역할 관리 시스템을 호출해 "설문 생성 권한이 있는가"를 확인해야 한다고 해봅시다. 기술적으로는 HTTP API 호출이지만, 설문 도메인 입장에서는 생성 권한 확인이라는 도메인 규칙입니다.

```java
public interface SurveyPermissionChecker {
    boolean hasUserCreationPermission(String userId);
}
```

인터페이스 이름은 "역할 관리 API 호출"이 아니라 "설문 생성 권한 확인"이라는 도메인 의미를 담아야 합니다.

```java
public class CreateSurveyService {
    private SurveyPermissionChecker permissionChecker;

    public Long createSurvey(CreateSurveyRequest req) {
        validate(req);

        if (!permissionChecker.hasUserCreationPermission(req.getRequestorId())) {
            throw new NoPermissionException();
        }

        Survey survey = new Survey(req.getTitle(), req.getQuestions());
        surveyRepository.save(survey);
        return survey.getId();
    }
}
```

`SurveyPermissionChecker`의 구현체는 인프라스트럭처 영역에 둡니다. 도메인 영역은 도메인 의미를 담은 인터페이스에 의존하고, 실제 API 호출 방식은 인프라스트럭처가 맡습니다.

---

## 7.5 패키지 위치와 인터페이스 분리

도메인 서비스는 도메인 로직을 표현하므로 다른 도메인 구성요소와 같은 패키지에 둡니다. 예를 들어 주문 금액 계산 서비스라면 주문 애그리거트와 같은 도메인 패키지에 위치하는 식입니다.

```text
order
 ├─ domain
 │   ├─ Order.java
 │   ├─ OrderLine.java
 │   └─ DiscountCalculationService.java
 └─ application
     └─ PlaceOrderService.java
```

도메인 서비스가 많아지거나 성격별로 구분하고 싶다면 `domain.service` 같은 하위 패키지를 둘 수 있습니다. 중요한 것은 응용 서비스나 인프라스트럭처가 아니라 **도메인 영역에 둔다**는 점입니다.

도메인 서비스의 로직이 고정되어 있으면 클래스로 바로 구현해도 됩니다. 하지만 외부 시스템, 룰 엔진, 별도 계산 엔진처럼 특정 구현 기술에 의존한다면 인터페이스와 구현을 분리합니다.

| 상황 | 위치 |
|------|------|
| 도메인 서비스 인터페이스 | 도메인 영역 |
| 순수 계산 로직 구현체 | 도메인 영역 |
| 외부 API/룰 엔진 연동 구현체 | 인프라스트럭처 영역 |

이렇게 분리하면 도메인 영역이 특정 기술에 종속되지 않고, 테스트할 때도 대체 구현을 사용하기 쉽습니다.

---

## 토론 주제

### Topic 1. 도메인 서비스는 설계를 맑게 하는가, 흐리게 하는가?

도메인 서비스는 여러 애그리거트에 걸친 규칙을 표현하기 좋지만, 남용하면 도메인 모델이 빈약해지고 서비스 클래스만 커질 수 있습니다. 회사나 팀마다 이 경계를 판단하는 기준도 꽤 다릅니다.

**같이 생각해볼 것:**
- 어떤 로직을 엔티티/밸류에 두고, 어떤 로직을 도메인 서비스로 분리했는가?
- 도메인 서비스가 단순 유틸리티 클래스처럼 변했던 경험이 있는가?
- 서비스가 많아질 때 도메인 개념이 더 선명해졌는가, 오히려 흩어졌는가?

---

### Topic 2. 도메인 서비스는 어디에 두어야 하는가?

도메인 서비스는 이름은 서비스지만 응용 서비스와 다르고, 구현 방식에 따라 도메인 영역과 인프라스트럭처 영역 사이에 걸칠 수도 있습니다.

**같이 생각해볼 것:**
- 도메인 서비스 클래스를 애그리거트와 같은 패키지에 두었는가, 별도 service 패키지에 두었는가?
- 외부 시스템 연동이 필요한 도메인 서비스를 인터페이스로 분리한 경험이 있는가?