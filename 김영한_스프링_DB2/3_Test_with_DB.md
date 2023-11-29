### 데이터베이스 연동

테스트 구성 시 데이터베이스에 잘 접근해서 데이터 조회/수정이 정상적으로 가능한지 테스트할 필요가 있다.  
이를 위해 테스트 환경에서도 db에 접근할 수 있도록 src/test/resources/application.properties에 db 설정 정보를 추가한다.

```properties
spring.profiles.active=test
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa
logging.level.org.springframework.jdbc=debug
```

테스트 코드의 경우 db1 강의에서 작성한 코드를 재활용했다.  
이 때 단건 생성/수정의 경우 테스트에 성공하지만, 여러 건에 대한 조회의 경우 테스트에 실패한다.

```java
@Test
void findItems() {
    //given
    Item item1 = new Item("itemA-1", 10000, 10);
    Item item2 = new Item("itemA-2", 20000, 20);
    Item item3 = new Item("itemB-1", 30000, 30);

    itemRepository.save(item1);
    itemRepository.save(item2);
    itemRepository.save(item3);

    //여기서 3개 이상이 조회되는 문제가 발생
    test(null, null, item1, item2, item3);
}
```

이는 서비스 환경과 테스트 환경의 db가 구분되지 않아서 발생하는 문제이다.  
서비스 환경에서 기존에 삽입했던 데이터가 테스트 과정에 영향을 미치고 있다.

### 데이터베이스 분리

위에서 발생한 문제를 해결하는 간단한 방법은 테스트용 database를 별도로 만드는 것이다.  
사용하는 db에서 새롭게 데이터베이스를 생성하고, 테스트 환경에 설정한다.  
아래 예시에서는 서비스 환경과 구분된 testcase 데이터베이스를 생성해서 연결했다.

```properties
spring.profiles.active=test
spring.datasource.url=jdbc:h2:tcp://localhost/~/testcase
spring.datasource.username=sa
```

이렇게 하면 위에서 실패했던 목록 조회에 대한 테스트가 정상적으로 완료된다.  
다만 테스트를 두 번 돌리거나, 데이터를 수정하는 다른 테스트케이스와 함께 실행하면 테스트에 실패한다.

테스트는 다른 테스트와 격리되어야 하고, 반복해서 실행할 수 있어야 한다.  
이를 위해서는 테스트를 수행한 후 변경된 데이터를 초기화해야 한다.  
sql의 delete문을 이용해서 데이터를 삭제하는 것도 가능하다.  
하지만 테스트 중간에 예외가 발생하는 등의 이유로 해당 로직이 실행되지 못할 수 있기 때문에, 다른 방법이 권장된다.

### 데이터 롤백

테스트 케이스 실행 후 데이터를 롤백하기 위해 트랜잭션을 활용할 수 있다.  
PlatformTransactionManager를 자동으로 주입받고, BeforeEach를 통해 각 테스트 케이스 전에 트랜잭션을 시작한다.  
또한 각 테스트케이스 실행 후에는 AfterEach를 통해 트랜잭션을 롤백하도록 하면 된다.  
설사 실행 과정에서 문제가 생겨 AfterEach 문이 실행되지 않더라도, 트랜잭션은 커밋되지 않았기 때문에 db에 남지 않는다.

```java
package hello.itemservice.domain;

@Slf4j
@SpringBootTest
class ItemRepositoryTest {

    @Autowired
    ItemRepository itemRepository;

    @Autowired
    PlatformTransactionManager transactionManager;
    TransactionStatus status;

    @BeforeEach
    void beforeEach() {
        //트랜잭션 시작
        status = transactionManager.getTransaction(new DefaultTransactionDefinition());
    }

    @AfterEach
    void afterEach() {
        //트랜잭션 롤백
        transactionManager.rollback(status);
    }

    @Test
    void save() {
        //given
        Item item = new Item("itemA", 10000, 10);

        //when
        Item savedItem = itemRepository.save(item);

        //then
        Item findItem = itemRepository.findById(item.getId()).get();
        assertThat(findItem).isEqualTo(savedItem);
    }
}
```

