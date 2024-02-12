### Presentation Layer 테스트 1

Presentation Layer는 가장 먼저 사용자(클라이언트)의 요청을 받는 계층이다.  
Presentation Layer에서 가장 주요하게 테스트해야 할 것은, 사용자의 요청을 validation 하는 부분이다.  
이를 위해 하위에 존재하는 Business Layer와 Persistence Layer는 Mocking하고, Presentation Layer에 대한 단위 테스트를 작성하는 형식으로 진행할 것이다.

<img src="./images/5_3_Presentation.png" width=500>

이 때 Mock은 가짜, 대역이라는 의미를 가지고 있다.  
테스트 대상에만 집중하고 싶을 때, 다른 부분은 가짜로 정상 동작하는 것처럼 처리하는 방식이다.  
스프링 프레임워크에서는 스프링 MVC 동작을 재현할 수 있도록 지원하는 MockMvc를 제공한다.  
이를 이용하여 Presentation Layer에 대한 테스트를 작성해보자.

이번에도 요구 사항이 추가되었다고 가정하자.  
어드민 페이지가 존재해서, 카페 사장님이나 관리자가 상품을 등록하는 것이 가능해야 한다.  
상품명, 상품 타입, 판매 상태, 가격을 입력받아서 등록하는 기능을 구현한다.

먼저 컨트롤러 단에 다음과 같이 api를 추가한다.

```java
package sample.cafekiosk.spring.api.controller.product;

@RequiredArgsConstructor
@RestController
public class ProductController {

    private final ProductService productService;

    @PostMapping("/api/v1/products/new")
    public void createProduct(ProductCreateRequest request) {
        productService.createProduct(request);
    }

    ...
}
```

컨트롤러 단에서 요청 받는 데이터 타입을 ProductCreateRequest dto로 정의한다.

```java
package sample.cafekiosk.spring.api.controller.product.dto.request;

@Getter
public class ProductCreateRequest {

    private ProductType type;
    private ProductSellingStatus sellingStatus;
    private String name;
    private int price;

    @Builder
    private ProductCreateRequest(ProductType type, ProductSellingStatus sellingStatus, String name, int price) {
        this.type = type;
        this.sellingStatus = sellingStatus;
        this.name = name;
        this.price = price;
    }
}
```

ProductCreateRequest에서는 id, productNumber를 제외한 정보를 받는다.  
productNumber는 가장 최근 생성한 상품의 productNumber에 1을 더한 값을 부여하도록 서비스 단에 구현할 것이다.  
이를 위해서 레포지토리 단에서 가장 최근의 productNumber를 조회하는 메서드를 추가해야 한다.

```java
package sample.cafekiosk.spring.domain.product;

@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    ...
    @Query(value = "select p.product_number from product p order by id desc limit 1", nativeQuery = true)
    String findLatestProductNumber();
}
```

이에 대한 테스트 코드도 다음과 같이 작성한다.  
일반적인 정상 케이스와 함께, 저장된 상품이 하나도 없는 경우에 대한 테스트도 추가한다.

```java
package sample.cafekiosk.spring.domain.product;

@ActiveProfiles("test")
//@SpringBootTest
@DataJpaTest
class ProductRepositoryTest {

    @Autowired
    private ProductRepository productRepository;

    ...

    @DisplayName("가장 마지막으로 저장한 상품의 상품번호를 읽어온다.")
    @Test
    void findLatestProductNumber() {
        // given
        String targetProductNumber = "003";

        Product product1 = createProduct("001", HANDMADE, SELLING, "아메리카노", 4000);
        Product product2 = createProduct("002", HANDMADE, HOLD, "카페라떼", 4500);
        Product product3 = createProduct(targetProductNumber, HANDMADE, STOP_SELLING, "팥빙수", 7000);
        productRepository.saveAll(List.of(product1, product2, product3));

        // when
        String latestProductNumber = productRepository.findLatestProductNumber();

        // then
        assertThat(latestProductNumber).isEqualTo(targetProductNumber);
    }

    @DisplayName("가장 마지막으로 저장한 상품의 상품번호를 읽어올 때, 상품이 하나도 없는 경우에는 null을 반환한다.")
    @Test
    void findLatestProductNumberWhenProductIsEmpty() {
        // when
        String latestProductNumber = productRepository.findLatestProductNumber();

        // then
        assertThat(latestProductNumber).isNull();
    }

    private Product createProduct(String productNumber, ProductType type, ProductSellingStatus sellingStatus, String name, int price) {
        return Product.builder()
                .productNumber(productNumber)
                .type(type)
                .sellingStatus(sellingStatus)
                .name(name)
                .price(price)
                .build();
    }
}
```

이제 서비스 단의 상품 등록 로직을 구현한다.  
TDD 방식으로 진행하기 위해 먼저 테스트 코드를 작성한다.  
레포지토리 단과 동일하게, 기존에 상품이 있는 경우와 없는 경우에 대한 테스트를 함께 작성한다.

