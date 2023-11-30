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






