### MyBatis 소개

MyBatis는 JdbcTemplate에 비해 더 많은 기능을 제공하는 SQLMapper이다.  
MyBatis의 장점은 쿼리를 xml 안에 작성하기 때문에 여러 줄의 쿼리를 작성하기 쉽다.

```xml
<update id="update">
    update item
    set item_name=#{itemName},
        price=#{price},
        quantity=#{quantity}
    where id = #{id}
</update>
```

또한 동적 쿼리를 작성할 때 JdbcTemplate은 string concatenation으로 직접 쿼리를 문자열로 붙여나갔다.  
이와 달리 MyBatis에서는 xml을 이용하여 보다 직관적으로 동적 쿼리를 구성할 수 있다.

```xml
<select id="findAll" resultType="Item">
    select id, item_name, price, quantity
    from item
    <where>
        <if test="itemName != null and itemName != ''">
            and item_name like concat('%',#{itemName},'%')
        </if>
        <if test="maxPrice != null">
            and price &lt;= #{maxPrice}
        </if>
    </where>
</select>
```

다만 MyBatis의 경우 별도로 라이브러리를 설치하고 설정해야 한다.  
JdbcTemplate은 스프링에 내장되어 있기 때문에 좀 더 접근성이 좋은 편이다.

### MyBatis 설정

먼저 build.gradle에 MyBatis 관련 의존성을 명시한다.

```groovy
//MyBatis 스프링 부트 3.0 추가
implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.1'
```

또한 application.properties에 MyBatis 관련 설정을 명시한다.  
src와 test 환경 모두에 작성해야 한다.

```properties
#MyBatis
mybatis.type-aliases-package=hello.itemservice.domain
mybatis.configuration.map-underscore-to-camel-case=true
logging.level.hello.itemservice.repository.mybatis=trace
```

type-aliases-package의 경우 xml에서 참조할 타입의 라이브러리를 지정한다.  
원래는 각 쿼리의 xml 마다 타입에 라이브러리를 함께 명시해야 하지만, 위와 같이 작성하면 라이브러리를 생략할 수 있다.

map-underscore-to-camel-case의 경우 db의 snake_case 칼럼들을 camelCase로 변환할지를 설정한다.  
이를 통해 별도로 alias를 설정하지 않아도 자동으로 칼럼명을 변환하여 사용할 수 있다.

logging은 특정 라이브러리에서 실행되는 mybatis 쿼리에 대한 로그를 보기 위한 설정이다.

### 적용1 - 기본

이제 MyBatis를 이용해서 데이터에 접근해보자.  
MyBatis를 사용하기 위해서는 먼저 각각의 매핑 xml 호출에 사용할 매퍼 인터페이스를 정의해야 한다.

```java
package hello.itemservice.repository.mybatis;

@Mapper
public interface ItemMapper {

    void save(Item item);

    void update(@Param("id") Long id, @Param("updateParam") ItemUpdateDto updateParam);

    Optional<Item> findById(Long id);

    List<Item> findAll(ItemSearchCond itemSearch);
}
```

이제 각 메서드에 매핑되는 xml 파일을 작성하면 된다.  
자바 코드가 아니므로 src/main/resources 하위에 작성하되, 동일한 라이브러리 위치에 작성해야 한다.
위 인터페이스의 위치와 맞게 src/main/resources/hello/itemservice/repository/mybatis/ItemMapper.xml 로 작성한다.

> xml 파일의 위치를 임의로 설정하고 싶다면 application.properties에 설정을 추가하면 된다.
> ex) `mybatis.mapper-locations=classpath:mapper/**/*.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="hello.itemservice.repository.mybatis.ItemMapper">

    <insert id="save" useGeneratedKeys="true" keyProperty="id">
        insert into item (item_name, price, quantity)
        values (#{itemName}, #{price}, #{quantity})
    </insert>

    <update id="update">
        update item
        set item_name=#{updateParam.itemName},
            price=#{updateParam.price},
            quantity=#{updateParam.quantity}
        where id = #{id}
    </update>

    <select id="findById" resultType="Item">
        select id, item_name, price, quantity
        from item
        where id = #{id}
    </select>

    <select id="findAll" resultType="Item">
        select id, item_name, price, quantity
        from item
        <where>
            <if test="itemName != null and itemName != ''">
                and item_name like concat('%', #{itemName}, '%')
            </if>
            <if test="maxPrice != null">
                and price &lt;= #{maxPrice}
            </if>
        </where>
    </select>

</mapper>
```

