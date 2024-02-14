### Mockito로 Stubbing 하기

이번에는 새로운 요구사항을 구현해보면서, Mock을 언제 어떻게 사용하는지에 대해서 알아보자.  
특정 날짜의 매출 합계를 메일로 전송하는 기능에 대한 니즈가 있다고 해보자.

위 기능을 위해서는 먼저 db로부터 특정 날짜 구간의 결제 완료된 주문을 조회하는 작업이 필요하다.  
레포지토리 단에 jpql을 이용하여 해당 메서드를 구현한다.

```java
package sample.cafekiosk.spring.domain.order;

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    @Query("select o from Order o where o.registeredDateTime >= :startDateTime" +
        " and o.registeredDateTime < :endDateTime" +
        " and o.orderStatus = :orderStatus")
    List<Order> findOrdersBy(LocalDateTime startDateTime, LocalDateTime endDateTime, OrderStatus orderStatus);
}
```

이제 주문 통계를 계산하고 메일을 전송하는 OrderStatisticsService를 구현한다.

> 메일 전송 처럼 외부 API를 사용하는 로직의 경우, 트랜잭션을 걸지 않는 것이 좋다.  
> 트랜잭션을 걸었는데 외부 API 응답이 늦어지면, 늦어진 시간만큼 오래 커넥션을 점유하고 있게 된다.

```java
package sample.cafekiosk.spring.api.service.order;

@RequiredArgsConstructor
@Service
public class OrderStatisticsService {

    private final OrderRepository orderRepository;
    private final MailService mailService;

    public boolean sendOrderStatisticsMail(LocalDate orderDate, String email) {
        // 해당 일자에 결제완료된 주문들을 가져와서
        List<Order> orders = orderRepository.findOrdersBy(
            orderDate.atStartOfDay(),
            orderDate.plusDays(1).atStartOfDay(),
            OrderStatus.PAYMENT_COMPLETED
        );

        // 총 매출 합계를 계산하고
        int totalAmount = orders.stream()
            .mapToInt(Order::getTotalPrice)
            .sum();

        // 메일 전송
        boolean result = mailService.sendMail(
            "no-reply@cafekiosk.com",
            email,
            String.format("[매출통계] %s", orderDate),
            String.format("총 매출 합계는 %s원입니다.", totalAmount)
        );

        if (!result) {
            throw new IllegalArgumentException("매출 통계 메일 전송에 실패했습니다.");
        }

        return true;
    }

}
```

이 때 MailService에서는 MailSendClient에 메일 전송을 요청하고, 관련된 히스토리를 남기도록 구현해보자.  
메일 전송 히스토리는 MailSendHistory 엔티티로 저장하도록 한다.

```java
package sample.cafekiosk.spring.domain.history.mail;

@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Entity
public class MailSendHistory extends BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String fromEmail;
    private String toEmail;
    private String subject;
    private String content;

    @Builder
    private MailSendHistory(String fromEmail, String toEmail, String subject, String content) {
        this.fromEmail = fromEmail;
        this.toEmail = toEmail;
        this.subject = subject;
        this.content = content;
    }

}
``

이에 대한 레포지토리도 함께 정의한다.

```java
package sample.cafekiosk.spring.domain.history.mail;

@Repository
public interface MailSendHistoryRepository extends JpaRepository<MailSendHistory, Long> {
}
```

MailSendClient는 아래와 같이 간단하게만 구현한다.

```java
package sample.cafekiosk.spring.client.mail;

@Slf4j
@Component
public class MailSendClient {

    public boolean sendEmail(String fromEmail, String toEmail, String subject, String content) {
        log.info("메일 전송");
        throw new IllegalArgumentException("메일 전송");
    }

}
```

최종적인 MailService 코드는 아래와 같다.

```java
package sample.cafekiosk.spring.api.service.mail;

@RequiredArgsConstructor
@Service
public class MailService {

    private final MailSendClient mailSendClient;
    private final MailSendHistoryRepository mailSendHistoryRepository;

    public boolean sendMail(String fromEmail, String toEmail, String subject, String content) {
        boolean result = mailSendClient.sendEmail(fromEmail, toEmail, subject, content);
        if (result) {
            mailSendHistoryRepository.save(MailSendHistory.builder()
                .fromEmail(fromEmail)
                .toEmail(toEmail)
                .subject(subject)
                .content(content)
                .build()
            );
            return true;
        }

        return false;
    }
}
```

이제 OrderStatisticsService에 대한 테스트 코드를 작성해야 하는데, 문제가 있다.  
현재는 sendOrderStatisticsMail을 호출할 때마다 메일을 실제로 전송하게 된다.  
테스트 실행 시 매번 메일을 전송하는 것은 시간과 비용에 있어 낭비이다.  
이를 Mocking 해서 처리해보자.  

```java
package sample.cafekiosk.spring.api.service.order;

