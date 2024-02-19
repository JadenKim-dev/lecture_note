### Layered Architecture와 테스트

스프링 MVC 등의 프레임워크에서는 주로 Layered Architecture를 사용하여 웹 어플리케이션을 구성한다.  
보통은 Presentation Layer, Business Layer, Persistence Layer로 계층을 나눈다.  
계층을 구분하는 가장 큰 목적은 관심사를 분리하는 것이다.  
역할과 책임을 중심으로 계층을 나눔으로써, 애플리케이션의 유지보수가 보다 편리해진다.

이 때 각 계층에 스프링 MVC와 JPA 등의 기술들이 개입하기 때문에 테스트를 작성하기 어렵다고 느낄 수 있다.  
하지만 테스트의 기본 원리는 기술이 개입된다고 해도 변하지 않는다.  
테스트가 불가능한 부분을 분리해내서 테스트 가능한 부분에 집중하고, 테스트가 문서로 역할할 수 있도록 깔끔하게 작성하는 등의 기본 원칙은 그대로이다.  

다만 테스트 대상이 Spring + JPA로 구성된 웹 어플리케이션일 때 무엇을 어떻게 테스트하면 좋을지를 알 필요가 있다.  
지금까지 우리는 특정 클래스, 메서드를 대상으로 하는 단위 테스트 위주로 테스트 코드를 작성해 왔다.  
하지만 이렇게 각각의 개체들을 테스트 하는 것만으로는, 이들이 함께 협력하여 동작하는 상황에서도 예상대로 동작할 것이라고 장담할 수 없다.

**통합 테스트**는 여러 모듈이 협력하는 기능을 통합적으로 검증하는 테스트이다.  
작은 범위의 단위 테스트를 넘어, 기능 전체를 통합적으로 검증함으로써 기능의 신뢰성을 높일 수 있다.  
각 모듈에 대해서 풍부하게 단위 테스트를 작성하고 큰 기능들에 대해서 통합 테스트를 추가로 작성하면, 테스트로 보다 넓은 범위를 검증할 수 있게 된다.

### 스프링 & JPA 훑어보기

#### 1. 스프링

먼저 라이브러리와 프레임워크의 차이에 대해서 살펴보자.  
라이브러리를 사용하는 상황에서는 내가 작성한 코드가 능동적인 주체가 된다.  
내 코드에서 외부의 라이브러리를 가져와서 사용하는 형식이다.  
이와 달리 프레임워크는 내가 작성한 코드가 수동적인 대상이 된다.  
이미 동작하고 있는 시스템 내에 나의 코드를 삽입하여 동작하게 만드는 식으로 앱을 구현한다.

스프링은 프레임워크로써 제공되기 때문에, 스프링 사용 시 스프링이 갖춰놓은 기술과 환경을 사용하게 된다.  
스프링의 기술들은 IoC(Inversion of Control), DI(Dependency Injection), AOP(Aspect Oriented Programming)를 기반으로 하고 있다.

IoC는 제어의 역전이라는 의미이다.  
전통적인 방식에서는 A 객체와 B 객체가 협력하는 상황이라면, A 객체 내부에서 B 객체를 생성해서 사용해야 한다.  
하지만 이 경우 A 객체와 B 객체가 강결합 되는 문제가 발생한다.  
IoC는 객체를 생성하고 관리하는 제어 권한을 객체 내부가 아닌 외부에서 가져야 한다는 개념이다.  
스프링에서는 IoC를 지키기 위해 외부에서 객체를 생성하여 주입하는 의존성 주입(DI)을 사용한다.  
스프링 컨테이너 내에서 각 객체들은 인터페이스에 의존하고, 실제 객체는 컨테이너에서 삽입한다.   
이를 통해 객체 간의 결합도가 낮아진다.

