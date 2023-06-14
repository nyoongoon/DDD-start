# 도메인 모델

- 기본적으로 특정 도메인을 개념적으로 표현한 것.
- 객체를 이용한 도메인 모델 -> 기능과 데이터 함꼐 보여줌
- 상태 다이어그램을 이용한 모델링.

# 도메인 모델 패턴

- 표현 / 응용 / 도메인 /인프라스트럭처

## 도메인 계층

- 도메인의 핵심 규칙을 구현
- ex) 주문 도메인 경우 '출고 전에 배송지를 변경할 수 있다'
- 핵심 규칙을 구현한 코드는 도메인 모델에만 위치하기 때문에
- 규칙이 바뀌거나 규칙을 확장해야할 때 다른 코드에 영향을 덜 주고 변경 내역을 모델에 반영할 수 있게 됨.

## 개념 모델과 구현 모델

- 프로젝트 초기에는 개요 수준의 개념 모델로 도메인에 대한 전체 윤곽을 이해하는데 집중,
- 구현하는 과정에서 개념 모델을 구현 모델로 점진적으로 발전시켜 나가야함.

# 엔티티와 벨류

- 도출한 모델은 크게 엔티티와 벨류로 구분할 수 있음.

## 엔티티

- 엔티티의 가장 큰 특징은 식별자를 가진다는 것.
- 식별자는 엔티티 객체마다 고유해서 각 엔티티는 서로 다른 식별자를 갖음.
- 엔티티를 생성하고 속성을 바꾸고 삭제할 때까지 식별자는 유지됨.

### 엔티티의 식별자 생성

- 흔히 사용하는 규칙은 현재 시간과 다른 값을 조합하는 것.
- UUID를 사용하여 식별자 생성.  (universally unique identifier)
- 아이디, 이메일(사전에 중복 방지)
- 일련번호

## 밸류 타입

- 밸류 타입은 개념적으로 완전한 하나를 표현할 때 사용

```java
// 밸류타입 적용안한 예시
public class ShippingInfo {
    private String receiverName;
    private String receiverPhoneNumber;

    private String shippingAddress1;
    private String shippingAddress2;
    private String shippingZipcode;
}
```

- 밸류타입 적용 예시 -> 개념적으로 완전한 하나를 잘 표현할 수 있음

```java
public class Receiver {
    private String name;
    private String phoneNumber;
}

```

```java
public class Address {
    private String address1;
    private String address2;
    private String zipcode;
}
```

```java
public class ShippingInfo {
    private Receiver receiver;
    private Address address;
}
```

- 밸류 객체의 데이터를 변경할 때는 기존 데이터를 변경하기보다는 변경한 데이터를 갖는 새로운 밸류 객체를 생성하는 방식을 선호함.

```java
public class Money {
    private int value;

    public Money add(Money money) {
        return new Money(this.value + money.value);
    }
    // value 변경 메서드 없음.
}
```

- 밸류 객체가 불변 객체가 아니라면 파라미터가 변경될 때 발생하는 문제를 방지하기 위해 데이터를 복사한 후 새로운 객체 생성로직을 따로 추가해야함.
- 두 밸류 객체를 비교할 땐 equals()에서 모든 속성이 같은지를 비교해야함.

### 엔티티 식별자와 밸류 타입

- 식별자 타입으로 String 대신 밸류타입을 사용하면 해당 필드의 의미를 명시적으로 할 수 있음.

```java
public class Order {
    private OrderNo id;
}
```

### 도메인 모델에 set메서드 넣지 않기

#### set메서드는 도메인의 핵심 개념이나 의도를 코드에서 사라지게 한다.

- completePayment()는 결제 완료했다는 의미를 갖는 반면에
- setOrderState()는 단순히 주문 상태값을 설정한다는 의미.

#### 도메인 객체를 생성할 때 온전하지 않은 상태가 될 수 있음.

```
Order order = new Order();
order.setOrderLine(lines);

// 주문자를 설정하지 않은 상태에서 주문 완료 처리 
order.setState(OrderState.PREPARING);
```

- 도메인 객체가 불완전한 상태로 사용되는 것을 막으려면 생성 시점에 필요한 것을 전달해주어야함.(생성자 사용)
- **생성자로 필요한 것을 모두 받으므로 다음처럼 생성자를 호출하는 시점에 검사 로직을 추가할 수 있음**