@SpringBootTest
class OrderStatisticsServiceTest {
    @Autowired
    private OrderStatisticsService orderStatisticsService;

    @Autowired
    private OrderProductRepository orderProductRepository;

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private MailSendHistoryRepository mailSendHistoryRepository;

    @MockBean
    private MailSendClient mailSendClient;

    @AfterEach
    void tearDown() {
        orderProductRepository.deleteAllInBatch();
        orderRepository.deleteAllInBatch();
        productRepository.deleteAllInBatch();
        mailSendHistoryRepository.deleteAllInBatch();
    }

    @DisplayName("결제완료 주문들을 조회하여 매출 통계 메일을 전송한다.")
    @Test
    void sendOrderStatisticsMail() {
        // given
        LocalDateTime now = LocalDateTime.of(2023, 3, 5, 0, 0);

        Product product1 = createProduct(HANDMADE, "001", 1000);
        Product product2 = createProduct(HANDMADE, "002", 2000);
        Product product3 = createProduct(HANDMADE, "003", 3000);
        List<Product> products = List.of(product1, product2, product3);
        productRepository.saveAll(products);

        Order order1 = createPaymentCompletedOrder(LocalDateTime.of(2023, 3, 4, 23, 59, 59), products);
        Order order2 = createPaymentCompletedOrder(now, products);
        Order order3 = createPaymentCompletedOrder(LocalDateTime.of(2023, 3, 5, 23, 59, 59), products);
        Order order4 = createPaymentCompletedOrder(LocalDateTime.of(2023, 3, 6, 0, 0), products);

        // stubbing
        when(mailSendClient.sendEmail(any(String.class), any(String.class), any(String.class), any(String.class)))
            .thenReturn(true);

        // when
        boolean result = orderStatisticsService.sendOrderStatisticsMail(LocalDate.of(2023, 3, 5), "test@test.com");

        // then
        assertThat(result).isTrue();

        List<MailSendHistory> histories = mailSendHistoryRepository.findAll();
        assertThat(histories).hasSize(1)
            .extracting("content")
            .contains("총 매출 합계는 12000원입니다.");
    }

    private Product createProduct(ProductType type, String productNumber, int price) {
        return Product.builder()
            .type(type)
            .productNumber(productNumber)
            .price(price)
            .sellingStatus(SELLING)
            .name("메뉴 이름")
            .build();
    }

    private Order createPaymentCompletedOrder(LocalDateTime now, List<Product> products) {
        Order order = Order.builder()
            .products(products)
            .orderStatus(OrderStatus.PAYMENT_COMPLETED)
            .registeredDateTime(now)
            .build();
        return orderRepository.save(order);
    }
}
```

MailSendClient를 MockBean으로 등록하고, Mockito.when을 이용하여 행위에 대한 stubbing을 적용했다.  
이를 통해 메일 전송이 발생하지 않은 채로 테스트를 실행할 수 있다.

### Test Double

이번에는 가짜 객체와 관련된 테스트 용어들을 알아보자.  
Test Double에는 총 5가지의 종류가 있다.  
먼저 Dummy는 단순히 모방하기만 한, 아무것도 하지 않는 깡통 객체이다.

Fake는 단순한 형태로 동작은 하나, 프로덕션에서 쓰기에는 부족한 객체이다.
예를 들어 메모리에 데이터를 저장하는 Fake Repository를 만들어서 테스트에 사용할 수 있다.

Stub은 미리 설정해둔 일부의 요청에만 미리 준비한 결과를 응답하는 객체이다.  
그 외의 요청을 하면 응답을 받을 수 없다.

Spy는 객체의 호출된 내용을 기록하여, 이후 보여줄 수 있는 객체이다.  
일부는 실제 객체차럼 동작시키고, 일부는 Stubbing 하는 것이 가능하다.

Mock은 행위에 대한 기대를 명세하고, 그에 따라 동작하도록 만들어진 객체이다.

여기서 Stub과 Mock을 잘 구분해야 한다.  
Stub은 상태 검증을 하는 방식으로, 특정 요청을 했을 때 어떤 식으로 내부 상태가 변화하는지를 검증한다.  
이와 달리 Mock은 행위 검증을 하는 방식으로, 특정 메서드가 어떤 식으로 호출되었는지를 검증한다.

### @Mock, @Spy, @InjectMocks

이번에는 순수하게 Mockito를 이용해서 테스트를 작성해보자.
기존에는 @MockBean으로 스프링 컨테이너에 가짜 빈을 등록하는 식으로 Mocking을 했다.  
하지만 스프링 컨테이너를 사용하지 않고 단위 테스트를 구성하는 경우에는 직접 Mockito를 이용해서 테스트를 작성해야 한다.  

MailClient에 메일 전송을 요청하는 MailService에 대한 테스트 코드를 작성해보자.  
@SpringBootTest를 달지 않고, 스프링 컨테이너 없이 테스트를 진행할 것이다.

```java
package sample.cafekiosk.spring.api.service.mail;

