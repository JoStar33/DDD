# Chapter 5. 스프링 데이터 JPA를 이용한 조회 기능
---

## 목차

1. [조회 모델과 CQRS](#51-조회-모델과-cqrs)
2. [검색 조건과 Specification](#52-검색-조건과-specification)
3. [정렬과 페이징](#53-정렬과-페이징)
4. [동적 인스턴스 생성](#54-동적-인스턴스-생성)
5. [@Subselect](#55-subselect)

---

## 5.1 조회 모델과 CQRS

**CQRS**는 상태를 바꾸는 모델과 데이터를 보여주는 모델을 분리하는 방식입니다.

| 구분 | 목적 | 예시 |
|------|------|------|
| **명령 모델** | 상태 변경, 도메인 규칙 실행 | 회원 가입, 비밀번호 변경, 주문 취소 |
| **조회 모델** | 화면에 필요한 데이터 조회 | 주문 목록, 주문 상세, 검색 결과 |

4장에서 애그리거트 로딩 전략을 다룰 때도 같은 결론이 나왔습니다. **상태 변경**에는 완전한 애그리거트가 필요하지만, **조회 기능**에는 유스케이스에 맞춘 데이터가 더 적합합니다.

조회 요구사항에 맞춰 애그리거트를 억지로 변경하면 도메인 모델이 화면 요구사항에 끌려갑니다. 따라서 명령 모델은 도메인 규칙과 일관성 유지에 집중하고, 조회 모델은 화면/API 응답과 조회 성능에 집중하도록 분리합니다.

```java
public interface OrderSummaryDao {
    List<OrderSummary> findByOrdererId(String ordererId);
}
```

조회 전용 DAO는 도메인 규칙을 실행하기보다 **목록/상세 화면에 필요한 데이터를 효율적으로 가져오는 역할**에 집중합니다. 즉, 조회 모델 분리는 단순한 DTO 분리가 아니라 도메인 모델을 보호하는 설계이기도 합니다.

---

## 5.2 검색 조건과 Specification

검색 조건이 단순하면 메서드 이름만으로 충분합니다.

```java
List<OrderSummary> findByOrdererId(String ordererId);
```

하지만 조건 조합이 늘어나면 `findByOrdererIdAndDateBetweenAndState...` 같은 메서드가 계속 늘어납니다. 이때 사용하는 것이 **Specification**입니다.

```java
public class OrderSummarySpecs {
    public static Specification<OrderSummary> ordererId(String ordererId) {
        return (root, query, cb) -> cb.equal(root.get("ordererId"), ordererId);
    }

    public static Specification<OrderSummary> orderDateBetween(
            LocalDateTime from, LocalDateTime to) {
        return (root, query, cb) -> cb.between(root.get("orderDate"), from, to);
    }
}
```

Spring Data JPA의 `Specification<T>`는 검색 조건 하나를 객체로 표현합니다. 내부적으로는 `toPredicate()`를 통해 JPA Criteria의 `Predicate`, 즉 SQL의 `where` 절에 들어갈 조건으로 변환됩니다. 조건은 `and()`, `or()`, `not()`으로 조합할 수 있습니다.

```java
Specification<OrderSummary> spec = Specification
    .where(OrderSummarySpecs.ordererId("user1"))
    .and(OrderSummarySpecs.orderDateBetween(from, to));

List<OrderSummary> results = orderSummaryDao.findAll(spec);
```

검색 조건이 선택적으로 붙는 경우 스펙 조합이 특히 유용합니다. 주문자, 주문일, 주문 상태처럼 값이 있는 조건만 조합하면 메서드 이름이 폭발적으로 늘어나는 문제를 줄일 수 있습니다.

---

## 5.3 정렬과 페이징

정렬은 두 가지 방식으로 지정할 수 있습니다.

**메서드 이름에 정렬 포함**

```java
List<OrderSummary> findByOrdererIdOrderByNumberDesc(String ordererId);
```

간단하지만 정렬 기준이 늘어나거나 사용자 선택에 따라 바뀌면 메서드 이름이 금방 길어집니다.

**Sort 파라미터 사용**

```java
List<OrderSummary> findByOrdererId(String ordererId, Sort sort);

Sort sort = Sort.by("number").descending()
    .and(Sort.by("orderDate").ascending());
```

페이징은 `Pageable`을 사용합니다.

```java
List<MemberData> findByNameLike(String name, Pageable pageable);

Pageable pageable = PageRequest.of(
    0,
    20,
    Sort.by("orderDate").descending()
);

List<MemberData> members = memberDataDao.findByNameLike("사용자%", pageable);
```

`Pageable`은 Spring Data가 제공하는 페이징 요청 인터페이스이고, `PageRequest`는 대표 구현체입니다. DB가 제공하는 기능은 아니지만, Spring Data JPA가 이를 해석해 최종 SQL의 `order by`, `limit`, `offset` 등에 반영합니다.

```sql
select *
from member
where name like '사용자%'
order by order_date desc
limit 20 offset 0;
```

---

## 5.4 동적 인스턴스 생성

조회 기능은 여러 애그리거트의 데이터를 조합해야 할 때가 많습니다. 이때 엔티티를 전부 로딩한 뒤 조합하면 N+1이나 불필요한 로딩 문제가 생깁니다.

JPQL의 `select new`를 사용하면 조회 결과를 DTO로 바로 만들 수 있습니다.

```java
@Query("""
    select new com.myshop.order.query.dto.OrderView(
        o.number, o.state, m.name, m.id, p.name
    )
    from Order o join o.orderLines ol, Member m, Product p
    where o.orderer.memberId.id = :ordererId
      and o.orderer.memberId.id = m.id
      and index(ol) = 0
      and ol.productId.id = p.id
    order by o.number.number desc
    """)
List<OrderView> findOrderView(String ordererId);
```

장점은 명확합니다.

- 화면에 필요한 필드만 가져옵니다.
- 지연 로딩/즉시 로딩 고민을 줄입니다.
- 조회 유스케이스에 맞는 형태로 데이터를 만들 수 있습니다.

즉, 조회 모델은 도메인 모델의 복사본이 아니라 **화면 요구사항에 맞춘 읽기 전용 모델**입니다.

---

## 5.5 @Subselect

하이버네이트의 `@Subselect`는 쿼리 결과를 `@Entity`처럼 매핑하는 기능입니다. DB View와 비슷하게, 여러 테이블을 조인한 결과를 하나의 조회 전용 엔티티로 다룰 수 있습니다.

```java
@Entity
@Immutable
@Subselect("""
    select o.order_number as number,
           o.orderer_id,
           o.orderer_name,
           o.state,
           p.name as product_name
    from purchase_order o
    join order_line ol on o.order_number = ol.order_number
    join product p on ol.product_id = p.product_id
    where ol.line_idx = 0
    """)
@Synchronize({"purchase_order", "order_line", "product"})
public class OrderSummary {
    @Id
    private String number;
    private String ordererId;
    private String ordererName;
    private String productName;

    protected OrderSummary() {}
}
```

`@Immutable`은 조회 전용임을 명시합니다. 이 애노테이션이 없으면 하이버네이트가 변경 감지를 통해 update를 시도할 수 있습니다.

`@Synchronize`는 관련 테이블에 변경이 있을 때 조회 전에 flush하도록 알려줍니다. 같은 트랜잭션 안에서 주문을 변경한 뒤 `OrderSummary`를 조회하면, 이전 값이 아니라 변경이 반영된 값을 읽도록 돕습니다.

다만 `@Subselect`는 하이버네이트 전용 기능이고, 실제 SQL은 서브쿼리 형태로 실행됩니다. 성능이 중요하거나 DB별 최적화가 필요하면 네이티브 SQL, QueryDSL, MyBatis 같은 선택지도 함께 검토합니다.

---

## 한 장 요약

조회 기능은 도메인 모델을 그대로 노출하는 일이 아니라 **유스케이스에 필요한 읽기 모델을 설계하는 일**입니다.

- 상태 변경은 애그리거트와 리포지터리 중심으로 처리합니다.
- 목록/상세 조회는 조회 전용 DAO와 DTO를 적극적으로 사용합니다.
- 검색 조건 조합은 `Specification`으로 다룹니다.
- 정렬/페이징은 `Sort`, `Pageable`로 표현합니다.
- 복잡한 조회 모델은 `select new`, `@Subselect`, QueryDSL, 네이티브 SQL 등 목적에 맞게 선택합니다.

---

**같이 생각해볼 것:**
- 화면별 DTO를 따로 만들면 중복인가, 명확한 계약인가?