```java
public class Order {
    //...
    public Order(Orderer orderer, List<OrderLine> orderLines,
                 ShippingInfo shippingInfo, OrderState state) {
        setOrderer(orderer); //생성자 내부에서 검증 로직 가능
        setOrderLines(orderLines); //생성자 내부에서 검증 로직 가능 -> 생성 시점에 검증 가능
        //...
    }

    private void setOrderer(Orderer orderer) {
        if (orderer == null) throw new IllegalArgumentException("no orderer");
        this.orderer = orderer;
    }

    private void setOrderLines(List<OrderLine> orderLines) {
        verifyAtLeastOneOrMoreOrderLines(orderLines);
        this.orderLines = orderLines;
        calculateTotalAmounts();
    }

    private void verifyAtLeastOneOrMoreOrderLines(List<OrderLine> orderLines) {
        if (orderLines == null || orderLines.isEmpty()) {
            throw new IllegalArgumentException("no OrderLine");
        }
    }

    private void calculateTotalAmounts() {
        this.totalAmounts = orderLines.stream().mapToInt(x -> x.getAmounts()).sum();
    }
}
```

- 위 코드의 set 메서드는 앞서 언급한 set메서드와 중요한 차이점이 있는데, **바로 접근 범위가 private이라는 점 !**
- 위 set 메서드는 클래스 내부에서 데이터를 변경할 목적으로만 사용됨.
- -> 정리 : "불변 밸류 타입을 사용하면 밸류 타입에 set 메서드 구현하지 않는다. 특별한 이유 없으면 밸류타입은 불변으로 구현한다."

## 도메인 용어와 유비쿼터스 언어

- 도메인에서 사용하는 용어를 코드에 반여하지 않으면 개발자에게 코드를 해석해야하는 부담을 준다.

# 아키텍처 개요

## 아키텍처

### 표현영역

- HTTP요청을 응용 영역이 필요로 하는 형식으로 변환해서 응용 영역에 전달하고 응용역역의 응답을 HTTP응답으로 변환하여 전송

### 응용영역

- 시스템이 사용자에게 제공해야할 기능을 구현.
- 응용 영역은 기능을 구현하기 위해 **도메인 영역의 도메인 모델을 사용**.
- ex) 주문취소 기능을 제공하는 응용서비스를 예로, 주문 도메인 모델을 사용해서 기능을 구현

```java
public class CancelOrderService {
    @Transactional
    public void cancelOrder(String orderId) {
        Order order = findOrderById(orderId);
        if (order == null) throw new OrderNotFoundException(orderId);
        order.cancel();
    }
}
```

- 응용 서비스는 로직을 직접 수행하기 보다는 도메인 모델에 로직 수행을 위인함다
- 위 코드도 주문 취소 로직을 직접 구현하지 않고 Order객체에 취소처리를 위임하고 있다.

### 도메인 영역

- 도메인 영역은 도메인 모델을 구현한다.
- 도메인 모델은 도메인의 핵심 로직을 구현한다.
- 예를 들어 주문 도메인은 배송지 변경, 결제 완료, 주문 총액 계산과 같은 핵심 로직을 도메인 모델에서 구현한다.

### 인프라스트럭처 영역

- 구현 기술에 대한 것

## 계층 구조 아키텍처

- 계층 구조를 엄격하게 적용한다면 상위 계층은 바로 아래의 계층에만 의존을 가져야하지만
- 구현의 편리함을 위해 계층 구조를 유연하게 적용하기도 함
- ex) 응용 계층은 바로 아래 계층인 도메인 계층에 의존하지만
- 외부 시스템과의 연동을 위해 더 아래 계층인 인프라스트럭처 계층에 의존하기도 함.
- -> 응용계층에서 인프라스트럭처 계층에 의존하면, 테스트의 어려움과 기능확장의 어려움이라는 두가지 문제가 발생할 수 있음
- -> DIP로 해결

## DIP

- 고수준 모듈이 제대로 동작하려면 저수준 모듈을 사용해야한다.
- -> 그런데 고수준 모듈이 저수준 모듈을 사용하면 -> 구현 변경과 테스트가 어려움.
- DIP는 이 문제를 해결하기 위해 저수준 모듈이 고수준 모듈에 의존하도록 바꿈 -> 추상화한 인터페이스
- -> 의존역전원칙 (Dependency Inversion Principle)

