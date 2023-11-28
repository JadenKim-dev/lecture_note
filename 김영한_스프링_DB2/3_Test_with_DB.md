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







