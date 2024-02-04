# TDD: Test Driven Development

TDD는 테스트 코드를 먼저 작성하고, 이후에 프로덕션 코드를 작성하는 방법론이다.  
보통 Red -> Green -> Refactoring 단계를 거치게 된다.  
먼저 프로덕션 코드가 없는 상태에서 실패하는 테스트 코드를 작성하고, 이 테스트에 성공하는 프로덕션 코드를 작성한다.  
이 때 프로덕션 코드에는 테스트만 통과할 수 있도록 최소한의 코드로 작성한다.  
이제 테스트에 성공했다면, 테스트 성공을 유지하면서 프로덕션 코드를 개선한다.

전체 음료의 가격을 계산하는 메서드를 TDD를 통해 작성한다고 해보자.  
먼저 다음과 같이 테스트 코드를 작성한다.

```java
@Test
void calculateTotalPrice() {
    CafeKiosk cafeKiosk = new CafeKiosk();
    Americano americano = new Americano();
    Latte latte = new Latte();

    cafeKiosk.add(americano);
    cafeKiosk.add(latte);

    int totalPrice = cafeKiosk.calculateTotalPrice();

    assertThat(totalPrice).isEqualTo(8500);
}
```

이제 해당 테스트를 통과하는 최소한의 프로덕션 코드(calculateTotalPrice)를 개발한다.  
극단적이긴 하지만, 다음과 같이 값을 직접 반환하는 식이어도 괜찮으니 테스트만 통과하면 된다.

```java
public int calculateTotalPrice() {
    return 8500;
}
```

마지막으로 해당 코드를 리팩토링 한다.  
테스트가 통과하도록 유지하면서 본래 의도했던 로직을 구현하면 된다.

```java
public int calculateTotalPrice() {
    int totalPrice = 0;
    for (Beverage beverage : beverages) {
        totalPrice += beverage.getPrice();
    }
    return totalPrice;
}
```

구현이 완성되었다.  
이제 프로덕션 코드에 대한 테스트 코드가 있으므로 보다 과감한 리팩토링이 가능해진다.  
다음과 같이 stream 기반의 코드로 리팩토링하는 과정에서도, 테스트를 통해 안정감 있게 구현이 가능하다.

```java
public int calculateTotalPrice() {
    return beverages.stream()
        .mapToInt(Beverage::getPrice)
        .sum();
}
```

TDD 방식의 가장 핵심적인 가치는 빠른 피드백이다.  
만약 프로덕션 코드를 작성한 후에 테스트 코드를 작성한다면, 일정을 이유로 테스트 코드를 아예 작성하지 않거나, 일부의 케이스만 작성하고 누락할 위험이 크다.  
특히 보통 해피 케이스로만 테스트를 구성하는 경우가 많다.  
이 경우에는 뒤늦게 테스트 케이스를 작성하는 과정에서 잘못된 구현을 확인하게 될 가능성이 많다.

TDD 방식으로 구현을 하게 되면 복잡도가 낮은, 테스트가 가능한 형태로 로직을 구현하게 된다.  
앞서 구현했던 주문을 생성하는 메서드를 떠올려보면, 처음에는 메서드 내부에서 LocalDateTime.now()를 호출하여 현재 시간을 받아오는 식으로 구현했었다.  
이 상태에서는 테스트가 불가능하다고 생각해 테스트 코드를 작성하지 않거나 미뤄둘 가능성이 있다.  
만약 테스트 코드를 먼저 작성하게 되면, 테스트를 염두해 둔 상태로 로직을 구현하게 되기 때문에 테스트가 가능한 형태로 코드가 완성된다.

또한 쉽게 발견하기 어려운 엣지 케이스를 발견할 수 있게 해주고, 구현한 내용에 대해서 빠르게 피드백을 받을 수 있게 한다.  
테스트 코드가 뒷받침 되고 있기 때문에 과감하게 리팩토링을 진행하는 것도 가능해진다.

TDD를 사용하는 것은 관점을 변화시키는데 있어서 주요한 도구이다.  
기존에 프로덕션 코드를 작성한 후 테스트 코드를 작성할 때에는, 구현부를 검증하는 보조 수단으로써 테스트를 사용했다.  
하지만 TDD 방식을 사용했을 때에는 프로덕션 코드와 테스트 코드가 지속적으로 상호작용 하며, 더 견고한 프로덕션 코드를 개발할 수 있게 되었다.

또한 TDD는 클라이언트 관점에서 피드백을 줄 수 있는 도구이다.  
실제로 소프트웨어가 사용되는 단계에 가기 이전에, 해당 객체를 생성하고 사용하는 입장에서 지속적으로 피드백을 받을 수 있다.  
TDD가 언제나 만능은 아니지만, 대부분의 상황에서 큰 이점을 가지기 때문에 익숙해지기 위해 노력하는 것이 좋다.