### 테스트

- 고수준 모듈이 저수준 모듈에 직접 의존했다면 저수준 모듈이 만들어지기 전까지 테스트를 할 수 없었겠지만
- 인터페이스를 사용하므로 대역객체를 사용해서 테스트를 진행할 수 있다.
- 다음은 대역객체를 사용해서 Customer가 존재하지 않는 경우 익셉션이 발생하는지 검증하는 테스트 코드의 예.

```java
public class CalculateDiscountServiceTest {
    @Test
    public void noCustmer_thenExceptionalShouldBeThrown() {
        CustomerRepository stubRepo = mock(CustomerRepository.class);
        when(stubRepo.findById("noCustId")).thenReturn(null);

        RuleDiscounter stubRule = (cust, lines) -> null;

        CalculateDiscountService calDisSvc =
                new CalculateDiscountService(stubRepo, stubRule);
        assertThrows(NoCustomerExceptin.class, () -> calDisSvc.calculateDiscount(someLines, "noCUstId"));
    }
}
```

- 위 테스트 코드는 실제 구현 클래스가 없어도 스텁이나 모의 객체 같은 테스트 목적의 대역을 사용하여 거의 모든 상황을 테스트할 수 있음.

### DIP 주의사항

- DIP를 적용할 때 하위 기능을 추상화한 인터페이스는 고수준 모듈 관점에서 도출해야함.
-

## 도메인 영역의 주요 구성요소

### 엔티티

- 고유의 식별자는 갖는 개체로 자신의 라이프 사이클을 갖는다.
- 주문, 회원, 상품과 같이 도메인의 고유한 개념을 표현함.
- 도메인 모델의 데이터를 포함하여 해당 데이터와 관련된 기능을 함께 제공

### 밸류

- 고유의 식별자를 갖지 않는 객체로 주로 개념적으로 하나인 값을 표현할 때 사용
- 배송지 주소를 표현하기 위한 주소나 구매 금액을 위한 금액과 같은 타입이 밸류타입
- 엔티티의 속성으로 사용할 뿐만 아니라, 다른 밸류타입의 속성으로도 사용

### 애그리거트

- 연관된 엔티티와 밸류 객체를 개념적으로 하나로 묶은 것.
- ex) 주문과 관련된 Order 엔티티, OrderLine 밸류, Orderer 밸류 객체를 '주문'애그리거트로 묶음

### 리포지터리

- 도메인 모델의 영속성을 처리.

### 도메인 서비스

- 특정 엔티티에 속하지 않은 도메인 로직을 제공.
- '할인금액계산'은 상품, 쿠폰, 회원 등급 등 다양한 조건을 이용해서 구현하게 되는데 이렇게 도메인 로직이 여러 엔티티와 밸류를 필요하다면 도메인 서비스에서 로직 구현

### 엔티티와 밸류

- 도메인 모델의 엔티티와 DB 관계형 모델의 엔티티는 같은 것이 아님
- **가장 큰 차이점은 도메인 모델의 엔티티는 데이터와 함께 도메인 기능을 함께 제공한다는 것 !**
- 또다른 차이점은 도메인 모델의 엔티티는 두 개 이상의 데이터가 개념적으로 하나인 경우 밸류타입을 이용해서 표현할 수 있음.

### 애그리거트

- 도메인 모델이 복잡해지면 개발자가 전체 구조가 아닌 한 개의 엔티티와 밸류에만 집중하는 상황이 발생함
- 도메인 모델에서 전체 구조를 이해하는데 도움이 되는 것은 바로 애그리거트
- 애그리거트는 관련 객체를 하나로 묶은 군집
- ex) 주문 : 주문이라는 도메인 개념은 '주문', '배송지 정보', '주문자', '주문목록' 등의 하위 모델로 구성됨
- 이 하위 개념을 표현한 모델을 하나로 묶어서 '주문'이라는 상위 개념으로 표현할 수 있음.

#### 루트 엔티티