AOP는 비즈니스 로직과 무관한 코드들을 관점(Aspect)으로 분리한다.  
대표적으로 트랜잭션 세팅 작업, 로깅 세팅 작업 등을 관점이라고 볼 수 있다.  
AOP는 관점들을 각각 모아서 모듈로 분리해냄으로써, 코드에서는 비즈니스 로직에만 집중할 수 있게 한다.

#### 2. JPA

객체 지향적인 개발 방식과 관계형 데이터베이스를 다루는 방식은 그 사이의 간극이 매우 크다.  
이전에는 개발 단계에서 객체의 협력을 중심으로 적절히 구현을 했더라도, 데이터를 저장할 때에는 그것을 하나하나 다시 분리해서 db의 테이블 및 칼럼에 맞춰서 저장해야 했다.  
ORM은 이 사이의 간극을 매꿔서, 객체 지향적인 개발 방식을 그대로 유지하면서도 데이터를 RDB에 저장할 수 있도록 지원하는 기술이다.

JPA는 ORM 기술을 자바 표준으로 정의한 것이며, 그 구현체로 Hibernate를 많이 사용한다.  
JPA를 사용하면 반복적인 CRUD 쿼리 작업의 상당 부분을 자동으로 처리할 수 있다.

#### 엔티티 정의

수업에서는 Order와 Product를 다대다 관계로 구성할 것이다.  
이 때 중간 테이블로 OrderProduct를 사용하여, 일대다 + 다대일 관계로 풀어낼 것이다.

### 프로젝트 구현

이전에 정의했던 요구 사항을 보다 확장해서 정의해보자.  
고객은 주문이 가능한 상품 리스트를 조회할 수 있어야 한다.  
이 때 각 음료들은 판매 중, 판매 보류, 판매 중지 상태를 가지고, 고객들에게는 판매 중이거나 판매 보류 중인 음료 목록을 보여준다.  
고객들이 반환 받는 데이터에는 다음의 정보가 포함되어야 한다.  
`id, 상품 번호, 상품 타입, 판매 상태, 상품 이름, 가격`

먼저 상품 타입과 판매 상태에 대한 enum을 각각 정의한다.  
판매 상태 enum 에는 사용자에게 보여줄 판매 상태의 목록(판매 중, 판매 보류)을 반환하는 forDisplay()를 정의한다.

```java
package sample.cafekiosk.spring.domain.product;

@Getter
@RequiredArgsConstructor
public enum ProductType {
    HANDMADE("제조 음료"),
    BOTTLE("병 음료"),
    BAKERY("베이커리");

    private final String text;
}
```

```java
package sample.cafekiosk.spring.domain.product;

@Getter
@RequiredArgsConstructor
public enum ProductSellingStatus {
    SELLING("판매중"),
    HOLD("판매보류"),
    STOP_SELLING("판매중지");

    private final String text;

    public static List<ProductSellingStatus> forDisplay() {
        return List.of(SELLING, HOLD);
    }
}
```

다음으로 생성일, 수정일 과 같은 공통 정보를 담고 있는 추상 클래스를 정의하고, 각 엔티티에서 상속해서 사용하도록 한다.

```java
package sample.cafekiosk.spring.domain;

@Getter
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class BaseEntity {
    @CreatedDate
    private LocalDateTime createdDateTime;

    @LastModifiedDate
    private LocalDateTime modifiedDateTime;
}
```

> 자동으로 생성일, 수정일 정보를 받아오게 하기 위해 스프링 어플리케이션에 @EnableJpaAuditing을 적용해야 한다.

이제 데이터 CRUD를 수행하기 위해 스프링 데이터 JPA를 이용하여 레포지토리를 정의한다.  
쿼리 메서드 기능을 이용하여 판매 상태에 따라 상품 목록을 조회하는 메서드를 정의한다.

```java
package sample.cafekiosk.spring.domain.product;

@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    /**
     * select *
     * from product
     * where selling_status in ('SELLING', 'HOLD');
     */
    List<Product> findAllBySellingStatusIn(List<ProductSellingStatus> sellingStatuses);
}
```