```java
package sample.cafekiosk.spring.api.service.product;

@ActiveProfiles("test")
@SpringBootTest
class ProductServiceTest {

    @Autowired
    private ProductService productService;

    @Autowired
    private ProductRepository productRepository;

    @AfterEach
    void tearDown() {
        productRepository.deleteAllInBatch();
    }

    @DisplayName("신규 상품을 등록한다. 상품번호는 가장 최근 상품의 상품번호에서 1 증가한 값이다.")
    @Test
    void createProduct() {
        // given
        Product product = createProduct("001", HANDMADE, SELLING, "아메리카노", 4000);
        productRepository.save(product);

        ProductCreateRequest request = ProductCreateRequest.builder()
                .type(HANDMADE)
                .sellingStatus(SELLING)
                .name("카푸치노")
                .price(5000)
                .build();

        // when
        ProductResponse productResponse = productService.createProduct(request);

        // then
        assertThat(productResponse)
                .extracting("productNumber", "type", "sellingStatus", "name", "price")
                .contains("002", HANDMADE, SELLING, "카푸치노", 5000);

        List<Product> products = productRepository.findAll();
        assertThat(products).hasSize(2)
                .extracting("productNumber", "type", "sellingStatus", "name", "price")
                .containsExactlyInAnyOrder(
                        tuple("001", HANDMADE, SELLING, "아메리카노", 4000),
                        tuple("002", HANDMADE, SELLING, "카푸치노", 5000)
                );
    }

    @DisplayName("상품이 하나도 없는 경우 신규 상품을 등록하면 상품번호는 001이다.")
    @Test
    void createProductWhenProductsIsEmpty() {
        // given
        ProductCreateRequest request = ProductCreateRequest.builder()
                .type(HANDMADE)
                .sellingStatus(SELLING)
                .name("카푸치노")
                .price(5000)
                .build();

        // when
        ProductResponse productResponse = productService.createProduct(request);

        // then
        assertThat(productResponse)
                .extracting("productNumber", "type", "sellingStatus", "name", "price")
                .contains("001", HANDMADE, SELLING, "카푸치노", 5000);

        List<Product> products = productRepository.findAll();
        assertThat(products).hasSize(1)
                .extracting("productNumber", "type", "sellingStatus", "name", "price")
                .contains(
                        tuple("001", HANDMADE, SELLING, "카푸치노", 5000)
                );
    }

    private Product createProduct(String productNumber, ProductType type, ProductSellingStatus sellingStatus, String name, int price) {
        return Product.builder()
                .productNumber(productNumber)
                .type(type)
                .sellingStatus(sellingStatus)
                .name(name)
                .price(price)
                .build();
    }

}
```

이제 해당 테스트를 통과하는 프로덕션 코드를 다음과 같이 작성한다.  
등록할 productNumber를 생성하는 로직은 별도의 메서드로 분리한다.

```java
package sample.cafekiosk.spring.api.service.product;

@Transactional(readOnly = true)
@RequiredArgsConstructor
@Service
public class ProductService {

    private final ProductRepository productRepository;

    @Transactional
    public ProductResponse createProduct(ProductCreateRequest request) {
        String nextProductNumber = createNextProductNumber();

        Product product = request.toEntity(nextProductNumber);
        Product savedProduct = productRepository.save(product);

        return ProductResponse.of(savedProduct);
    }

    private String createNextProductNumber() {
        String latestProductNumber = productRepository.findLatestProductNumber();
        if (latestProductNumber == null) {
            return "001";
        }

        int latestProductNumberInt = Integer.parseInt(latestProductNumber);
        int nextProductNumberInt = latestProductNumberInt + 1;

        return String.format("%03d", nextProductNumberInt);
    }

}
```

### @Transactional readOnly 적용하기

CQRS는 데이터 저장소의 Command(쓰기)와 Query(읽기) 작업을 분리하는 패턴이다.  
쓰기 작업과 읽기 작업을 분리하면, 각 작업 별로 요청을 보낼 database를 분리하는 것이 가능해진다.  
이를 통해 한 쪽에서 장애가 발생했더라도, 다른 종류의 작업에는 영향을 미치지 않도록 할 수 있다.

스프링에서는 @Transactional의 readOnly 옵션을 통해 각 작업을 분리할 수 있다.  
이 때 JPA에서는 readOnly로 트랜잭션이 적용된 경우, 변경 감지를 위한 스냅샷 저장 등을 하지 않기 때문에 성능 면에서도 이점이 있다.  
하나의 서비스 안에서 읽기 작업과 쓰기 작업이 동시에 존재한다면, 클래스 단에서 기본으로 readOnly=true로 적용하고, 메서드에서 필요한 경우에 개별적으로 쓰기가 가능하게 적용하는 편이 좋다.

```java
package sample.cafekiosk.spring.api.service.product;

@Transactional(readOnly = true)
@RequiredArgsConstructor
@Service
public class ProductService {

    private final ProductRepository productRepository;

    @Transactional
    public ProductResponse createProduct(ProductCreateRequest request) {
        ...
    }

    public List<ProductResponse> getSellingProducts() {
        ...
    }
}
```

이와 같이 readOnly를 적용했는지에 따라서 메서드의 동작이 달라지므로, 테스트 코드에서 트랜잭션이 의도한 대로 적용되었는지 테스트하는게 좋다.  
이를 위해서 가능하면 테스트 클래스에 일괄적으로 @Transactional을 적용하지는 않는게 좋다.  
테스트 클래스에 @Transactional을 적용하면, 대상 메서드의 트랜잭션을 테스트하는 것이 불가능하다.