- 애그리거트는 군집에 속한 객체를 관리하는 루트 엔티티를 갖는다.
- 루트 엔티티는 애그리거트에 속해 있는 엔티티와 밸류 객체를 이용하여 애그리거트가 구현해야할 기능을 제공함.
- 애그리거트를 사용하는 코드는 루트가 제공하는 기능을 실행하고, 루트를 통해서 간접적을 ㅗ애그리거느 태 다른 엔티티나 밸류에 접근

### 리포지터리

- 엔티티나 밸류가 요구사항에서 도출되는 도메인 모델이라면 리포지터리는 구현을 위한 도메인 모델임.

# 애그리거트\

## 경계 설정

- 경계를 설정할 때 기본이 되는 것은 도메인 규칙과 요구사항.

## 애그리거트 루트

- 애그리거트 루트는 속한 객체들을 포함한
- 애그리거트의 일관성이 깨지지 않도록, 도메인 기능을 구현
- ex) 주문 애그리거트는 배송지 변경, 상품 변경과 같은 기능을 제공하고, 애그리거트 루트인 Order가 이 기능을 구현한 메서드를 제공해야함.

### 도메인 규칙과 일관성

- 불필요한 중복을 피하고 애그리거트 루트를 통해서만 도메인 로직을 구현하게 만들려면 도메인 모델에 대해 다음의 두가지를 습관적으로 적용해야함
-
    1. 단순히 필드를 변경하는 set 메서드를 공개 범위로 만들지 않는다.
-
    2. 밸류 타입은 불변으로 구현한다.

#### public set()

- 공개 set 메서드는 도메인의 의미나 의도를 표현하지 못하고 도메인 로직을 도메인 객체가 아닌 응용 영역이나 ㅛㅍ현 영역으로 분산시킴.
- 도메인 로직이 한 곳에 응집되지 않음 -> 유지보수 어려움.

### 지연로딩 및 N+1 문제

- Id 참조방식을 사용하면서ㅎ N+1 조회 같은 문제가 발생하지 않도록 조회 전용 쿼리를 사용. (조인 이용)

### 애그리거트 팩토리

- 애그리거트가 갖고 있는 데이터를 이용해서 다른 애그리거트를 생성해야한다면 애그리거트에 팩토이 메서드를 구현하는 것을 고려해보기

# 스프링 DataJpa

## 매핑 구현

### 엔티티와 밸류 기본 매핑 구현

- 애그리거트와 JPA 매핑을 위한 기본 규칙은 다음과 같음
- 애그리거트 루트는 엔티티이므로 @Entity로 매핑 설정.
- 한 테이블에 엔티티와 밸류가 같이 있다면
- -> 밸류는 @Embeddable로 매핑 설정
- -> 밸류 타입 프로퍼티는 @Embedded로 매핑 설정

```java

@Entity
public class Order {
    @Embedded
    private Orderer orderer;
    @Embedded
    private ShippingInfo shippingInfo;
}
```

### 기본 생성자

- Jpa에서 @Entity와 @Embeddeable로 클래스를 매핑하려면 기본 생성자를 제공해야한다.
- DB에서 데이터를 읽어와 매핑된 객체를 생성할 때 기본생성자를 사용해서 객체를 생성.
- 불변타입은 기본 생성자가 필요없음에도 불구하고 기본 생성자 추가해야함.
- -> 기본 생성자는 JPA프로바이더가 객체를 생성할 때만 사용함.
- -> protected로 선언하여 다른코드에서 사용하지 못하도록함.

### 필드 접근 방식 사용

- get/set : 도메인 의도가 사라지고 객체가 아닌 데이터 기반으로 엔티티를 구현할 가능성이 높아짐.
- set대신 의도가 잘 드러나는 기능을 제공해야함.
- -> 객체가 제공할 기능 중심으로 엔티티를 구현하게끔 JPA 매핑 처리를 프로퍼티 방식이 아닌
- -> 필드 방식으로 선택하는게 좋다,.. ???

# 리포지터리와 모델 구현

## 애그리거트의 영속성 전파

- @Embeddable 매핑 타입은 함께 저장되고 삭제 되므로 cascade 속성을 추가로 설정하지 않아도 됨
- @Entity타입에 대한 매핑은 cacade 속성을 사용해서 저장과 삭제시에 함께 처리되도록 설정.

## 식별자 생성 기능

