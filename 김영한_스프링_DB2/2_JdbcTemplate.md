### 소개와 설정

sql을 직접 작성해서 사용하길 원하는 경우 JdbcTemplate을 사용하면 된다.

JdbcTemplate의 경우 spring-jdbc에 포함되어있는 기술이고, 별다른 설정도 필요 없어서 간단하게 사용할 수 있다.
build.gradle에 spring-jdbc를 추가하고, DB 사용을 위한 h2 database 라이브러리를 추가하면 설정이 완료된다.

```groovy
//JdbcTemplate 추가
implementation 'org.springframework.boot:spring-boot-starter-jdbc'
//H2 데이터베이스 추가
runtimeOnly 'com.h2database:h2'
```

JdbcTemplate은 템플릿 메서드 패턴을 사용하여 jdbc를 직접 사용할 때 작성했던 반복적인 코드를 효과적으로 줄여준다.  
실행할 sql문 및 파라미터와 함께 응답값을 매핑하는 Mapper를 전달하기만 하면 그 외의 작업은 자동으로 처리된다.  
(커넥션 획득, 커넥션 동기화, 스프링 예외 변환 등)  
다만 JdbcTemplate을 사용할 경우 동적인 sql을 작성하는 데에는 한계가 있다는 점을 고려해야 한다.

### 적용1 - 기본

JdbcTemplate을 사용하는 Repository 구현체를 다음과 같이 구현할 수 있다.

```java
package hello.itemservice.repository.jdbctemplate;

@Slf4j
public class JdbcTemplateItemRepositoryV1 implements ItemRepository {

    private final JdbcTemplate template;

    public JdbcTemplateItemRepositoryV1(DataSource dataSource) {
        this.template = new JdbcTemplate(dataSource);
    }

    @Override
    public Item save(Item item) {
        String sql = "insert into item(item_name, price, quantity) values (?,?,?)";
        KeyHolder keyHolder = new GeneratedKeyHolder();
        template.update(connection -> {
            //자동 증가 키
            PreparedStatement ps = connection.prepareStatement(sql, new String[]{"id"});
            ps.setString(1, item.getItemName());
            ps.setInt(2, item.getPrice());
            ps.setInt(3, item.getQuantity());
            return ps;
        }, keyHolder);

        long key = keyHolder.getKey().longValue();
        item.setId(key);
        return item;
    }
}
```

생성자에서는 DataSource를 주입받아서 이를 이용해 JdbcTemplate 객체를 생성한다.  
해당 template 객체를 이용해서 저장, 수정, 조회 등의 db 작업을 진행한다.

데이터 저장 및 수정 시에는 template.update 메서드를 사용한다.  
이 때 db 설계시 정한 대로 식별자인 id에 자동 증가 키를 삽입하도록 해야 한다.  
이를 위해 데이터 저장 시 template에 id를 바인딩하지 않아야 하는데, 추후에 데이터에 삽입된 id를 알기 위해서는 db에 다시 쿼리를 날려야 한다.

KeyHolder를 이용하면 이 부분이 자동화 된다.  
template.update 메서드의 첫번째 파라미터로 람다를 넘겨서, KeyHolder를 이용해서 prepareStatement를 생성하도록 한다.
데이터를 저장한 후에는 keyHolder.getKey()를 통해 삽입된 자동 증가 키를 얻어 올 수 있다.

다음으로 수정, 단건 조회의 경우 다음과 같이 간단하게 작성할 수 있다.

```java
@Override
public void update(Long itemId, ItemUpdateDto updateParam) {
    String sql = "update item set item_name=?, price=?, quantity=? where id=?";
    template.update(sql,
            updateParam.getItemName(),
            updateParam.getPrice(),
            updateParam.getQuantity(),
            itemId);
}

@Override
public Optional<Item> findById(Long id) {
    String sql = "select id, item_name, price, quantity from item where id = ?";
    try {
        Item item = template.queryForObject(sql, itemRowMapper(), id);
        return Optional.of(item);
    } catch (EmptyResultDataAccessException e) {
        return Optional.empty();
    }
}

private RowMapper<Item> itemRowMapper() {
    return ((rs, rowNum) -> {
        Item item = new Item();
        item.setId(rs.getLong("id"));
        item.setItemName(rs.getString("item_name"));
        item.setPrice(rs.getInt("price"));
        item.setQuantity(rs.getInt("quantity"));
        return item;
    });
}
```

