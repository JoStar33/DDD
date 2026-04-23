# Chapter 4. 리포지터리와 모델 구현 — WMS 적용

> **기준 프로젝트:** WMS (Spring Boot 3.2 / Java 17 / Spring Data JPA + QueryDSL)

---

## 1. 리포지터리 구조 — JPA + QueryDSL 분리

단순 조회는 Query Method로, 복잡한 동적 조회는 QueryDSL로 처리합니다. 두 구현을 한 인터페이스로 묶어 서비스는 QueryDSL의 존재를 몰라도 됩니다.

```java
public interface InboundRepository
        extends JpaRepository<Inbound, Long>, InboundRepositoryCustom {
    Optional<Inbound> findByIdAndActivated(Long id, ActiveYN activated);
}

public interface InboundRepositoryCustom {
    List<InboundResponseDto> selectInboundList(InboundSearchDto dto);
    Long selectInboundCount(InboundSearchDto dto);
}
// InboundRepositoryCustomImpl이 QueryDSL로 구현
```

---

## 2. QueryDSL — DTO 직접 조회

엔티티 대신 `QInboundResponseDto` 프로젝션으로 필요한 컬럼만 가져옵니다. `eqNumber()`, `betweenDay()` 같은 null-safe 헬퍼로 동적 조건을 처리합니다.

```java
return queryFactory
    .select(new QInboundResponseDto(
        inbound.id, inbound.wikey, inbound.statusCode,
        center.name, vendor.displayName,
        inboundItem.availableQuantity.sum()
    ))
    .from(inbound)
    .leftJoin(center).on(center.id.eq(inbound.centerId))
    .where(
        betweenDay(inbound.availableDay, dto.getStartDay(), dto.getEndDay()),
        eqNumber(center.id, dto.getCenterId()),  // null이면 조건 제외
        inbound.activated.eq(ActiveYN.Y)
    )
    .fetch();
```

---

## 3. @Embeddable 매핑

**StockCountByStatus** — 재고 5가지 상태(가용·배분·정지·파손·대기)를 밸류 타입으로 묶어 `WAREHOUSE_STOCK` 테이블에 flat하게 저장합니다.

```java
@Embeddable
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class StockCountByStatus {
    private Long productItemId;
    private Integer availableQuantity;
    private Integer allocationQuantity;
    private Integer stopQuantity;
    private Integer damageQuantity;
    private Integer standbyQuantity;
}
```

**Orderer / Delivery** — 같은 테이블에 두 밸류가 함께 들어갈 때 `@AttributeOverrides`로 컬럼명 충돌을 방지합니다.

```java
@Embedded
@AttributeOverrides({
    @AttributeOverride(name = "recipient",    column = @Column(name = "recipient")),
    @AttributeOverride(name = "mainMobileNo", column = @Column(name = "main_mobile_no")),
    @AttributeOverride(name = "name",         column = @Column(name = "orderer_name"))
})
private Orderer orderer;
```

---

## 4. 영속성 전파 — CascadeType.ALL + 논리 삭제

`InboundItem`은 `CascadeType.ALL`로 `Inbound`와 생명주기를 맞춥니다. WMS는 물리 삭제 대신 **논리 삭제**(`activated = N`)를 사용하므로 `orphanRemoval`은 쓰지 않습니다. 루트의 `delete()` 메서드가 하위 항목들도 직접 비활성화합니다.

```java
public void delete() {
    verifyDeleted();
    this.activated = ActiveYN.N;
    for (InboundItem item : this.getActivatedItems()) {
        item.delete();
    }
}
```

---

## 5. 식별자 생성 — GenerateUtil

DB PK(`id`, `IDENTITY` 전략)와 비즈니스 식별자(`wikey`)를 분리합니다. `wikey`는 `날짜(yyMMdd) + 5자리 시퀀스` 조합(예: `WI240115-00001`)으로 생성합니다.

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public String generateWikey() {
    String date = DateUtil.getDateToStringFormat(LocalDate.now(), "yyMMdd");
    return generateCode.generateCodes(1, "WI" + date, "INBOUND" + date, FORMAT).get(0);
}
```

`REQUIRES_NEW` → 외부 트랜잭션 롤백과 무관하게 시퀀스가 소비됩니다 (결번 허용).

---

## 6. BaseEntity — 공통 필드 자동화

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public class BaseEntity {
    protected ActiveYN activated;
    @CreatedDate  protected LocalDateTime createdAt;
    @LastModifiedDate protected LocalDateTime updatedAt;
    @CreatedBy    protected Long createdUserId;
    @LastModifiedBy protected Long updatedUserId;
}
```

Spring Security 컨텍스트에서 `createdUserId`, `updatedUserId`를 자동 주입합니다.

---

## 7. 도메인에 JPA 어노테이션 — WMS의 선택

엄밀히는 DIP 위반이지만, JPA 교체 가능성이 낮으므로 **생산성을 우선**한 실용적 선택입니다.

| 구분 | 분리 방식 | WMS 방식 |
|------|---------|---------|
| 도메인 순수성 | O | X |
| 클래스 수 | 2배 | 그대로 |
| 변환 코드 | 필요 | 불필요 |

---

## 핵심 요약

| DDD 개념 | WMS 구현 |
|----------|---------|
| **리포지터리 분리** | `JpaRepository` + `RepositoryCustom` (QueryDSL) |
| **@Embeddable** | `StockCountByStatus`, `Orderer`, `Delivery` |
| **영속성 전파** | `CascadeType.ALL` + 논리 삭제 |
| **기본 생성자** | `@NoArgsConstructor(PROTECTED)` |
| **식별자 생성** | `GenerateUtil` — 날짜+시퀀스, `REQUIRES_NEW` |
| **공통 필드** | `BaseEntity` (`@MappedSuperclass`) + JPA Auditing |

---

## 토론 주제

### 💬 Topic 1. `REQUIRES_NEW`로 식별자를 생성하면 결번이 발생한다 — 괜찮은가?

외부 트랜잭션이 롤백되어도 시퀀스는 소비되어 `WI240115-00003` 같은 번호가 영구 결번됩니다.

**생각해볼 것:**
- 입고번호에 결번이 생겨도 비즈니스에 문제가 없는가?
- DB 시퀀스 방식 대신 UUID 또는 Snowflake ID를 쓰면 어떤 트레이드오프가 있는가?
- 결번 없는 채번이 정말 필요하다면 어떻게 구현할 수 있을까?

### 💬 Topic 2. QueryDSL이 DTO를 직접 반환하는 것 — 레이어 경계 위반인가?

리포지터리가 `InboundResponseDto`를 반환하면 인프라 계층이 응용/표현 계층의 DTO를 알게 됩니다. 성능(불필요한 컬럼 로딩 없음) 측면에서는 유리하지만 의존 방향이 역전됩니다.

**생각해볼 것:**
- 리포지터리가 `ResponseDto`를 반환하는 것은 레이어 원칙 위반인가?
- 조회 전용 모델을 도메인 모델과 완전 분리(CQRS)하면 구조가 어떻게 달라지는가?