- 식별자 생성 규칙이 있다면 엔티티를 생성할 때 식별자를 엔티티가 별도 서비스로 식별자 생성기느응ㄹ ㅜㅂㄴ리
- 식별자 생셩 규칙은 도메인 규칙이므로 도메인 영역에 식별자 생성 기능을 위치시킬 수 있음.
- 또는 리포지터리에 식별자 생성 메서드를 추가하고 리포지터리 구현클래스에서 알맞게 구현 가능.

## 도메인 구현과 DIP

- JPA를 사용한 이 장의 리포지터리는 DIP 원칙을 어기고 있음
- 엔티티는 구현 기술인 JPA에 특화된 @Entity 등의 애너테이션을 사용
- DIP에 따르면 @Entity등은 구현기술에 속하므로 모데인모델은 구현 기술에 의존하지 말아야하는데 의존하고 있음
- 리포지터리 인터페이스도 도메인 페키지에서 구현기술은 JpaRepository 인터페이스를 상속
- 즉, 도메인이 인프라에 의존하고 있음.
  --> 과한 DIP보다 애그리거트, 리포지터리 등 모데인 모델 사용할 때 구현 기술에 의존하는 것이 좋을 수도 있음.

# 스프링 데이터 JPA를 이용한 조회 기능

## CQRS

- CQRS는 명령(Command) 모델과 조회(Query) 모델을 분리하는 패턴
- 명령 모델은 상태를 변경하는 기능을 구현할 때 사용
- 조회 모델은 데이터를 조회하는 기능을 구현할 때 사용.
- 도메인 모델은 명령 모델을 주로 사용됨.
- 정렬, 페이징, 검색 조건 지정과 같은 기능은 주문 목록, 상품 상세와 같은 조회 기능에 사용됨.
- 리포지터리(도메인 모델)과 DAO(데이터 접근)을 혼용하여 사용

## 검색을 위한 스펙

- 검색 조건을 다양하게 조합해야할 때 사용할 수 있는 것이 스펙
- 스펙은 애그리거트가 특정 조건을 충족하는지 검사할 때 사용하는 인터페이스.
- 스펙 인터페이스는 다음과 같이 정의

```java
public interface Speficiation<T> {
    public booleanisSatisfiedBy(T agg);
}
```

- isSatisfiedBy() 메서드의 agg 파라미터는 검사 대상이 되는 객체
- 스펙을 지포지터리에 사용하면 agg는 애그리거트 루트가 되고,
- 스펙을 DAO에 사용하면 agg는 검색 결과로 리턴할 데이터 객체가 됨.

## 병렬 지정

- 메서드 이름에 OrderBy 사용해서 정렬 기준 지정
- Sort를 인자로 전달.

## 페이징 처리하기

- Sort타입과 마찬가지로 find메서드에 Pageable 타입 파라미터를 사용하면 페이징을 자동 처리해줌.
- Pageable을 사용하는 메서드의 리턴타입이 Pagge일 경우
- -> 스프링 데이터 JPA는 목록 조회 쿼리와 함꼐 COUNT 쿼리도 실행해서 조건에 해당하는 데이터 개수를 구함.
- findBy프로퍼티 형식메서드에서 Pageable을 파라미터로 사용하더라도 리턴타입이 List면 COUNT쿼리 실행 X
- spec을 사용하는 finaAll에 Pageable을 사용하면 리턴 타입이 Page가 아니더라도 COUNT 쿼리 실행됨.

## 하이버네이트 @Subselect 사용

- @Immutable, @Subselect, @Synchronize는 하이버네이트 전용 애너테이션으로, 이 태그를 사용하면 테이블이 아닌 쿼리 결과를 @Entity로 매핑할 수 있음.
- @Subselect는 조회 쿼리를 값으로 갖는다. select 쿼리의 결과를 매핑할 테이블처럼 사용한다.
- @Immutable -> 뷰를 수정할 수 없듯 @Subselect로 조회한 @Entity 역시 수정 불가. -> 실질적인 매핑 테이블이 없으므로 에러발생
- @Synchronize -> 엔티티를 로딩하기 전에 지정한 테이블과 관련된 변경이 발생하면 플러시.



# 응용 서비스와 표현 영역

## 응용 서비스의 역할