서비스 단에서는 상품 엔티티 정보에서 필요한 정보만 추출하여 DTO로 반환할 것이다.  
이를 위해 먼저 DTO 객체인 ProductResponse를 정의한다.  
Product 엔티티로부터 편리하게 변환할 수 있도록 생성 메서드 of()를 정의한다.

```java
package sample.cafekiosk.spring.api.service.product.response;

@Getter
public class ProductResponse {
    private Long id;
    private String productNumber;
    private ProductType type;
    private ProductSellingStatus sellingStatus;
    private String name;
    private int price;

    @Builder
    private ProductResponse(Long id, String productNumber, ProductType type, ProductSellingStatus sellingStatus, String name, int price) {
        this.id = id;
        this.productNumber = productNumber;
        this.type = type;
        this.sellingStatus = sellingStatus;
        this.name = name;
        this.price = price;
    }

    public static ProductResponse of(Product product) {
        return ProductResponse.builder()
                .id(product.getId())
                .productNumber(product.getProductNumber())
                .type(product.getType())
                .sellingStatus(product.getSellingStatus())
                .name(product.getName())
                .price(product.getPrice())
                .build();
    }
}
```

이제 다음과 같이 서비스 계층을 정의할 수 있다.  
레포지토리 계층을 통해 데이터를 조회하고, 이를 dto로 변환하여 반환한다.

```java
package sample.cafekiosk.spring.api.service.product;

@RequiredArgsConstructor
@Service
public class ProductService {

    private final ProductRepository productRepository;

    public List<ProductResponse> getSellingProducts() {
        List<Product> products = productRepository.findAllBySellingStatusIn(ProductSellingStatus.forDisplay());

        return products.stream()
                .map(ProductResponse::of)
                .collect(Collectors.toList());
    }
}
```

마지막으로 컨트롤러 객체를 다음과 같이 정의하면 요구사항 구현이 1차적으로 완료된다.

```java
package sample.cafekiosk.spring.api.controller.product;

@RequiredArgsConstructor
@RestController
public class ProductController {
    private final ProductService productService;

    @GetMapping("/api/v1/products/selling")
    public List<ProductResponse> getSellingProducts() {
        return productService.getSellingProducts();
    }
}
```

이제 application.yml에 앱을 실행하기 위한 설정을 진행한다.  
이 때 local, test 프로파일을 분리하여, 각 환경에서의 설정을 별도로 정의한다.  
ddl-auto에는 앱 실행 시 데이터를 초기화 전략을 설정하는데, 각 프로파일 별로 신경 써서 작성해야 한다.  

defer-datasource-initialization에는 data.sql이 실행되는 타이밍을 설정할 수 있다.  
기본값인 false로 지정하면 Hibernate 세팅 전에 data.sql이 실행되고, true로 지정하면 ddl을 통한 테이블 생성이 끝난 후에 실행한다.  
테스트 환경에서는 직접 given에서 필요한 데이터를 삽입할 것이기 때문에 `sql.init.mode: never`로 설정했다.

```yaml
spring:
  profiles:
    default: local

  datasource:
    url: jdbc:h2:mem:~/cafeKioskApplication
    driver-class-name: org.h2.Driver
    username: sa
    password:

  jpa:
    hibernate:
      ddl-auto: none
---
spring:
  config:
    activate:
      on-profile: local

  jpa:
    hibernate:
      ddl-auto: create
    show-sql: true
    properties:
      hibernate:
        format_sql: true
    defer-datasource-initialization: true # (2.5~) Hibernate 초기화 이후 data.sql 실행

  h2:
    console:
      enabled: true
---
spring:
  config:
    activate:
      on-profile: test

  jpa:
    hibernate:
      ddl-auto: create
    show-sql: true
    properties:
      hibernate:
        format_sql: true

  sql:
    init:
      mode: never
```

기본 데이터를 삽입하기 위한 data.sql은 resources 하위에 작성한다.

