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


