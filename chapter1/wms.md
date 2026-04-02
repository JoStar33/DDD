# Chapter 1. 도메인 모델 시작하기 —  WMS 적용

> **기준 프로젝트:** WMS (Spring Boot 3.2 / Java 17)  

---

## 1. 우리 서비스의 도메인

###  WMS란?

제가 다니는 회사는 반려동물 용품을 판매하는 이커머스 플랫폼입니다.  
고객이 앱에서 주문을 넣으면, 창고에서 상품을 꺼내 포장하고 배송하는 물류 흐름 전체를  
관리하는 시스템이 **WMS(Warehouse Management System)** 입니다.

### 탄생 배경

기존에는 물류 데이터가 3개 시스템에 분산되어 있었습니다.

| 시스템 | 역할 | 문제 |
|--------|------|------|
| Pet-friends DB | 주문·상품 원천 데이터 | |
| OTOO (외부 물류 솔루션) | 창고 입출고 처리 | 재고 기준이 달라 수시로 불일치 발생 |
| CJ 시스템 | 택배 배송 연동 | |

> "어느 창고에 재고가 얼마나 있는가?" 라는 질문에 시스템마다 답이 달랐습니다.  
> **WMS = 세 시스템을 통합해 재고의 단일 진실 공급원을 만드는 것입니다.**

### 무슨 일을 하는 서비스인가

```
고객 주문 발생
    │
    ▼
[출고 등록] ──────────────── outbound 도메인
    │ 창고에서 상품 집품·포장
    ▼
[배송사 전송] ─────────────── CJ/TeamFresh 외부 연동 (최근에는 물류창고 내재화를 하고있어요.)
    │
    ▼
[출고 완료] → 재고 차감 ───── stock 도메인

공급사 납품
    │
    ▼
[입고 등록] ──────────────── inbound 도메인
    │ 상품 수령·검수
    ▼
[입고 완료] → 재고 증가 ───── stock 도메인

고객 반품 요청
    │
    ▼
[반품 처리] ──────────────── reverse 도메인
```

### 하위 도메인 구조

```
wms
├── inbound      (입고)      ← 공급사 → 창고, 상품 수령·검수
├── outbound     (출고)      ← 창고 → 고객, 주문 집품·배송
├── stock        (재고)      ← 창고별 상품 재고 현황 (5가지 상태)
├── reverse      (반품)      ← 고객 → 창고, 반품 수령·처리
├── processing   (유통가공)  ← 창고 간 이동, 재포장 등 가공 처리
└── standard     (표준정보)  ← 창고(Center), 상품, 사용자 마스터
```

각 도메인은 `domain / application / infrastructure` 3개 패키지로 분리됩니다.  
배송은 CJ/TeamFresh와 연동, 재고 대사는 OTOO와 비교 → **직접 구현하지 않는 도메인**의 예

---

## 2. 도메인 모델 패턴 — 4계층 매핑

| 계층 | WMS 구조 | 예시 |
|------|---------|------|
| **표현** | Controller | HTTP 요청·응답 |
| **응용** | `*.application` Service | 유스케이스 조율 |
| **도메인** | `*.domain` Entity / VO / Enum | **업무 규칙 직접 구현** |
| **인프라** | `*.infrastructure` Repository, Kafka | DB, CJ 시스템 연동 |

핵심: 업무 규칙은 도메인 계층 안에 있어야 합니다.

```java
// Inbound.java (domain 계층) — "완료일은 저번달 1일 ~ 오늘" 규칙이 엔티티 안에
public void complete(LocalDate completeDay) {
    validateCompleteDay(completeDay);  // 업무 규칙 검증
    this.statusCode = InboundStatusCode.COMPLETED;
    this.completeDay = completeDay;
}
```

---

## 3. 엔티티와 밸류 — 우리 코드에서 찾기

### 엔티티 — 식별자를 가진 객체

| 엔티티 | 식별자 |
|--------|--------|
| `Inbound` | `id`, `wikey` |
| `Outbound` | `id`, `wokey` |
| `Stock` | `id` (Long) — 비즈니스 유니크 키는 `getUniqueKey()` = `"centerId-productItemId"` |
| `Reverse` | `id`, `reverseCode` |
| `Center` | `id`, `centerCode` |

### 밸류 타입 — 실제 코드의 `@Embeddable`

#### `Invoice` — 운송장 (배송사 + 송장번호는 항상 함께)

```java
@Embeddable
public class Invoice {
    private Parcel parcel;  // 배송사
    private String number;  // 송장번호

    public boolean isNotEmpty() {
        return parcel != null && StringUtils.hasText(number);
    }
}
```

#### `Orderer` & `Delivery` — 출고의 주문자·배송지