```sql
insert into product(product_number, type, selling_status, name, price)
values ('001', 'HANDMADE', 'SELLING', '아메리카노', 4000),
       ('002', 'HANDMADE', 'HOLD', '카페라떼', 4500),
       ('003', 'BAKERY', 'STOP_SELLING', '크루아상', 3500);
```

### Persistence Layer 테스트

스프링 데이터 JPA를 이용하여 간단하게 영속성 계층을 구성한 경우에도 레포지토리 단에 대한 테스트가 필요한지 의문일 수 있다.  
하지만 레포지토리 단의 메서드도 우리가 작성한 코드의 일부이기 때문에 검증이 필요하다.  
특히 where 조건이 늘어나는 등 메서드가 복잡해지면 의도한 대로 쿼리가 발생하지 않을 가능성이 커진다.  
또한 레포지토리 단에서 QueryDsl을 사용하도록 변경되는 등 기술에 변화가 생기는 상황에서도 코드의 정상 동작을 보장하기 위해서는 테스트 코드가 반드시 필요하다.

영속성 계층의 테스트는 데이터 저장소를 다루는 일부의 기능만을 확인하기 때문에 단위 테스트의 성격을 가진다.  
하지만 JPA 기술을 사용하기 위해서는 통합 테스트를 수행하는 것처럼 스프링 컨테이너를 띄워서 필요한 의존성 주입을 마쳐야 한다.  
이를 위해 @SpringBootTest를 테스트 클래스에 붙인다.  

> 스프링 데이터 JPA 사용을 위한 빈들을 등록해주는 @DataJpaTest를 사용할 수도 있다.  
> 다만 @DataJpaTest를 사용하면 @Transactional이 함께 적용되고, @SpringBootTest를 사용하는 다른 테스트들과 환경을 통합할 수 없는 문제가 있다.

전체 테스트 코드는 다음과 같다.  
테스트 프로파일을 사용하기 위해 @ActiveProfiles("test")을 테스트 클래스에 붙였다.  
테스트의 given절에서는 세 개의 Product 객체를 생성하고, 이를 레포지토리에 저장한다.  
when절에서 레포지토리의 조회 메서드를 실행하고, then에서 적절히 아이템이 조회 되었는지를 테스트한다.  
이 때 리스트를 검증할 때에는 먼저 리스트의 사이즈를 체크한다.  
이어서 extracting을 통해 각 객체에서 일부의 필드만을 추출한 뒤, containsExactlyInAnyOrder를 통해 tuple을 이용하여 각 데이터의 값을 검증한다.

```java
package sample.cafekiosk.spring.domain.product;

@ActiveProfiles("test")
//@SpringBootTest
@DataJpaTest
class ProductRepositoryTest {

    @Autowired
    private ProductRepository productRepository;

    @DisplayName("원하는 판매상태를 가진 상품들을 조회한다.")
    @Test
    void findAllBySellingStatusIn() {
        // given
        Product product1 = Product.builder()
                .productNumber("001")
                .type(HANDMADE)
                .sellingStatus(SELLING)
                .name("아메리카노")
                .price(4000)
                .build();
        Product product2 = Product.builder()
                .productNumber("002")
                .type(HANDMADE)
                .sellingStatus(HOLD)
                .name("카페라떼")
                .price(4500)
                .build();
        Product product3 = Product.builder()
                .productNumber("003")
                .type(HANDMADE)
                .sellingStatus(STOP_SELLING)
                .name("팥빙수")
                .price(7000)
                .build();
        productRepository.saveAll(List.of(product1, product2, product3));

        // when
        List<Product> products = productRepository.findAllBySellingStatusIn(List.of(SELLING, HOLD));

        // then
        assertThat(products).hasSize(2)
                .extracting("productNumber", "name", "sellingStatus")
                .containsExactlyInAnyOrder(
                        tuple("001", "아메리카노", SELLING),
                        tuple("002", "카페라떼", HOLD)
                );
    }
}
```
