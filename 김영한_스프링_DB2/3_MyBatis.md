### MyBatis 소개

MyBatis는 JdbcTemplate에 비해 더 많은 기능을 제공하는 SQLMapper이다.  
MyBatis의 장점은 쿼리를 xml 안에 작성하기 때문에 여러 줄의 쿼리를 작성하기 쉽다는 점과, 동적 쿼리 작성이 쉽다는 점이다.

```xml
<update id="update">
    update item
    set item_name=#{itemName},
        price=#{price},
        quantity=#{quantity}
    where id = #{id}
</update>
```