- 응용 서비스는 자용자의 요청을 처리하기 위해 리포지터리에서 도메인 객체를 가져와 사용한다.
- 응용 서비스는 주로 도메인 객체 간의 흐름을 제어하기 때문에 다음과 같이 단순한 형태를 갖는다.

```
public Result doSomeFunc(SoemReq req){
    // 1.리포지터리에서 애그리거트를 구한다.
    SomeAgg agg = someAggRepository.findById(req.getId());
    checkNull(agg);
    // 2. 애그리거트의 도메인 기능을 실행.
    add.doFunc(re.getValue());
    // 3. 결과를 리턴한다.
    return createSuccessResult(agg);
}
```

- 응용 서비스가 복잡하다면 응용서비스에서 도메인 로직의 일부를 구현하고 있을 가능성이 높음
- 응용 서비스는 트랜잭션 처리도 담당.
- 응용 서비스의 주요 역할로 접근제어와 이벤트 처리가 있음.

### 도메인 로직 넣지 않기

#### 코드의 응집성이 떨어짐

- 도메인 데이터와 그 데이터를 조작하는 도메인 로직이 한 영역에 위치하지 않고
- 서로 다른 영역에 위치한다는 것은 도메인 로직을 파악하기 위해 여러 영역을 분석해야한다는 것.

#### 중복 구현 가능성

- 여러 응용 서비스에서 동일한 도메인 로직을 구현할 가능성이 높아짐.

## 응용 서비스의 구현

- 응용 서비스는 표현 영역과 도메인 영역을 연결하는 매개체 역할을 하는데 이는 디자인 패턴에서 파사드 같은 역할을 한다.
- 응용 서비스 자체는 복잡한 로직을 수행하지 않기 때문에 응용서비스의 구현은 어렵지 않음

### 응용 서비스의 크기

- 한 응용 서비스 클래스에 회원 도메인의 모든 기능 구현하기
- -> 한 클래스에 코드가 모이면 연관성 적은 코드가 함께 위치하게되서 코드 이해 방해
- 구분되는 기능별로 서비스 클래스를 구현하는 방식은 한 응용 서비스 클래스에서 한 개 내지 2~3개의 기능을 구현한다
- ex) 암오 변경만을 위한 응용 서비스 클래스가 존재

```java
public class ChangePasswordService {
    private MemberRepository memberRepository;

    public void changePassword(String memberId, String curPw, String newPw) {
        Member member = memberRepository.findById(memberId);
        if (member == null) throw new NoMemberException(memberId);
        member.changePassword(curPw, newPw);
    }
}
```
- -> 이 방식을 사용하면 클래스 개수는 많아지지만 한 클래스에 관련 기능을 모두 구현하는 것과 비교해서
- 코드 품질을 일정수준으로 유지하는데 도움이 됨.
- 각 클래스별로 필요한 의존 객체만 포함하므로 다른 기능을 구현한 코드에 영향 받지 않음
- -> 각 기능마다 동일한 로직을 구현할 경우 여러 클래스에 중복해서 동일한 코드를 구현할 가능성이 있음
- 이 경우 다음과 같이 클래스에 로직을 구현해서 코드 중복되는 것을 방지 가능
- 
```java
// 각 응용 서비스에서 공통되는 로직을 별도 클래스로 구현
public final class MemberServiceHelper{
    public static Member findExistingMember(MemberRepository repo, String memberId){
        Member member = memberRepository.findById(memberID);
        if(member == null){
            throw new NoMemberException(memberId);
        }
        return member;
    }
}
```
```java
import static com.myshop.member.application.MemberServiceHelper.*;

public class ChangePasswordService{
    private MemberRepository memberRepository;
    
    public void changePassword(String memberId, String curPw, String newPw){
        Member member = findExisitingMember(memberRepository, memberId);
        member.changePassword(curPw, newPw);
    }
}
```