xml의 mapper 태그의 namespace에는 매칭되는 매퍼 인터페이스를 지정하면 된다.  

각 xml 태그의 id는 인터페이스의 매칭되는 메서드의 이름을 적으면 된다.  
파라미터들은 #{}로 명시한다.  
인터페이스가 하나의 파라미터만 받을 경우에는, 매퍼에서 넘긴 객체의 프로퍼티들이 각각 바인딩 된다.(insert)  
메서드에서 여러 개의 파라미터를 받을 경우에는, 메서드의 각 파라미터들에 @Param을 붙여서 매핑해주어야 한다.(update)

insert 문의 경우 insert 태그를 사용한다.  
db의 자동 생성된 키를 사용할 때에는 useGeneratedKeys에 true를 넣고, keyProperty에 해당하는 프로퍼티 이름을 적으면 된다.  

update, select 문도 마찬가지로 각각 update, select 태그를 사용하면 된다.  
select문의 경우 쿼리의 결과를 resultType에 지정한 객체로 자동으로 변환해준다.  
mybatis.type-aliases-package=hello.itemservice.domain 으로 설정했기 때문에 전체 라이브러리 명을 적지 않아도 된다.  
또한 map-underscore-to-camel-case=true 로 지정되어 있기 때문에 자동으로 camelCase로 변환된다.  
메서드의 반환 타입의 경우 하나의 객체를 반환 받을 때는 단일 객체 또는 Optional로 받고, 여러 개를 받을 때는 List로 받는다.

또한 동적 쿼리를 만들 때에는 where 테그와 if 태그를 통해 편리하게 구성할 수 있다.  
where 태그 내에 if 태그를 작성하게 되는데, test에 작성한 조건이 충족 되면 if 태그 내의 구문이 쿼리에 추가된다.  
이 때 if가 하나도 충족하지 않으면 where 키워드 자체가 제외되고, 하나만 충족하면 and를 where로 바꿔주는 등 편의 기능이 제공된다.

> xml에서 사용하는 '<', '>' 등의 특수 문자는 구문 내에서 사용이 불가능하다.  
> &lt; &gt; 등으로 변환해서 사용하거나, CDATA 내에 작성해야 한다.

### 적용2 - 설정과 실행

이제 MyBatis ItemMapper를 사용하는 레포지토리를 구현하자.  
인터페이스로 정의한 Mapper의 경우 자동으로 구현체를 생성하여 의존성이 주입된다.  

```java
package hello.itemservice.repository.mybatis;

@Slf4j
@Repository
@RequiredArgsConstructor
public class MyBatisItemRepository implements ItemRepository {

    private final ItemMapper itemMapper;

    @Override
    public Item save(Item item) {
        log.info("itemMapper class={}", itemMapper.getClass());
        itemMapper.save(item);
        return item;
    }

    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        itemMapper.update(itemId, updateParam);
    }

    @Override
    public Optional<Item> findById(Long id) {
        return itemMapper.findById(id);
    }

    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        return itemMapper.findAll(cond);
    }
}
```

주입받은 itemMapper에 위임하는 식으로 간단하게 작성할 수 있다.  
이제 해당 레포지토리를 사용하도록 Config를 작성하고 적용시키면 된다.  
정상적으로 실행되고, 테스트도 통과하는 것을 확인할 수 있다.

```java
package hello.itemservice.config;

@Configuration
@RequiredArgsConstructor
public class MyBatisConfig {

    private final ItemMapper itemMapper;

    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }

    @Bean
    public ItemRepository itemRepository() {
        return new MyBatisItemRepository(itemMapper);
    }

}
```

> Datasource, TransactionManager 등의 객체들은 MyBatis 내부에서 자동으로 주입받아서 사용한다.

```java
package hello.itemservice;

@Slf4j
@Import(MyBatisConfig.class)
@Import(V2Config.class)
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
}
```

### 적용3 - 분석

