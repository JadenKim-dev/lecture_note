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

