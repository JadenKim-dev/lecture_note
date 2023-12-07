### 소개1 - 기존 방식의 문제점

쿼리를 직접 작성하면 그 자체로는 문자열이다.  
따라서 컨파일 타임에 타입 체크가 불가능하고, 직접 실행하기 전까지는 정상 작동 여부를 장담할 수 없다.  
컴파일 에러는 빌드 및 실행하기 전부터도 미리 알 수 있지만, 런타임 에러는 직접 실행하고 나서야만 알 수 있다.

만약 SQL이 클래스처럼 자바 코드 처럼 작성 가능해서 타입 체킹이 가능하다면 어떨까?  
이 경우 SQL 자체가 type-safe 해져서 컴파일 시 에러 체킹이 가능해진다.  
이 뿐만 아니라 적절한 IDE를 사용한다면 코드 어시스트의 도움을 받을 수도 있다.

Querydsl은 쿼리를 type-safe하게 작성할 수 있게 도와주는 라이브러리이다.  
자바 코드를 통해서 JPQL을 작성할 수 있게 지원한다.

JPQL은 여러 가지 조건이 엮여 있는 조회 쿼리를 작성하는데 특히 도움이 된다.  
예를 들어 20살 ~ 40살 사이의 김씨 성을 가진 멤버 중, 나이가 많은 순으로 3명을 조회하는 쿼리를 작성한다고 해보자.  
JPQL을 이용해서 해당 쿼리를 작성하면 다음과 같다.

```java
@Test
public void jpql() {
    String query =
    "select m from Member m " +
    "where m.age between 20 and 40 " +
    "and m.name like 'Kim%' " +
    "order by m.age desc";
     
    List<Member> resultList =
        entityManager.createQuery(query, Member.class)
            .setMaxResults(3).getResultList();
}
```

JPQL은 sql과 유사해서 쿼리 작성이 쉽다는 장점이 있지만, 타입 체킹이 불가능하다는 한계가 있다.

JPA에서 공식으로 지원하는 Criteria API도 있다.

```java
@Test public void jpaCriteriaQuery() {
    CriteriaBuilder cb = entityManager.getCriteriaBuilder();
    CriteriaQuery<Member> cq = cb.createQuery(Member.class);
    Root<Member> root = cq.from(Member.class);

    Path<Integer> age = root.get("age");
    Predicate between = cb.between(age, 20, 40);

    Path<String> path = root.get("name");
    Predicate like = cb.like(path, "Kim%");

    CriteriaQuery<Member> query = cq.where( cb.and(between, like) );
    query.orderBy(cb.desc(age));

    List<Member> resultList = 
        entityManager.createQuery(query).setMaxResults(3).getResultList(); 
}
```

다만 사용하기 매우 복잡하고, 엔티티의 각 프로퍼티들은 type safe하게 지원되지 않는다.

### 소개2 - 해결