sql을 넘기고, 파라미터 바인딩을 하고, 조회 시에는 조회 결과를 객체로 변환하는 Mapper를 넘기면 된다.  
단건 조회 시에는 queryForObject 메서드를 사용하는데, 조회 결과가 없을 때에는 EmptyResultDataAccessException이 발생하므로 이를 예외 처리해야 한다.

```java
<T> T queryForObject(String sql, RowMapper<T> rowMapper, Object... args) throws
  DataAccessException;
```

> queryForObject 메서드 실행 시 데이터가 초과해서 조회되면 IncorrectResultSizeDataAccessException이 발생한다.  
> 위 예시에서는 where 조건으로 식별자인 id를 사용해서 조회하기 때문에 발생할 일이 없다.

### 적용2 - 동적 쿼리

마지막으로 동적 쿼리를 사용해야 하는 목록 조회이다.  
목록 조회 시에는 template.query 메서드를 사용한다.

```java
<T> List<T> query(String sql, RowMapper<T> rowMapper, Object... args) throws
  DataAccessException;
```

단건 조회시와 동일하게 조회 결과를 객체로 변환하는 Mapper를 넘겨야 한다.  
template 내부에서 알아서 결과 목록에 대해 순회하기 때문에, 단건 조회시 넘긴 mapper를 동일하게 넘기면 된다.

문제가 되는 부분은 동적으로 쿼리를 작성하는 부분이다.  
사용자는 itemName과 maxPrice를 각각 입력하거나 입력하지 않을 수 있고, 이에 따라 쿼리를 동적으로 작성해야 한다.

```sql
-- itemName(x), maxPrice(x)
select id, item_name, price, quantity from item

-- itemName(o), maxPrice(x)
select id, item_name, price, quantity from item
  where item_name like concat('%',?,'%')

-- itemName(x), maxPrice(o)
select id, item_name, price, quantity from item
  where price <= ?

-- itemName(o), maxPrice(o)
select id, item_name, price, quantity from item
  where item_name like concat('%',?,'%')
    and price <= ?
```

위와 같이 동적으로 쿼리를 처리하기 위해서 다음과 같이 복잡한 로직을 작성해야 한다.

```java
@Override
public List<Item> findAll(ItemSearchCond cond) {
    String itemName = cond.getItemName();
    Integer maxPrice = cond.getMaxPrice();

    String sql = "select id, item_name, price, quantity from item";

    if (StringUtils.hasText(itemName) || maxPrice != null) {
        sql += " where";
    }

    boolean andFlag = false;
    List<Object> param = new ArrayList<>();
    if (StringUtils.hasText(itemName)) {
        sql += " item_name like concat('%',?,'%')";
        param.add(itemName);
        andFlag = true;
    }

    if (maxPrice != null) {
        if (andFlag) {
            sql += " and";
        }
        sql += " price <= ?";
        param.add(maxPrice);
    }

    log.info("sql={}", sql);
    return template.query(sql, itemRowMapper(), param.toArray());
}
```

### 적용3 - 구성과 실행

이제 위에서 구현한 JdbcTemplateItemRepositoryV1을 사용하도록 Config를 구성해야 한다.  
DataSource를 주입받고, 이를 이용하여 JdbcTemplateItemRepositoryV1 객체를 생성해서 등록한다.

```java
package hello.itemservice.config;

@Configuration
@RequiredArgsConstructor
public class JdbcTemplateV1Config {

    private final DataSource dataSource;

    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }

    @Bean
    public ItemRepository itemRepository() {
        return new JdbcTemplateItemRepositoryV1(dataSource);
    }
}
```

해당 Config를 사용하도록 SpringBootApplication이 달린 메인 클래스에 어노테이션을 변경한다.

```java
package hello.itemservice;

@Slf4j
//@Import(MemoryConfig.class)
@Import(JdbcTemplateV1Config.class)
@SpringBootApplication(scanBasePackages = "hello.itemservice.web")
public class ItemServiceApplication {
    ...
}
```

