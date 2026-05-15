# Chapter 8. 애그리거트 트랜잭션 관리
---

## 목차

1. [애그리거트와 트랜잭션](#81-애그리거트와-트랜잭션)
2. [선점 잠금](#82-선점-잠금)
3. [비선점 잠금](#83-비선점-잠금)
4. [오프라인 선점 잠금](#84-오프라인-선점-잠금)

---

## 8.1 애그리거트와 트랜잭션

같은 애그리거트를 두 사용자가 동시에 수정하면 일관성이 깨질 수 있습니다.

예를 들어 운영자가 주문을 배송 상태로 변경하는 동안 고객이 배송지를 변경한다고 해봅시다. 두 트랜잭션은 같은 주문을 다루지만, 각자 별도의 애그리거트 객체를 조회합니다. 운영자 입장에서는 기존 배송지 기준으로 배송을 시작했는데, 고객의 변경까지 함께 반영되면 실제 업무 흐름과 데이터가 어긋납니다.

이런 충돌을 막기 위해 애그리거트 단위의 트랜잭션 관리가 필요합니다.

| 방식 | 핵심 아이디어 |
|------|---------------|
| **선점 잠금** | 먼저 조회한 트랜잭션이 끝날 때까지 다른 트랜잭션을 대기시킨다 |
| **비선점 잠금** | 동시에 접근은 허용하되, 커밋 시점에 버전 충돌을 검사한다 |
| **오프라인 선점 잠금** | 여러 트랜잭션에 걸친 긴 작업 동안 수정 권한을 선점한다 |

---

## 8.2 선점 잠금

**선점 잠금**은 먼저 애그리거트를 구한 스레드가 트랜잭션을 끝낼 때까지 다른 스레드가 같은 애그리거트를 수정하지 못하게 막는 방식입니다.

DBMS의 행 잠금을 이용해 구현하며, JPA에서는 `PESSIMISTIC_WRITE` 잠금 모드를 사용할 수 있습니다.

```java
Order order = entityManager.find(
        Order.class,
        orderNo,
        LockModeType.PESSIMISTIC_WRITE
);
```

스프링 데이터 JPA에서는 `@Lock`을 사용합니다.

```java
public interface OrderRepository extends JpaRepository<Order, OrderNo> {
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("select o from Order o where o.number = :number")
    Optional<Order> findByIdForUpdate(@Param("number") OrderNo number);
}
```

선점 잠금은 강력하지만 대기 시간이 길어질 수 있고, 여러 애그리거트를 서로 다른 순서로 잠그면 교착 상태가 발생할 수 있습니다.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints({
    @QueryHint(name = "javax.persistence.lock.timeout", value = "2000")
})
@Query("select o from Order o where o.number = :number")
Optional<Order> findByIdForUpdate(@Param("number") OrderNo number);
```

따라서 잠금 최대 대기 시간을 지정하고, 여러 대상을 잠글 때는 항상 같은 순서로 잠그는 규칙을 두는 것이 좋습니다.

---

## 8.3 비선점 잠금

**비선점 잠금**은 동시에 접근하는 것을 막지 않고, 변경 내용을 DB에 반영하는 시점에 충돌 여부를 확인합니다. 보통 버전 값을 이용합니다.

```sql
update purchase_order
set version = version + 1,
    state = ?,
    shipping_info = ?
where order_number = ?
  and version = ?
```

다른 트랜잭션이 먼저 수정해서 버전이 바뀌었다면 update 결과가 0건이 되고, 충돌로 처리합니다.

JPA에서는 `@Version`을 사용합니다.

```java
@Entity
@Table(name = "purchase_order")
public class Order {
    @EmbeddedId
    private OrderNo number;

    @Version
    private long version;

    public boolean matchVersion(long version) {
        return this.version == version;
    }
}
```

응용 서비스는 사용자가 조회한 시점의 버전을 요청 값으로 받아 현재 버전과 비교할 수 있습니다.

```java
@Transactional
public void startShipping(StartShippingRequest req) {
    Order order = orderRepository.findById(new OrderNo(req.getOrderNumber()));
    if (order == null) throw new NoOrderException();

    if (!order.matchVersion(req.getVersion())) {
        throw new VersionConflictException();
    }

    order.startShipping();
}
```

표현 영역은 버전 충돌을 사용자에게 알려 다시 조회하거나 재시도하게 합니다.

```java
try {
    startShippingService.startShipping(req);
    return "shippingStarted";
} catch (OptimisticLockingFailureException | VersionConflictException ex) {
    return "startShippingTxConflict";
}
```

애그리거트 루트가 아닌 내부 엔티티만 변경되는 경우 JPA가 루트의 버전을 증가시키지 않을 수 있습니다. 이때는 `OPTIMISTIC_FORCE_INCREMENT`를 사용해 루트 버전을 강제로 증가시킬 수 있습니다.

```java
Order order = entityManager.find(
        Order.class,
        orderNo,
        LockModeType.OPTIMISTIC_FORCE_INCREMENT
);
```

---

## 8.4 오프라인 선점 잠금

**오프라인 선점 잠금**은 한 트랜잭션 안에서 끝나지 않는 긴 수정 흐름에 사용합니다. 예를 들어 수정 폼 조회와 수정 요청은 보통 서로 다른 트랜잭션입니다. 사용자가 폼을 열어둔 동안 다른 사용자의 수정을 막고 싶다면 일반 선점 잠금만으로는 부족합니다.

오프라인 선점 잠금은 수정 폼을 열 때 잠금을 얻고, 수정이 끝나면 잠금을 해제합니다.

```java
public interface LockManager {
    LockId tryLock(String type, String id) throws LockException;
    void checkLock(LockId lockId) throws LockException;
    void releaseLock(LockId lockId) throws LockException;
    void extendLockExpiration(LockId lockId, long inc) throws LockException;
}
```

컨트롤러나 응용 서비스는 잠금을 얻은 뒤 `LockId`를 화면에 전달하고, 수정 요청 시 다시 받아 잠금이 유효한지 확인합니다.

```java
public DataAndLockId getDataWithLock(Long id) {
    LockId lockId = lockManager.tryLock("data", id.toString());
    Data data = dataDao.select(id);
    return new DataAndLockId(data, lockId);
}

public void edit(EditRequest req, LockId lockId) {
    lockManager.checkLock(lockId);
    updateData(req);
    lockManager.releaseLock(lockId);
}
```

사용자가 수정하지 않고 브라우저를 닫을 수 있으므로 잠금에는 유효 시간이 필요합니다. 유효 시간이 지나면 잠금을 만료시켜 다른 사용자가 다시 잠금을 얻을 수 있어야 합니다.

DB를 사용한다면 잠금 대상과 잠금 ID, 만료 시간을 저장하는 테이블을 둘 수 있습니다.

```sql
create table locks (
    type varchar(255),
    id varchar(255),
    lockid varchar(255),
    expiration_time datetime,
    primary key (type, id)
);
```

오프라인 선점 잠금은 충돌을 미리 막을 수 있지만, 잠금 만료와 연장, 비정상 종료 처리를 함께 설계해야 합니다.

---

## 토론 주제

### Topic 1. 충돌은 미리 막을 것인가, 나중에 감지할 것인가?

선점 잠금은 충돌을 미리 막지만 대기와 교착 상태 비용이 있고, 비선점 잠금은 처리량은 좋지만 사용자가 나중에 충돌을 마주합니다.

**같이 생각해볼 것:**
- 각자 경험한 시스템에서 실제로 충돌이 문제가 되었던 기능은 무엇이었는가?
- 선점 잠금과 비선점 잠금 중 어떤 방식을 선택했고, 그 판단 기준은 무엇이었는가?
- 사용자가 충돌을 직접 해결해야 하는 UX를 어떻게 설계했는가?

---

### Topic 2. 긴 편집 흐름의 잠금은 현실적으로 운영 가능한가?

오프라인 선점 잠금은 문서, 설정, 상품 정보처럼 오래 열어두고 수정하는 화면에서 유용하지만, 잠금 만료와 강제 해제 같은 운영 문제가 함께 따라옵니다.

**같이 생각해볼 것:**
- 긴 편집 화면에서 "누가 수정 중인지"를 보여준 경험이 있는가?
- 잠금을 자동 만료시키는 기준을 어떻게 잡았는가?
- 잠금보다 자동 병합, 변경 이력, 재시도 UX가 더 나았던 경우는 없었는가?
