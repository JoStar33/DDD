# Chapter 6. 응용 서비스와 표현 영역
---

## 목차

1. [표현 영역과 응용 영역](#61-표현-영역과-응용-영역)
2. [응용 서비스의 역할](#62-응용-서비스의-역할)
3. [응용 서비스 구현](#63-응용-서비스-구현)
4. [표현 영역](#64-표현-영역)
5. [값 검증](#65-값-검증)
6. [조회 전용 기능과 응용 서비스](#66-조회-전용-기능과-응용-서비스)

---

## 6.1 표현 영역과 응용 영역

도메인이 제 기능을 하려면 사용자의 요청을 도메인 모델까지 전달하는 연결 지점이 필요합니다. 이 연결 지점을 담당하는 계층이 **표현 영역**과 **응용 영역**입니다.

| 영역 | 역할 |
|------|------|
| **표현 영역** | HTTP 요청, 폼 값, 쿠키, 세션 등을 해석하고 응답을 만든다 |
| **응용 영역** | 사용자가 요청한 기능을 실행하기 위해 도메인 객체의 흐름을 제어한다 |
| **도메인 영역** | 핵심 규칙과 상태 변경 로직을 구현한다 |

표현 영역은 사용자가 어떤 기능을 실행하려는지 판단하고, 요청 데이터를 응용 서비스가 이해할 수 있는 형태로 변환합니다.

```java
@PostMapping("/member/join")
public ModelAndView join(HttpServletRequest request) {
    String email = request.getParameter("email");
    String password = request.getParameter("password");

    JoinRequest joinReq = new JoinRequest(email, password);
    joinService.join(joinReq);

    return new ModelAndView("member/joinComplete");
}
```

여기서 컨트롤러가 회원 가입 규칙을 직접 처리하지 않습니다. 표현 영역은 요청을 해석하고 변환할 뿐이고, 실제 기능 실행은 응용 서비스에 위임합니다.

---

## 6.2 응용 서비스의 역할

응용 서비스는 **사용자가 요청한 기능을 실행하는 진입점**입니다. 주로 리포지터리에서 애그리거트를 조회하고, 애그리거트의 도메인 기능을 실행하고, 필요하면 결과를 반환합니다.

```java
public class ChangeShippingService {
    private OrderRepository orderRepository;

    @Transactional
    public void changeShipping(ChangeShippingRequest req) {
        Order order = orderRepository.findById(req.getOrderNo());
        if (order == null) throw new NoOrderException(req.getOrderNo());

        order.changeShippingInfo(req.getShippingInfo());
    }
}
```

응용 서비스의 흐름은 보통 단순합니다.

1. 리포지터리에서 애그리거트를 구합니다.
2. 애그리거트의 도메인 기능을 실행합니다.
3. 트랜잭션 안에서 변경 결과를 저장합니다.
4. 필요한 결과를 표현 영역에 반환합니다.

새 애그리거트를 생성하는 경우도 비슷합니다.

```java
@Transactional
public OrderNo placeOrder(OrderRequest req) {
    checkOrderRequest(req);

    Order order = new Order(
        orderRepository.nextOrderNo(),
        req.getOrderer(),
        req.getOrderLines(),
        req.getShippingInfo()
    );

    orderRepository.save(order);
    return order.getNumber();
}
```

### 도메인 로직 넣지 않기

응용 서비스가 복잡해진다면 도메인 로직이 응용 서비스로 새고 있는지 의심해야 합니다. 응용 서비스는 흐름을 제어하는 역할이지, 도메인 규칙을 구현하는 곳이 아닙니다.

```java
// ✅ 도메인 규칙은 Member가 판단
public class ChangePasswordService {
    @Transactional
    public void changePassword(String memberId, String oldPw, String newPw) {
        Member member = memberRepository.findById(memberId);
        if (member == null) throw new NoMemberException(memberId);

        member.changePassword(oldPw, newPw);
    }
}
```

비밀번호 변경에서 "현재 비밀번호가 맞는가"는 회원 도메인의 규칙입니다. 따라서 `Member` 안에 있어야 합니다.

```java
public class Member {
    public void changePassword(String oldPw, String newPw) {
        if (!matchPassword(oldPw)) throw new BadPasswordException();
        setPassword(newPw);
    }

    public boolean matchPassword(String password) {
        return passwordEncoder.matches(password);
    }
}
```

반대로 응용 서비스가 직접 비밀번호를 비교하고 값을 변경하면 문제가 생깁니다.

```java
// ❌ 도메인 규칙이 응용 서비스로 흘러나옴
if (!passwordEncoder.matches(oldPw, member.getPassword())) {
    throw new BadPasswordException();
}
member.setPassword(newPw);
```

이렇게 되면 같은 규칙이 회원 탈퇴, 비밀번호 초기화 같은 다른 서비스에도 중복될 수 있습니다. 도메인 규칙은 도메인 영역에 모아야 응집도가 높아지고 변경 범위가 줄어듭니다.

---

## 6.3 응용 서비스 구현

응용 서비스는 표현 영역과 도메인 영역 사이의 **Facade** 역할을 합니다. 구현 자체는 단순하지만, 클래스 크기와 의존 방향을 주의해야 합니다.

### 응용 서비스의 크기

응용 서비스를 구성하는 방식은 크게 두 가지입니다.

| 방식 | 장점 | 단점 |
|------|------|------|
| 한 도메인의 기능을 한 서비스에 모음 | 공통 로직 재사용이 쉽다 | 클래스가 커지고 관련 없는 기능이 얽히기 쉽다 |
| 기능별 서비스로 분리 | 클래스 책임이 작고 의존성이 명확하다 | 클래스 수가 늘고 공통 로직 중복이 생길 수 있다 |

```java
// 한 클래스에 여러 기능이 모이는 방식
public class MemberService {
    public void join(MemberJoinRequest req) { ... }
    public void changePassword(String memberId, String curPw, String newPw) { ... }
    public void initializePassword(String memberId) { ... }
    public void leave(String memberId, String curPw) { ... }
}
```

작은 시스템에서는 편하지만, 기능이 늘어나면 한 클래스에 계속 코드를 끼워 넣기 쉽습니다. 기능별로 분리하면 클래스 수는 늘지만 각 서비스가 필요한 의존성만 갖게 됩니다.

```java
public class ChangePasswordService {
    private MemberRepository memberRepository;

    @Transactional
    public void changePassword(String memberId, String curPw, String newPw) {
        Member member = findExistingMember(memberId);
        member.changePassword(curPw, newPw);
    }
}
```

공통 로직이 반복된다면 별도 헬퍼나 도메인 서비스로 분리할 수 있습니다. 단, 공통화한다는 이유로 도메인 규칙을 응용 서비스 보조 클래스로 빼내면 안 됩니다.

### 표현 영역에 의존하지 않기

응용 서비스의 파라미터나 반환 타입에 `HttpServletRequest`, `HttpSession`, `Model`, `Errors` 같은 표현 영역 타입을 사용하면 안 됩니다.

```java
// ❌ 응용 서비스가 표현 기술에 의존
public class AuthenticationService {
    public void authenticate(HttpServletRequest request) {
        String id = request.getParameter("id");
        String password = request.getParameter("password");
        HttpSession session = request.getSession();
        session.setAttribute("auth", new Authentication(id));
    }
}
```

이렇게 구현하면 응용 서비스를 단독 테스트하기 어렵고, 웹 기술이 바뀌면 응용 서비스도 함께 바뀝니다. 세션과 쿠키 관리는 표현 영역의 책임입니다.

```java
// ✅ 응용 서비스는 필요한 값만 받는다
public class AuthenticationService {
    public AuthenticationResult authenticate(String id, String password) {
        Member member = memberRepository.findById(id);
        if (member == null || !member.matchPassword(password)) {
            throw new AuthenticationException();
        }
        return new AuthenticationResult(member.getId(), member.getName());
    }
}
```

### 트랜잭션 처리

응용 서비스는 도메인 상태 변경을 하나의 트랜잭션으로 묶어야 합니다. 예를 들어 여러 회원을 차단하는 기능에서 일부 회원만 차단되고 일부는 실패하면 데이터 일관성이 깨집니다.

```java
@Transactional
public void blockMembers(List<String> memberIds) {
    List<Member> members = memberRepository.findByIdIn(memberIds);
    for (Member member : members) {
        member.block();
    }
}
```

스프링에서는 보통 응용 서비스 메서드에 `@Transactional`을 붙여 트랜잭션 범위를 지정합니다. 조회 전용 기능이라면 `@Transactional(readOnly = true)`를 사용할 수 있습니다.

---

## 6.4 표현 영역

표현 영역의 책임은 크게 세 가지입니다.

1. 사용자가 시스템을 사용할 수 있는 흐름을 제공합니다.
2. 사용자의 요청을 응용 서비스에 전달하고 결과를 응답으로 변환합니다.
3. 쿠키나 세션처럼 사용자의 연결 상태를 관리합니다.

```java
@PostMapping("/member/changePassword")
public String changePassword(ChangePasswordRequest req, Errors errors) {
    String memberId = SecurityContext.getAuthentication().getId();
    req.setMemberId(memberId);

    try {
        changePasswordService.changePassword(req);
        return "member/changePasswordSuccess";
    } catch (BadPasswordException | NoMemberException ex) {
        errors.reject("idPasswordNotMatch");
        return "member/changePasswordForm";
    }
}
```

표현 영역은 HTTP 요청을 응용 서비스 입력으로 바꾸고, 응용 서비스의 성공/실패 결과를 화면이나 API 응답으로 바꿉니다. 즉, **요청/응답 변환과 흐름 제어**가 표현 영역의 핵심입니다.

표현 영역이 세션을 관리한다는 점도 중요합니다. 로그인 성공 후 세션에 인증 정보를 저장하거나, 로그아웃 시 세션을 제거하는 일은 응용 서비스가 아니라 컨트롤러나 보안 프레임워크가 처리해야 합니다.

---

## 6.5 값 검증

값 검증은 표현 영역과 응용 서비스 양쪽에서 수행할 수 있습니다.

| 검증 위치 | 주로 담당하는 검증 |
|-----------|------------------|
| **표현 영역** | 필수 값, 형식, 길이, 범위 같은 입력 형식 검증 |
| **응용 서비스** | 데이터 존재 여부, 중복 여부, 권한 같은 논리 검증 |
| **도메인 영역** | 도메인 불변식과 핵심 규칙 검증 |

응용 서비스에서 모든 값을 검증할 수도 있지만, 첫 번째 오류에서 예외가 발생하면 사용자가 한 번에 모든 입력 오류를 알기 어렵습니다. 표현 영역에서 기본 형식 검증을 먼저 수행하면 사용자 경험이 좋아집니다.

```java
@PostMapping("/member/join")
public String join(JoinRequest joinRequest, Errors errors) {
    new JoinRequestValidator().validate(joinRequest, errors);
    if (errors.hasErrors()) return "member/joinForm";

    try {
        joinService.join(joinRequest);
        return "member/joinComplete";
    } catch (DuplicateIdException ex) {
        errors.rejectValue("id", "duplicate");
        return "member/joinForm";
    }
}
```

응용 서비스는 표현 영역을 신뢰하면 안 됩니다. 같은 응용 서비스를 웹 컨트롤러, 배치, API, 메시지 소비자가 모두 호출할 수 있기 때문입니다. 따라서 최소한의 논리 검증은 응용 서비스에서 반드시 수행해야 합니다.

```java
@Transactional
public void join(JoinRequest req) {
    if (req == null) throw new IllegalArgumentException("no request");
    if (memberRepository.existsById(req.getId())) {
        throw new DuplicateIdException(req.getId());
    }

    Member member = new Member(req.getId(), req.getName(), req.getPassword());
    memberRepository.save(member);
}
```

검증 오류를 한 번에 모아 전달해야 한다면 `ValidationError` 목록을 만들고 하나의 예외로 던지는 방식도 사용할 수 있습니다.

---

## 6.6 조회 전용 기능과 응용 서비스

조회 기능은 상태를 변경하지 않고 화면에 필요한 데이터를 읽는 것이 목적입니다. 5장에서 다룬 것처럼 조회 전용 DAO나 DTO를 사용하면 도메인 애그리거트를 거치지 않고 필요한 데이터를 효율적으로 가져올 수 있습니다.

```java
public class OrderListService {
    private OrderViewDao orderViewDao;

    public List<OrderView> getOrderList(String ordererId) {
        return orderViewDao.selectByOrderer(ordererId);
    }
}
```

위 서비스처럼 단순히 DAO를 호출해서 결과를 반환하는 일만 한다면 응용 서비스가 별다른 가치를 제공하지 못합니다. 이런 경우 표현 영역에서 조회 전용 DAO를 직접 사용해도 됩니다.

```java
@GetMapping("/myorders")
public String list(ModelMap model) {
    String ordererId = SecurityContext.getAuthentication().getId();
    List<OrderView> orders = orderViewDao.selectByOrderer(ordererId);

    model.addAttribute("orders", orders);
    return "order/list";
}
```

다만 조회 과정에서 권한 확인, 여러 DAO 조합, 외부 시스템 호출, 캐시 정책 같은 흐름 제어가 필요하다면 조회 전용 응용 서비스를 두는 편이 낫습니다.

---

## 한 장 요약

응용 서비스와 표현 영역은 도메인 모델을 외부 요청과 연결하는 계층입니다.

- 표현 영역은 요청을 해석하고 응답을 만드는 역할을 합니다.
- 응용 서비스는 리포지터리와 도메인 객체를 사용해 기능 실행 흐름을 제어합니다.
- 도메인 규칙은 응용 서비스가 아니라 도메인 영역에 둡니다.
- 응용 서비스는 `HttpServletRequest`, `HttpSession` 같은 표현 기술에 의존하면 안 됩니다.
- 상태 변경 기능은 응용 서비스에서 트랜잭션으로 묶어 처리합니다.
- 값 검증은 표현 영역과 응용 서비스가 역할을 나눠 수행할 수 있습니다.
- 단순 조회 전용 기능은 굳이 응용 서비스를 만들지 않고 표현 영역에서 조회 DAO를 직접 사용할 수도 있습니다.

---

## 토론 주제

### 💬 Topic 1. 응용 서비스는 어디까지 얇아야 하는가?

응용 서비스는 흐름 제어만 담당해야 한다고 하지만 실제 서비스에서는 권한, 검증, 외부 API 호출, 알림 발송 같은 코드가 쉽게 섞입니다.

**같이 생각해볼 것:**
- 응용 서비스가 비대해졌다는 신호는 무엇인가?
- 권한 검사는 표현 영역, 응용 서비스, 도메인 중 어디에 두는 것이 좋은가?
- 외부 API 호출은 응용 서비스에서 직접 해도 되는가, 별도 도메인/인프라 서비스로 감싸야 하는가?

---

### 💬 Topic 2. 조회 전용 DAO를 컨트롤러에서 직접 써도 괜찮은가?

책은 단순 조회라면 표현 영역에서 조회 전용 기능을 바로 사용해도 된다고 말합니다. 하지만 모든 요청이 서비스를 거쳐야 한다는 팀 규칙을 가진 곳도 많습니다.

**같이 생각해볼 것:**
- 컨트롤러가 조회 DAO를 직접 의존하면 계층 구조가 흐려지는가?
- 단순 위임만 하는 조회 서비스는 어떤 가치를 제공하는가?
- 조회 로직이 점점 복잡해질 때 어느 시점에 응용 서비스로 분리할 것인가?