### 응용 서비스의 인터페이스와 클래스
- 응용 서비스를 구현할 때 논쟁이 될만한 것이 인터페이스가 필요한 지이다. 다음과 같이 인터페이스를 만들고 이를 상속한 클래스를 만드는 것이 필요할까?
```java
public interface ChangePasswordService{
    public void changePassword(String memberId, String curPw, String newPw);
}
public class ChangePasswordServiceImpl implements ChangePasswordService{
    //... 구현
}
```
- 인터페이스가 필요한 상황 중 하나는 구현클래스가 여러개인 것. -> 런타임에 교체하는 것은 드움
- -> **인터페이스가 명확하게 필요하기 전까지 응용 서비스에 대한 인터페이스를 작성하는 것이 좋은 선택이라고 볼 수 없음**
- -> TDD를 하고 표현영역부터 개발한다면 응용서비스의 인터페이스를 이용해서 컨트롤러 구현을 완성해나갈 수 있음
- -> Mockoto와 같은 테스트 도구는 클래스에 대해서도 테스트용 대역 객체를 만들 수 있음. 

### 메서드 파라미터와 값 리턴
- 응용서비스가 제공하는 메서드는 모데인을 이용해서 사용자가 요구한 기능을 실행하는데 필요한 값을 파라미터로 전달받아야한다
- 응용서비스에 데이터로 전달할 요청 **파라미터가 두개 이상 존재하면 별도 DTO 클래스를 사용**하는 것이 편리하다. 
- 값 리턴 : 응용 서비스에서 애그리거트 자체를 리턴하면 편할 수 있지만 **도메인 로직 실행을 응용 서비스와 표현 서비스 두곳에서 할 수 있게 됨**
- -> 이것은 기능 실행 로직을 응용 서비스와 표현 영역에 분산시켜 코드의 응집도를 낮추는 원인이 됨.

### 표현 영역에 의존하지 않기
- 응용 서비스의 파라미터 타입을 결정할 떄 주의할 점 -> 표현 영역과 관련된 타입을 사용하면 안됨!
- ex) 표현영역에 해당하는 HttpServletRequest나 HttpSession을 응용 서비스에 파라미터로 전달하면 안됨!
```java
@Controller
@RequestMapping("/member/changePassword")
public class MemberPasswordController{
  @PostMapping
  public String submit(HttpServletRequest request){
      changePasswordService.changePassword(request);
  } 
}
```
- 응용 서비스에서 표현영역에 대한 의존이 발생하면 응용서비스만 단독으로 테스트하기가 어려워짐
- 표현영역의 구현이 변경되면 응용서비스의 구현도 함께 변경해야하는 문제 발생
- -> 더 심각한 것은 응용 서비스가 표현영역의 역할까지 대신하는 상황이 벌어질 수도 있음
```java
public class AuthenticationService{
    public void authentication(HttpServletRequest request){
        //...
      HttpSession session = request.getSession();
      session.setAttribute("auth", new Authentication(id));
    }
}
```
- HttpSession이나 쿠키는 표현영역의 상태에 해당하는데 이 상태를 응용서비스에서 변경해버리면 
- -> 표현영역의 코드만으로 표현영역의 상태가 어떻게 변경되는지 추적하기 어려워짐. -> 표현영역의 응집도가 깨지는 것
- -> 파라미터와 리턴타입으로 표현영역의 구현기술을 사용하지 않는 것이 중요!

### 트랜잭션 처리
- ** 트랜잭션을 관리하는 것은 응용 서비스**의 중요한 역할 !!


## 표현영역 
- 표현 영역의 책임은 크게 3가지
- 1. 사용자가 시스템을 사용할 수 있는 **흐름(링크/화면)을 제공하고 제어**
- 2. 사용자의 요청을 알맞은 응용 서비스에 **전달하고 결과를 사용자에게 제공** (결과 알맞은 형식으로 변환 및 에러코드에 알맞은 처리)
- 3. 사용자의 **세션을 관리**

- 표현영역의 주된 역할 중 하나는 사용자의 연결 상태인 세션을 관리하는 것
- 웹은 쿠키나 서버 세션을 이용해서 사용자의 연결상태를 관리 -> 권한 검사와 연결되는 내용...

