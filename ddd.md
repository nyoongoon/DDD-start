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
public class ShippingInfo{
    private String receiverName;
    private String receiverPhoneNumber;
    
    private String shippingAddress1;
    private String shippingAddress2;
    private String shippingZipcode;
}
```
- 밸류타입 적용 예시 -> 개념적으로 완전한 하나를 잘 표현할 수 있음
```java
public class Receiver{
    private String name;
    private String phoneNumber;
}

```
```java
public class Address{
    private String address1;
    private String address2;
    private String zipcode;
}
```
```java
public class ShippingInfo{
    private Receiver receiver;
    private Address address;
}
```
- 밸류 객체의 데이터를 변경할 때는 기존 데이터를 변경하기보다는 변경한 데이터를 갖는 새로운 밸류 객체를 생성하는 방식을 선호함.
```java
public class Money{
    private int value;
    public Money add(Money money){
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
public class Order{
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
public class Order{
    //...
    public Order(Orderer orderer, List<OrderLine> orderLines, 
                 ShippingInfo shippingInfo, OrderState state){
        setOrderer(orderer); //생성자 내부에서 검증 로직 가능
        setOrderLines(orderLines); //생성자 내부에서 검증 로직 가능 -> 생성 시점에 검증 가능
        //...
    }
    private void setOrderer(Orderer orderer){
        if(orderer == null) throw new IllegalArgumentException("no orderer");
        this.orderer = orderer;
    }
    private void setOrderLines(List<OrderLine> orderLines){
        verifyAtLeastOneOrMoreOrderLines(orderLines);
        this.orderLines = orderLines;
        calculateTotalAmounts();
    }
    private void verifyAtLeastOneOrMoreOrderLines(List<OrderLine> orderLines){
        if(orderLines == null || orderLines.isEmpty()){
            throw new IllegalArgumentException("no OrderLine");
        }
    }
    private void calculateTotalAmounts(){
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
public class CancelOrderService{
    @Transactional
    public void cancelOrder(String orderId){
        Order order = findOrderById(orderId);
        if(order == null) throw new OrderNotFoundException(orderId);
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
public class CalculateDiscountServiceTest{
    @Test
    public void noCustmer_thenExceptionalShouldBeThrown(){
        CustomerRepository stubRepo = mock(CustomerRepository.class);
        when(stubRepo.findById("noCustId")).thenReturn(null);
        
        RuleDiscounter stubRule = (cust, lines) -> null;
        
        CalculateDiscountService calDisSvc = 
                new CalculateDiscountService(stubRepo, stubRule);
        assertThrows(NoCustomerExceptin.class, ()-> calDisSvc.calculateDiscount(someLines, "noCUstId"));
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
- 1. 단순히 필드를 변경하는 set 메서드를 공개 범위로 만들지 않는다.
- 2. 밸류 타입은 불변으로 구현한다. 

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
public class Order{
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
public interface Speficiation<T>{
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