```java
@Embeddable
public class Orderer {
    private String recipient;     // 수령인
    private String mainMobileNo;  // 주 연락처
    private String name;          // 주문자명
    // ...
}

@Embeddable
public class Delivery {
    private String zipcode;
    private String address;
    private String detailAddress;
    private String enterMethod;   // 진입 방법
    // ...
}

// Outbound 엔티티에서 사용 — 12개 필드가 2개 개념으로
@Entity
public class Outbound {
    private Orderer orderer;
    private Delivery delivery;
}
```

#### `StockCountByStatus` — 재고의 핵심 밸류 타입

재고는 단순 숫자가 아니라 **5가지 상태의 묶음**이라고 봐야합니다.

```java
@Embeddable
public class StockCountByStatus {
    private Integer availableQuantity;   // 가용 (출고 가능)
    private Integer allocationQuantity;  // 할당 (출고예정으로 묶임)
    private Integer stopQuantity;        // 중지
    private Integer damageQuantity;      // 손상
    private Integer standbyQuantity;     // 대기 (입고예정으로 들어올 수량)

    // 상태 전이 로직을 밸류 타입이 직접 가진다 — 새 인스턴스 반환(불변)
    public StockCountByStatus completeInbound(Integer availQty, Integer receivedQty) {
        return StockCountByStatus.builder()
                .availableQuantity(this.availableQuantity + receivedQty)
                .standbyQuantity(this.standbyQuantity - availQty)
                // ...
                .build();
    }

    public StockCountByStatus registerOutbound(Integer qty) {
        return StockCountByStatus.builder()
                .availableQuantity(this.availableQuantity - qty)
                .allocationQuantity(this.allocationQuantity + qty)
                // ...
                .build();
    }
}
```

**재고 흐름 정리:**
```
입고예정 등록  → registerInbound()  → standby +
입고완료       → completeInbound()  → available +,  standby -
출고예정 등록  → registerOutbound() → allocation +, available -
출고완료       → completeOutbound() → allocation -
출고취소       → cancelOutbound()   → allocation -, available +
```

---

## 4. 도메인 모델에 set 메서드 넣지 않기

### 의도를 드러내는 메서드

```java
// ❌ 가상의 나쁜 예시 (실제 코드에 없음) — 상태만 바꿀 뿐, 검증 로직이 서비스로 흩어짐
public void setStatusCode(InboundStatusCode statusCode) { ... }

// ✅ 실제 코드 — 도메인 의도 + 업무 규칙이 메서드 안에 함께
public void complete(LocalDate completeDay) {
    validateCompleteDay(completeDay);             // 완료일 범위 검증
    this.statusCode = InboundStatusCode.COMPLETED;
    this.completeDay = completeDay;
}

public void send() {
    this.statusCode = InboundStatusCode.SENT;
    this.lockedTimeBeforeSending = LocalDateTime.now();  // 20분 잠금 시작
}
```

### `StockCountByStatus` — set이 없는 불변 밸류

```java
// ❌ 가상의 나쁜 예시 (실제 코드에 없음) — 계산 로직이 서비스에 노출됨
stock.getStockCountByStatus().setAvailableQuantity(...);
stock.getStockCountByStatus().setStandbyQuantity(...);

// ✅ 실제 코드 — 로직이 밸류 타입 안에 캡슐화됨
StockCountByStatus updated = stock.getStockCountByStatus()
                                  .completeInbound(availQty, receivedQty);
stock.applyStockCount(updated);
```

### 열거형이 허용 액션을 판단

```java
// ❌ 가상의 나쁜 예시 (실제 코드에 없음) — 서비스에서 카테고리별 분기
if (category == MOVE || category == ADJUST || ...) throw new BadRequestException(...);

// ✅ 실제 코드 — 도메인 규칙이 열거형 안에
public enum InboundViewAction {
    COMPLETE;
    public boolean isNotAllowedBy(InboundCategory category) {
        return NOT_ALLOWED_CATEGORIES.contains(category);
    }
}
```

---

## 핵심 요약

| DDD 개념 |  WMS |
|----------|---------------|
| **도메인** | inbound / outbound / stock / reverse / processing / standard |
| **엔티티** | `Inbound`(wikey), `Outbound`(wokey), `Stock`, `Center` 등 |
| **밸류 타입** | `Invoice`, `Orderer`, `Delivery`, `StockCountByStatus` (`@Embeddable`) |
| **도메인 규칙 위치** | 엔티티 메서드 안 (`complete()`, `send()`, `canAdjust()`) |
| **set 금지** | 의미 있는 메서드로 상태 변경, 밸류는 불변으로 설계 |
