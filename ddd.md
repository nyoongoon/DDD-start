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