## 값 검증 
- 값 검증은 표현영역과 응용 서비스 두 곳에서 모두 수행할 수 있다. 
- -> 원칙적으로 모든 값에 대한 검증은 응용서비스에서 처리한다. 
- 예를 들어 회원가입을 처리하는 응용서비스는 파라미터로 전달받은 값이 올바른지 검사해야한다. 
- 표현영역은 잘못된 값이 존재하면 이를 사용자에게 알려주고 값을 다시 입력받아야 한다.
- 스프링 MVC는 폼에 입력한 값이 잘못된 경우 에러메시지를 보여주기 위해 Errors나 BindingResult를 사용하는데
- -> 컨트롤러에서 위와 같은 응용서비스를 사용하면 폼에 에러메시지를 보여주기 위해 다음과 같이 다소 번잡한 코드를 작성해야함.
- -> 응용서비스에서 각 값이 유효한지 확인할 목적으로 익셉션을 사용하면 사용자 경험이 좋지 않음(입력 폼 재입력)
- -> 응용서비스에서 값을 검사하는 시점에 첫번째 값이 올바르지 않아 예외를 발생시키면 나머지 항목에 대해 검증하지 못함
- ->-> 이런 불편을 해서 하기 위해 **응용서비스에서 에러코드를 모아 하나의 익셉션을오 발생**시키는 방법이 있음
```java
public class ExampleService {
  @Transactional
  public OrderNo placeOrder(OrderRequest orderRequest){
    List<ValidationError> errors = new ArrayList<>();
    if(orderRequest == null) {
        errors.add(ValidationError.of("empty"));
    }else{
        //...
    }
    // 응용서비스가 입력 오류를 하나의 익셉션으로 모아서 발생
    if(!errors.isEmpty()) throw new ValidationErrorException(errors);
  }    
}
```
- 표현영역은 응용서비스가 ValidationErrorException을 발생시키면  
- ->익셉션에서 에러 목록을 가져와 표현영역에 사용할 형태로 변환처리한다.
```java
public class ExampleController{
    @PostMapping("/orders/order")
    public String order(@ModelAttribute("orderReq") OrderRequest orderRequest, 
                        BindingResult bindingResult,
                        ModelMap modelMap){
        User user = (User) SecurityContextHolder.getContext()
                .getAuthentication().getPrincipal();
        orderRequest.setOrdererMemberId(MemberId.of(user.getUsername()));
        try{
            OrderNo orderNo = placeOrderService.placeOrder(orderRequest);
            modelMap.addAttribute("orderNo", orderNo.getNumber());
        }catch(ValidationErrorException e){
          // 응용 서비스가 발생키신 검증 에러 목록을
          // 뷰에서 사용할 형태로 변환
          e.getErrors().forEach(err->{
              if(err.hasName()){
                  bindingReesult.rejectValue(err.getName(), err.getCode());
              }else{
                  bindingResult.reject(err.getCode());
              }
          });
          populateProductsModel(orderRequest, modelMap);
          return "order/confirm";
        }
    }
}
```

- 표현영역에서 필수 값을 검증하는 방법도 있다.
- 스프링은 값 검증을 위한 Validator 인터페이스를 별도로 제공하므로 검증기를 따로 구현하면 간결하게 작성할 수 있음
```java
public class ExampleController{
    @PostMapping("/member/join")
    public String join(JoinRequest joinRequest, Errors errors){
        new JoinRequestValidater().validate(joinRequest, errors);
        if(errors.hasErrors()) return formView;
        try{
            joinService.join(joinRequest);
            return successView;
        }catch (DuplicateIdException ex){
            errors.rejectValue(ex.getPropertyName(), "duplicate");
            return formView;
        }
    }
}
```
- 표현영역에서 필수값과 값의 형식을 검사하면 실질적으로 응용서비스는 ID 중복 여부와 같은 논리적 오류만 검사할 수도 있음
- -> 응용서비스에서 값 검증을 모두 처리하면 작성코드는 늘어나지만  응용서비스의 완성도가 높아지는 장점도 있음
- 결) 표현, 응용에서 각각 역할별로 검증할 수 있으나, 응용에서 모두 검증하면 완성도가 높아짐.

## 권한 검사
- 보통 다음 세 곳에서 권한 검사를 수행할 수 있음
- 표현 영역
- 응용 서비스
- 도메인

### 표현 영역에서의 권한 검사
- 표현 영역에서 할 수 있는 기본적인 검사는 인증된 사용자인이 아닌지 검사하는 것
- ex) 회원 정보 변경 -> 회원 정보 변경과 관련된 URL은 인증된 사용자만 접근해야함.
- -> 이런 접근 제어를 하기 좋은 위치가 **서블릿 필터** -> 서블릿 필터에서 인증 정보를 생성, 인증 여부를 검사