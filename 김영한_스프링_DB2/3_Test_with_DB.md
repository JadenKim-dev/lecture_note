### 데이터베이스 연동

테스트 구성 시 데이터베이스에 잘 접근해서 데이터 조회/수정이 정상적으로 가능한지 테스트할 필요가 있다.  
이를 위해 테스트 환경에서도 db에 접근할 수 있도록 src/test/resources/application.properties에 db 설정 정보를 추가한다.

```properties
spring.profiles.active=test
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa
logging.level.org.springframework.jdbc=debug
```

테스트 코드의 경우 db1에서 작성한 코드를 재활용했다.  
이 때 단건 생성, 수정의 경우 테스트에 성공하지만, 여러 건에 대한 조회의 경우 테스트에 실패한다.

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