@ExtendWith(MockitoExtension.class)
class MailServiceTest {

    @Spy
    private MailSendClient mailSendClient;

    @Mock
    private MailSendHistoryRepository mailSendHistoryRepository;

    @InjectMocks
    private MailService mailService;

    @DisplayName("메일 전송 테스트")
    @Test
    void sendMail() {
        // given
//        when(mailSendClient.sendEmail(anyString(), anyString(), anyString(), anyString()))
//            .thenReturn(true);
        doReturn(true)
            .when(mailSendClient)
            .sendEmail(anyString(), anyString(), anyString(), anyString());

        // when
        boolean result = mailService.sendMail("", "", "", "");

        // then
        assertThat(result).isTrue();
        verify(mailSendHistoryRepository, times(1)).save(any(MailSendHistory.class));
    }

}
```

먼저 @Mock, @Spy를 프로퍼티에 붙이면, 해당 클래스의 가짜 객체가 생성되어 주입된다.  
@InjectMocks를 붙이면 이렇게 생성된 가짜 객체들을 이용하여, 특정 클래스의 객체를 생성해서 주입한다.

@Mock을 붙이면 `when(...).thenReturn()` 형태로 특정 메서드의 동작을 모킹한다.  
이 때 모킹하지 않은 메서드에 대해서는 기본적인 형태로만 구현된 가짜 로직을 호출한다(ex - null 반환).  

@Spy를 붙이면 `doReturn(...).when().메서드()` 형태로 모킹한다.  
이 때 모킹하지 않은 메서드는 실제 객체의 메서드가 동작하게 된다.

### BDDMockito

BDDMockito는 given, when, then을 통해 행위 중심으로 테스트를 작성할 수 있도록 지원하는 라이브러리이다.  
기존에는 given 절에 Mockito.when으로 객체를 모킹하는 식으로 작성했다.

```java
when(mailSendClient.sendEmail(anyString(), anyString(), anyString(), anyString()))
    .thenReturn(true);
```

BDDMockito를 사용하면 다음과 같이 보다 행위 중심적인 코드로 객체를 모킹할 수 있다.

```java
package sample.cafekiosk.spring.api.service.mail;

@ExtendWith(MockitoExtension.class)
class MailServiceTest {

    @Mock
    private MailSendClient mailSendClient;

    @Mock
    private MailSendHistoryRepository mailSendHistoryRepository;

    @InjectMocks
    private MailService mailService;

    @DisplayName("메일 전송 테스트")
    @Test
    void sendMail() {
        // given
//        Mockito.when(mailSendClient.sendEmail(anyString(), anyString(), anyString(), anyString()))
//            .thenReturn(true);
        given(mailSendClient.sendEmail(anyString(), anyString(), anyString(), anyString()))
            .willReturn(true);

        // when
        boolean result = mailService.sendMail("", "", "", "");

        // then
        assertThat(result).isTrue();
        verify(mailSendHistoryRepository, times(1)).save(any(MailSendHistory.class));
    }

}
```

### Classicist vs Mockist

Mockist는 가능한 대부분의 로직을 모킹 처리하자는 입장이다.  
어차피 각각 단위 테스트로 검증을 수행할 것이기 때문에, 불필요하게 실제 객체를 사용해서 테스트 할 필요가 없다는 것이다.  
Mockist의 입장에서는 앞서 수행한 Business Layer 단의 테스트에서도 레포지토리를 모킹하여 테스트 하는 방법을 택했을 것이다.

이와 달리 Classicist는 모킹을 최소화하고, 가능한 실제 객체를 많은 부분에서 사용하자는 입장이다.  
아무리 각 모듈이 단위 테스트로 검증되었다고 하더라도, 이들이 협업했을 때에도 의도한대로 동작할 것임을 보장할 수 없다고 본다.  
정교하게 Stubbing 한다고 하더라도, 가짜 객체가 실제 프로덕션 코드와 동일하게 동작할 것이라고 단언하기 어렵다는 것이다.  

지금까지 우리는 Classicist의 관점처럼 실제 객체를 최대한 사용하여 테스트를 진행했다.  
서비스 단에 대한 테스트 코드 작성 시, 실제 레포지토리 객체를 사용하여 통합 테스트로 진행했다.  
예제의 메일 전송 로직과 같이 외부 시스템과 상호작용 하는 부분만 모킹해서 처리하면 된다.