또한 이전에 사용했던 메모리 대신 db를 사용하도록 하기 위해 application.properties에 db 설정들을 삽입해야 한다.  
스프링 컨테이너에서는 이를 바탕으로 DataSource를 자동으로 생성해서 주입해준다.

```properties
spring.datasource.url=jdbc:h2:tcp://localhost/~/test
spring.datasource.username=sa
```

추가적으로, 실행되는 sql을 로그로 확인하기 위해 다음의 설정도 추가한다.

```properties
logging.level.org.springframework.jdbc=debug
```

### 이름 지정 파라미터1

지금까지 JdbcTemplate을 사용할 때에는 다음과 같이 순서대로 값을 명시해서 파라미터에 바인딩했다.

```java
String sql = "update item set item_name=?, price=?, quantity=? where id=?";
template.update(sql,
    itemName,
    price,
    quantity,
    itemId);
```

이렇게 파라미터를 바인딩할 경우 코드는 간단하지만, 파라미터의 개수가 늘어나서 순서가 혼동될 경우 매개변수가 뒤바뀌어서 저장되는 심각한 장애가 발생할 수 있다.

이를 방지하기 위해서 NamedParameterJdbcTemplate을 사용할 수 있다.  
NamedParameterJdbcTemplate은 이름 기반 파라미터 바인딩을 지원하는 템플릿으로, 기존의 ? 대신 `:파라미터 이름` 형식으로 매개변수를 sql에 삽입한다.

```java
insert into item (item_name, price, quantity) " +
    "values (:itemName, :price, :quantity)"
```

전체 코드는 아래와 같다.

```java
package hello.itemservice.repository.jdbctemplate;

@Slf4j
public class JdbcTemplateItemRepositoryV2 implements ItemRepository {

    private final NamedParameterJdbcTemplate template;

    public JdbcTemplateItemRepositoryV2(DataSource dataSource) {
        this.template = new NamedParameterJdbcTemplate(dataSource);
    }

    @Override
    public Item save(Item item) {
        String sql = "insert into item(item_name, price, quantity) " +
                "values (:itemName, :price, :quantity)";

        SqlParameterSource param = new BeanPropertySqlParameterSource(item);

        KeyHolder keyHolder = new GeneratedKeyHolder();
        template.update(sql, param, keyHolder);

        long key = keyHolder.getKey().longValue();
        item.setId(key);
        return item;
    }

    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        String sql = "update item " +
                "set item_name=:itemName, price=:price, quantity=:quantity " +
                "where id=:id";

        SqlParameterSource param = new MapSqlParameterSource()
                .addValue("itemName", updateParam.getItemName())
                .addValue("price", updateParam.getPrice())
                .addValue("quantity", updateParam.getQuantity())
                .addValue("id", itemId); //이 부분이 별도로 필요하다.

        template.update(sql, param);
    }

    @Override
    public Optional<Item> findById(Long id) {
        String sql = "select id, item_name, price, quantity from item where id = :id";
        try {
            Map<String, Object> param = Map.of("id", id);
            Item item = template.queryForObject(sql, param, itemRowMapper());
            return Optional.of(item);
        } catch (EmptyResultDataAccessException e) {
            return Optional.empty();
        }
    }

    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();

        SqlParameterSource param = new BeanPropertySqlParameterSource(cond);

        String sql = "select id, item_name, price, quantity from item";
        //동적 쿼리
        if (StringUtils.hasText(itemName) || maxPrice != null) {
            sql += " where";
        }

        boolean andFlag = false;
        if (StringUtils.hasText(itemName)) {
            sql += " item_name like concat('%',:itemName,'%')";
            andFlag = true;
        }

        if (maxPrice != null) {
            if (andFlag) {
                sql += " and";
            }
            sql += " price <= :maxPrice";
        }

        log.info("sql={}", sql);
        return template.query(sql, param, itemRowMapper());
    }

    private RowMapper<Item> itemRowMapper() {
        return BeanPropertyRowMapper.newInstance(Item.class); //camel 변환 지원
    }
}
```