각 테스트 케이스의 코드는 손 볼 필요가 없다.  
JdbcTemplate을 비롯한 데이터 접근 기술들은 내부에서 트랜잭션 동기화 매니저로부터 커넥션을 얻어온다.  
따라서 레포지토리에 의해 수행되는 db 작업들은 자동으로 트랜잭션의 영향을 받게 된다.

이제 모든 테스트 케이스를 함께 실행하거나 반복해서 실행해도 정상적으로 통과한다.

### @Transactional 적용

스프링에서는 위에서 한 것처럼 직접 트랜잭션을 시작/롤백 하지 않아도 되도록 어노테이션을 지원한다.  
@Transactional을 테스트 클레스에 적용하면 된다.

```java
@Transactional
@SpringBootTest
class ItemRepositoryTest {

    @Autowired
    ItemRepository itemRepository;

    /*
      모두 제거 가능
    @Autowired
    PlatformTransactionManager transactionManager;
    TransactionStatus status;


    @BeforeEach
    void beforeEach() {
        //트랜잭션 시작
        status = transactionManager.getTransaction(new DefaultTransactionDefinition());
    }

    @AfterEach
    void afterEach() {
        //트랜잭션 롤백
        transactionManager.rollback(status);
    }
    */
}
```

테스트 클래스/메서드에 @Transactional이 붙을 경우 먼저 트랜잭션을 시작하고, 테스트 로직을 수행한 뒤, 트랜잭션을 강제 롤백시킨다.  
(일반 서비스 로직에 붙을 경우에는 로직 성공 시 커밋, 예외 발생 시 롤백하는 식으로 작동한다.)  
만약 테스트와 서비스 모두에 @Transactional이 붙어 있을 경우, 서비스 단의 로직도 테스트의 트랜잭션에 참여하게 된다.

하지만 일부 테스트 케이스의 경우 db에 데이터를 커밋해서 확인해보고 싶을 수 있다.  
이 때에는 해당하는 클래스/메서드에 @Commit 어노테이션을 함께 붙이면 된다.  

### 임베디드 모드 DB

H2 데이터베이스는 테스트용 데이터베이스를 별도로 관리하지 않아도 되게끔 메모리 모드를 제공한다.  
다음과 같이 프로필이 test인 경우에 임베디드 db를 사용하는 커넥션을 등록하도록 설정하면 된다. 

```java
package hello.itemservice;

import hello.itemservice.config.*;
import hello.itemservice.repository.ItemRepository;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
import org.springframework.context.annotation.Profile;
import org.springframework.jdbc.datasource.DriverManagerDataSource;

import javax.sql.DataSource;


@Slf4j
@Import(JdbcTemplateV3Config.class)
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")
public class ItemServiceApplication {

	public static void main(String[] args) {
		SpringApplication.run(ItemServiceApplication.class, args);
	}

	@Bean
	@Profile("local")
	public TestDataInit testDataInit(ItemRepository itemRepository) {
		return new TestDataInit(itemRepository);
	}

	@Bean
	@Profile("test")
	public DataSource dataSource() {
		log.info("메모리 데이터베이스 초기화");
		DriverManagerDataSource dataSource = new DriverManagerDataSource();
		dataSource.setDriverClassName("org.h2.Driver");
		dataSource.setUrl("jdbc:h2:mem:db;DB_CLOSE_DELAY=-1");
		dataSource.setUsername("sa");
		dataSource.setPassword("");
		return dataSource;
	}

}
```

datasource의 url로 jdbc:h2:mem:db를 지정하면 어플리케이션이 실행되는 메모리에 db가 함께 뜨게 된다.  
어플리케이션이 종료되면 db 저장 내역도 함께 초기화된다.