어플리케이션 실행 시점에 MyBatis Spring 연동 모듈에서는 Mapper 인터페이스들을 찾아서 읽어오게 된다.  
이를 바탕으로 동적 프록시 객체를 생성하고, 스프링 컨테이너에 등록하게 된다.  
레포지토리 단에서는 이 때 생성된 프록시 객체를 주입받아서 사용하게 된다.
실제로 레포지토리 단에서 주입받은 객체의 클레스를 출력하면 다음과 같이 확인할 수 있다.

```
itemMapper class=class com.sun.proxy.$Proxy66
```

이를 통해 XML에서 적절한 데이터를 찾아서 쿼리로 호출하는 부분이 자동화된다.  
커넥션, 트랜잭션을 사용하는 부분이나 DataAccessException로의 스프링 예외 변환도 모듈 내부에서 자동으로 수행해준다.

### MyBatis 동적 쿼리

MyBatis는 동적 쿼리를 편리하게 사용하기 위한 다양한 태그들을 제공한다.  
if, choose (when, otherwise), trim (where, set), foreach 등의 태그가 존재한다.  

```xml
<select id="findActiveBlogLike" resultType="Blog">
    SELECT * FROM BLOG WHERE state = ‘ACTIVE’
    <choose>
        <when test="title != null">
            AND title like #{title}
        </when>
        <when test="author != null and author.name != null">
            AND author_name like #{author.name}
        </when>
        <otherwise>
            AND featured = 1
        </otherwise>
    </choose>
</select>
```

```xml
<select id="selectPostIn" resultType="domain.blog.Post">
    SELECT *
    FROM POST P
    <where>
        <foreach item="item" index="index" collection="list"
            open="ID in (" separator="," close=")" nullable="true">
                #{item}
        </foreach>
    </where>
</select>
```

### MyBatis 기타 기능

MyBatis에서는 어노테이션을 통한 쿼리 작성을 지원한다.  
다만 동적 쿼리는 만들 수 없기 때문에 쿼리가 간단한 경우에만 사용해야 한다. 

```java
@Select("select id, item_name, price, quantity from item where id=#{id}")
Optional<Item> findById(Long id);
```

또한 파라미터를 체워 넣는 PreparedStatement 방식으로 동작하는 #{} 외에도, 문자열 자체를 변수로 대체하는 ${}도 사용할 수 있다.  
다만 문자열 대체의 경우 SQL Injection에 취약하기 때문에 가급적 사용해선 안 된다.

```java
@Select("select * from user where ${column} = #{value}")
User findByColumn(
    @Param("column") String column, 
    @Param("value") String value
);
```

또한 sql - include 태그를 사용하면 중복되는 코드 조각을 편리하기 재활용할 수 있다.  
대표적으로 select 문의 조회 칼럼을 나열하는 부분을 재활용할 수 있다.  
또한 property 태그를 사용하면 매칭되는 프로퍼티의 값을 전달하는 것도 가능하다.

```xml
<sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql>

<select id="selectUsers" resultType="map">
    select
        <include refid="userColumns"><property name="alias" value="t1"/></include>,
        <include refid="userColumns"><property name="alias" value="t2"/></include>
    from some_table t1
        cross join some_table t2
</select>
``` 

```xml
<sql id="sometable">
    ${prefix}Table
</sql>

<sql id="someinclude">
    from
    <include refid="${include_target}"/>
</sql>

<select id="select" resultType="map">
    select
    field1, field2, field3
    <include refid="someinclude">
        <property name="prefix" value="Some"/>
        <property name="include_target" value="sometable"/>
    </include>
</select>
```

위 예시의 결과는 select field1, field2, field3 from SomeTable 이다.

또한 select 시 각 컬럼에 대해서 alias를 사용하는 대신, resultMap 태그에 매핑 정보를 따로 정리할 수 있다.  
이 때에는 select문의 resuptSet 속성으로 해당 태그를 연결하면 된다.

```xml
<resultMap id="userResultMap" type="User">
    <id property="id" column="user_id" />
    <result property="username" column="user_name"/>
    <result property="password" column="hashed_password"/>
</resultMap>

<select id="selectUsers" resultMap="userResultMap">
    select user_id, user_name, hashed_password
    from some_table
    where id = #{id}
</select>
```

또한 association, collection 태그를 사용하면 db join을 통한 연관관계 매핑도 가능하다.  
다만 성능과 복잡성 때문에 실효성이 없는 편이다.
