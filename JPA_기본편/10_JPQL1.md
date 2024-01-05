### 0) 개요

다양한 검색조건을 사용해서 데이터를 가져오기 위해서는 복잡한 쿼리를 짤 수 있어야 한다.  
JPA는 다양한 기술들을 통해서 쿼리 방법을 지원한다.

- JPQL(Java Persistence Query Language)
  - 직접 작성하거나, JPQL 빌더(JPA Criteria, QueryDSL)를 사용
- 네이티브 SQL
  - 특정 데이터베이스 벤더에 종속되는 쿼리를 작성해야 하는 경우 사용
- JDBC API, JdbcTemplate, MyBatis 등을 함께 사용하는 것도 가능

### 1) JPQL 소개

JPA를 사용하면 EntityManager.find()를 사용하거나, a.getB().getC() 처럼 객체 그래프를 탐색하여 데이터를 가져올 수 있다.  
이를 통해 엔티티 객체를 중심으로 개발하는 것이 가능하다.

다만 위 방법들만으로는 `나이가 18세 이상인 회원` 처럼 특정 조건이 적용된 검색 쿼리를 구성하는 것은 불가능하다.  
필요한 데이터만 DB에서 불러오려면 WHERE나 GROUPBY 등의 구문이 적용된 SQL이 필요하다.  
JPA는 검색을 할 때에도 테이블이 아닌 엔티티 객체를 대상으로 검색해야 하는데, 그렇다고 해서 모든 DB 데이터를 불러와서 객체로 변환해서 검색할 수는 없다.

이러한 문제를 해결하기 위해, JPA는 SQL을 추상화한 JPQL이라는 객체 지향 쿼리 언어를 제공한다.  
SQL 문법과 유사하게 SELECT, FROM, WHERE, GROUP BY, HAVING, JOIN 등의 기능을 지원한다.  
이 때 SQL은 데이터베이스 테이블을 대상으로 쿼리를 하는 반면에, JPQL은 엔티티 객체를 대상으로 쿼리를 한다.  
SQL을 추상화했기 때문에 특정 데이터베이스 쿼리 문법에 의존하지 않는다.

```java
/**
 * JPQL을 이용한 검색
 */
String jpql = "select m From Member m where m.username like '%kim%'";
List<Member> result = em.createQuery(jpql,Member.class).getResultList();
for (Member member : result) {
    System.out.println("member = " + member);
}
```

위 예제의 JPQL에서 Member는 MEMBER 테이블이 아닌, 엔티티 Member를 가리킨다.  
alias로 m을 사용하여 Member 엔티티 객체를 가리키도록 구성했다.  
이 때 JPQL은 다음과 같이 SQL로 변환되어 DB에 전달된다.

```bash
Hibernate:
    /* select m From Member m where m.username like '%kim%' */
        select
            member0_.MEMBER_ID as MEMBER_I1_6_,
            …
        from Member member0_ where member0_.USERNAME like '%kim%'
```

#### Criteria 소개

Criteria는 JPA에서 공식으로 지원하는 JPQL 빌더이다.  
Criteria를 통해 문자가 아닌 자바 코드로 JPQL을 작성할 수 있다.

다만 너무 복잡하고 유지보수가 어려워서 실용성이 떨어진다.  
보통은 Criteria 대신에 QueryDSL을 사용하는 것을 권장한다.

```java
// Criteria 사용 준비
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Member> query = cb.createQuery(Member.class);

// 루트 클래스 (조회를 시작할 클래스)
Root<Member> m = query.from(Member.class);

// 쿼리 생성
CriteriaQuery<Member> cq = query.select(m).where(cb.equal(m.get("username"), "kim"));
List<Member> resultList = em.createQuery(cq).getResultList();
```

#### QueryDSL 소개

QueryDSL은 Criteria 같은 JPQL 빌더로, 문자가 아닌 자바 코드로 JPQL을 작성할 수 있도록 지원한다.  
이를 통해 컴파일 시점에 문법 오류를 찾을 수 있다.  
단순해서 의미 파악이 용이하며, 동적쿼리 작성을 편리하게 지원하기 때문에 실무에서도 권장되는 기술이다.

```java
// JPQL: select m from Member m where m.age > 18
JPAFactoryQuery queryFactory = new JPAQueryFactory(em);
QMember m = QMember.member;
List<Member> list = queryFactory
        .selectFrom(m)
        .where(m.age.gt(18))
        .orderBy(m.name.desc())
        .fetch();
```

#### 네이티브 SQL 사용

JPA를 통해 SQL을 직접 사용하는 것도 가능하다.  
보통 JPQL로 작성이 불가능한 특정 데이터베이스에 의존적인 기능이 필요할 때 사용한다.  
(Ex. 오라클 CONNECT BY, 특정 DB만 사용하는 SQL 힌트)

```java
String sql = "SELECT ID, AGE, TEAM_ID, NAME FROM MEMBER WHERE NAME = 'kim'";  // SQL 쿼리를 직접 작성
List<Member> resultList = em.createNativeQuery(sql, Member.class).getResultList();
```

#### JDBC 직접 사용, JdbcTemplate

JPA를 사용하는 동시에, JDBC 커넥션을 직접 다루거나 스프링 JdbcTemplate, MyBatis 등을 함께 사용하는 것도 가능하다.

단 외부 기술을 사용하기 전에 영속성 컨텍스트를 강제로 플러시해야 할 수 있다.  
JPA는 플러시하기 전이라도 1차 캐시를 통해 영속성 컨텍스트에 존재하는 데이터를 조회할 수 있지만, JPA에서 관리하지 않는 기술들은 접근이 불가능하다.  
따라서 외부 기술을 통해 SQL을 실행하기 전에, 영속성 컨텍스트를 수동 플러시해야 한다.

```java
Member member1 = new Member();
member1.setUsername("member1");
em.persist(member1);

// JPA에서 지원하지 않는 기술 사용
// db에는 member1이 저장되지 않은 상태에서 조회가 된다.
List<Member> memberList = dbconn.executeQuery("select * from member");
```

### 2) JPQL - 기본 문법과 쿼리 API

JPQL은 SQL을 추상화한 것으로, 최종적으로는 SQL로 변환되어 실행된다.  
JPQL의 예시는 다음과 같다.

`select m from Member as m where m.age > 18`

JPQL 작성 시 엔티티와 속성은 대소문자를 구분하고(Member, age), JPQL 키워드는 대소문자를 구분하지 않는다(SELECT, FROM, where).  
또한 테이블 이름이 아닌 엔티티 이름(Member)을 사용하고, 엔티티에 대한 별칭은 필수이다(m).  
별칭 지정 시 as는 생략 가능하다.

SELECT, UPDATE, DELETE 문의 형식은 다음과 같다.

```bash
# SELECT 문
select_절
    from_절
    [where_절]
    [groupby_절]
    [having_절]
    [orderby_절]

# UPDATE 문
update_절
    [where_절]

# DELETE 문
delete_절
    [where_절]
```

select 절에서는 COUNT, SUM 처럼 통계값을 조회하는 것도 가능하다.  
이 때 집합 기능을 사용하기 위해 GROUP BY, HAVING 절을 사용하거나, 정렬을 위해 ORDER BY 절을 사용할 수도 있다.

```bash
select
    COUNT(m),      # 회원 수
    SUM(m.age),    # 나이 합
    AVG(m.age),    # 평균 나이
    MAX(m.age),    # 최대 나이
    MIN(m.age)     # 최소 나이
from Member m
```

JPQL의 실행 결과로는 TypedQuery, Query를 받을 수 있다.  
TypeQuery는 단일한 값을 조회할 때 사용하고, Query는 여러 값을 함께 조회할 때 사용한다.

```java
TypedQuery<Member> query = em.createQuery("select m from Member m", Member.class);

// String 타입의 컬럼 조회 → 반환 타입으로 TypedQuery<String> 사용
TypedQuery<String> query2 = em.createQuery("select m.username from Member m", String.class);

// String, int 2가지 타입의 컬럼을 함께 조회 → Query 사용
Query query3 = em.createQuery("select m.username, m.age from Member m");
```

쿼리를 통한 결과 조회시 데이터가 하나 이상이면 `query.getResultList()`를 통해 리스트로 반환받으면 된다.  
결과가 정확히 하나인 경우에는 `query.getSingleResult()`을 통해 단일 객체로 반환받으면 된다.  
단 조회 결과가 없을 경우 `query.getResultList()`를 사용하면 빈 리스트를 받지만, `query.getSingleResult()`를 사용하면 `javax.persistence.NoResultException` 예외가 발생한다.  
또한 `query.getSingleResult()`의 결과가 둘 이상이면 `javax.persistence.NonUniqueResultException` 예외가 발생한다.

```java
TypedQuery<Member> query = em.createQuery("select m from Member m", Member.class);
List<Member> resultList = query.getResultList();
Member result = query.getSingleResult();
```

JPQL은 파라미터 바인딩을 지원한다.  
파라미터 바인딩은 이름 또는 위치 기준으로 가능한데, 위치 기준으로 바인딩 할 경우 중간에 데이터가 추가되면 적절히 바인딩을 수정하지 않으면 섞일 위험이 있다.  
따라서 가능하면 이름 기준으로 바인딩할 것을 권장한다.

```java
// 이름 기준
Member result = em.createQuery("select m from Member m where m.username = :username", Member.class)
        .setParameter("username", "member1")
        .getSingleResult();

// 위치 기준
Member result = em.createQuery("select m from Member m where m.username ?= 1", Member.class)
        .setParameter(1, "member1")
        .getSingleResult();
```

### 3) 프로젝션(Projection)

프로젝션은 SELECT 절에 조회할 대상을 지정하는 것이다.  
엔티티, 임베디드 타입, 스칼라 타입(숫자, 문자등 기본 데이터 타입)이 프로젝션 대상이 될 수 있다.

```bash
SELECT m FROM Member m   # 엔티티 프로젝션
SELECT m.team FROM Member m   # 엔티티 프로젝션
SELECT m.address FROM Member m   # 임베디드 타입 프로젝션
SELECT m.username, m.age FROM Member m   # 스칼라 타입 프로젝션
```

먼저 기본적인 엔티티 프로젝션의 예시는 다음과 같다.  
엔티티 프로젝션을 통해 추출된 엔티티들은 모두 영속성 컨텍스트에 의해 관리되기 때문에, 변경 감지 등의 영속성 컨텍스트 기능들을 사용할 수 있다.

```java
※ src/main/java/hellojpa/jpaMain
Member member = new Member();
member.setUsername("member1");
em.persist(member);

em.flush();
em.clear();

List<Member> result = em.createQuery("select m from Member m", Member.class)
        .getResultList();
Member findMember = result.get(0);
findMember.setAge(20);
```

```bash
Hibernate:    # 변경 감지에 의해 정상적으로 update가 이루어짐
    /* update
        hellojpa.Member */ update
            Member
        …
```

연관 엔티티로 프로젝션을 할 때에는 createQuery() 에서 파라미터로 연관 엔티티의 클래스 타입을 넘겨준다.  
이 경우 연관 관계에 있는 엔티티를 JOIN을 통해 가져온다.

```java
List<Team> result = em.createQuery("select m.team from Member m", Team.class)
        .getResultList();
```

```bash
Hibernate:
    /* select m.team from Member m */
        select
            team1_.TEAM_ID as TEAM_ID1_11_,
            …
        from
            Member member0_
        inner join
            Team team1_
                on member0_.TEAM_ID=team1_.TEAM_ID
```

다만 위와 같이 createQuery에 인자로 넘기는 식으로 처리하기 보다는, sql 문법에 맞춰서 JPQL을 쓰는게 더 적절하다.  
`select t from Member m join m.team t`  
가능한 SQL과 형식이 유사해야 변환된 쿼리를 예측하기 좋고 관리하기도 쉽다.

동일한 인터페이스를 통해 임베디드 타입으로 프로젝션해서 가져오는 것도 가능하다.

```java
em.createQuery("select o.address from Order o", Address.class)
        .getResultList();
```

```bash
Hibernate:
    /* select o.address from Order o */
    select
        order0_.city as col_0_0_,
        order0_.street as col_0_1_,
        order0_.zipcode as col_0_2_
    from
        ORDERS order0_
```

마지막으로 스칼라 타입으로 가져오는 경우도 있다.  
이 때에는 distinct를 함께 사용해서 중복을 제거하는 것도 가능하다.

```java
em.createQuery("select distinct m.age, m.username from Member m")
        .getResultList();
```

```bash
Hibernate:
    /* select distinct m.age, m.username from Member m */
        select
            distinct member0_.age as col_0_0_,
            member0_.USERNAME as col_1_0_
        from
            Member member0_
```

다음과 같이 임의의 데이터를 조합하여 함께 조회해야 하는 경우도 있다.  
`SELECT m.username, m.age FROM Member m`

이 경우에는 Query 타입으로 조회할 수 있다.  
`query.getResultList()`를 통해 조회한 결과를 List 타입으로 받을 수 있다.  
List 안의 각 Object를 Object[]로 캐스팅하면 각 인덱스에 조회한 데이터가 저장되어 있다.

```java
List resultList = em.createQuery("select distinct m.age, m.username from Member m")
    .getResultList();  // age, memeber의 쌍으로 각 인덱스에 저장되어 있음
Object o = resultList.get(0);
Object[] result = (Object[]) o;

System.out.println("age = " + result[0]);
System.out.println("username = " + result[1]);
```

```bash
age = 20
username = member1
```

타입 캐스팅 하는 과정을 생략하기 위해 List의 제네릭으로 Object[]를 지정해서 조회하는 것도 가능하다.

```java
List<Object[]> resultList = em.createQuery("select distinct m.age, m.username from Member m")
    .getResultList();
Object[] result = resultList.get(0);
```

또는 new 명령어를 이용하여 DTO로 바로 조회하는 것도 가능하다.  
이 때 DTO 부분에는 패키지 명을 포함한 전체 DTO 클래스명을 입력해야 하고, DTO 클래스에는 순서와 타입이 일치하는 생성자가 정의되어 있어야 한다.

```java
// src/main/java/hellojpa/MemberDTO
public class MemberDTO {
    private String username;
    private int age;

    public MemberDTO(String username, int age) {
        this.username = username;
        this.age = age;
    }
    ... // Getter, Setter
}
```

```java
// src/main/java/hellojpa/JPAMain
List<MemberDTO> result = em.createQuery(
    "select new hellojpa.MemberDTO(m.username, m.age) from Member m", MemberDTO.class
).getResultList();
MemberDTO memberDTO = result.get(0);
```

가장 깔끔한 방법이지만, 전체 DTO 클래스의 경로명을 적어야 하는 불편함이 있긴 하다.  
QueryDSL을 사용하면 이런 불편함을 해결할 수 있다.

### 4) 페이징

페이징을 사용할 때에는 createQuery() 후에 setFirstResult(), setMaxResults()를 통해 페이징을 위한 설정 값을 지정하면 된다.

- setFirstResult(int startPosition) : 조회 시작 위치 설정 (0부터 시작)
- setMaxResults(int maxResult) : 조회할 데이터 수 설정

```java
※ src/main/java/hellojpa/JPAMain
/**
 * 데이터 세팅
 */
for(int i=0; i<101; i++) {
    Member member = new Member();
    member.setUsername("member" + i);
    member.setAge(i);
    em.persist(member);
}

/**
 * 페이징 쿼리 실행
 */
String jpql = "select m from Member m order by m.name desc";
List<Member> resultList = em.createQuery(jpql, Member.class)
    .setFirstResult(1)
    .setMaxResults(5)
    .getResultList();
System.out.println("result = " + result.size());
for (Member member1 : result) {
    System.out.println("member = " + member1);
}
```

```bash
result = 5
member = Member{id=99, username='member99', age=99}
member = Member{id=99, username='member98', age=98}
member = Member{id=98, username='member97', age=97}
member = Member{id=97, username='member96', age=96}
member = Member{id=96, username='member95', age=95}
```

페이징 API를 사용하면 자동으로 DB에 맞는 적절한 방언을 사용하여 페이징 쿼리가 구성된다.

```bash
# MySQL 방언
SELECT
    M.ID AS ID,
    M.AGE AS AGE,
    M.TEAM_ID AS TEAM_ID,
    M.NAME AS NAME
FROM
    MEMBER M
ORDER BY
    M.NAME DESC LIMIT ?, ?

# Oracle 방언
SELECT * FROM
    ( SELECT ROW_.*, ROWNUM ROWNUM_
    FROM
        ( SELECT
            M.ID AS ID,
            M.AGE AS AGE,
            M.TEAM_ID AS TEAM_ID,
            M.NAME AS NAME
        FROM MEMBER M
        ORDER BY M.NAME
        ) ROW_
    WHERE ROWNUM <= ?
    )
WHERE ROWNUM_ > ?
```

### 5) 조인

JPQL에서는 내부 조인, 외부 조인, 세타 조인을 지원한다.  
외래키 대신 참조를 명시하는 식으로 JOIN 구문을 작성하여, SQL에 비해 좀 더 객체스럽게 표현된다.
`SELECT m FROM Member m inner join m.team t`

먼저 내부 조인(INNER JOIN)은 Member는 있고 그 안에 Team은 없는 경우, Member, Team 둘다 조회되지 않는 식으로 동작하는 JOIN 문이다.

```java
String query = "select m from Member m inner join m.team t";
List<Member> result = em.createQuery(query, Member.class)
        .getResultList();
```

```bash
Hibernate:
    /* select m from Member m inner join m.team t */
        select
            member0_.MEMBER_ID as MEMBER_I1_6_,
            …
        from
            Member member0_
        inner join
            Team team1_
                on member0_.TEAM_ID=team1_.TEAM_ID
```

외부 조인(OUTER JOIN)은 Member는 있고 그 안에 Team은 없는 경우, Team은 Null이 들어가고 Member는 조회되는 식으로 동작한다.

```java
String query = "select m from Member m left join m.team t";
List<Member> result = em.createQuery(query, Member.class)
        .getResultList();
```

```bash
Hibernate:
    /* select m from Member m left join m.team t */
        select
            member0_.MEMBER_ID as MEMBER_I1_6_,
            …
        from
            Member member0_
        left outer join
            Team team1_
                on member0_.TEAM_ID=team1_.TEAM_ID
```

마지막으로 세타 조인은 @Id로 지정한 식별자 외의 칼럼을 이용하여 묶어서 가져올 수 있도록 지원한다.  
예를 들어서 회원 이름과 팀 이름이 같은 엔티티들을 조회하는 것이 가능하다.

```java
String query = "select m from Member m, Team t where m.username = t.name";
List<Member> result = em.createQuery(query, Member.class)
        .getResultList();
```

```bash
Hibernate:
    /* select m from Member m, Team t where m.username = t.name */
        select
            member0_.MEMBER_ID as MEMBER_I1_6_,
            …
        from
            Member member0_ cross
        join
            Team team1_
        where
            member0_.USERNAME=team1_.name
```

JPA 2.1부터는 JOIN 절에 ON 절을 활용하는 것이 가능하다.  
이를 통해 조인 대상이 되는 열을 ON 절 조건으로 필터링할 수 있다.  
아래는 회원과 팀을 조인하면서, 팀 이름이 teamA인 팀만 조인하는 예시이다.  
ON 절에 외래키인 member.teamId에 대한 일치 조건이 작성되고, 여기에 팀 이름에 대한 조건이 추가되었다.

```java
// src/main/java/hellojpa/JPAMain
String query = "select m from Member m left join m.team t on t.name = 'teamA'";
List<Member> result = em.createQuery(query, Member.class)
        .getResultList();
```

```bash
Hibernate:
    /* select m from Member m left join m.team t on t.name = 'teamA' */
        select
                member0_.MEMBER_ID as MEMBER_I1_6_,
                …
        from
            Member member0_
        left outer join
            Team team1_
                on member0_.TEAM_ID=team1_.TEAM_ID  # 기본 JOIN 조건
                and (    # ON 절에서 명시한 조건이 and 절로 추가됨
                    team1_.name='teamA'
                )
```

또는 ON절을 통해 연관관계가 없는 엔티티와 외부 조인(outer join)하는 것도 가능하다.  
아래는 회원의 이름과 팀의 이름이 같은 대상을 외부 조인하는 예시이다.

```java
// src/main/java/hellojpa/JPAMain
String query = "select m from Member m left join Team t on m.username = t.name";
List<Member> result = em.createQuery(query, Member.class)
        .getResultList();
```

```bash
Hibernate:
    /* select m from Member m left join Team t on m.username = t.name */
         select
                member0_.MEMBER_ID as MEMBER_I1_6_,
                …
        from
            Member member0_
        left outer join
            Team team1_
                on (  # ON 절에서 지정한 조건만 적용
                    member0_.USERNAME=team1_.name
                )
```

### 6) 서브 쿼리

서브쿼리는 쿼리 안에서 추가로 쿼리를 만들어서 사용하는 것이다.  
괄호 안에 서브 쿼리를 작성해서, 해당 쿼리의 결과를 이용하는 형식으로 본 쿼리를 작성한다.  
예를 들어 나이가 평균보다 많은 회원을 조회하는 쿼리를 다음과 같이 작성할 수 있다.

```sql
select m from Member m where m.age > (select avg(m2.age) from Member m2)
```

위 예시에서는 서브 쿼리 안에서 Member 엔티티에 대한 별칭인 m2를 새로 정의해서 사용했다.  
본 쿼리와 서브 쿼리에서 사용하는 데이터는 분리되므로, 새롭게 별칭을 정의할 수 있다.

한 건이라도 주문한 고객을 조회하는 쿼리는 다음과 같다.  
이 경우에는 본 쿼리에 있는 m을 가져와서 서브 쿼리의 where 절에 사용해야 한다.  
(다만 애초에 SQL에서 이런 식으로 서브쿼리를 작성하면 성능이 좋지 않음)

```sql
select m from Member m where (select count(o) from Order o where m = o.member) > 0
```

또한 여러 절들을 서브쿼리와 함께 사용해서 다양한 조건을 쿼리에 적용할 수 있다.  
먼저 EXISTS는 서브쿼리에 결과가 존재하면 True인 식으로 동작한다.  
예를 들어 팀A 소속인 회원을 조회한다고 하자.  
서브 쿼리 내에서는 특정 멤버가 속한 팀 중에서 이름이 팀A인 경우를 조회하고, EXISTS로 해당 서브쿼리의 결과가 존재하는지 확인하면 된다.

```sql
select m from Member m where exists (select t from m.team t where t.name = '팀A')
```

다음으로 ALL은 서브 쿼리의 결과들과 조합했을 때 조건이 모두 만족하면 True인 식으로 동작한다.  
예를 들어 전체 상품의 재고보다 주문량이 많은 주문들을 조회한다고 하자.  
이 때에는 서브쿼리에서 전체 제품의 재고(stockAmount)를 조회하고, where - ALL을 통해 각 주문의 주문량(orderAmoung)과 비교하면 된다.

```sql
select o from Order o where o.orderAmount > ALL (select p.stockAmount from Product p)
```

ANY, SOME: 조건을 하나라도 만족하면 True인 식으로 동작한다.  
예를 들어 어떤 팀이든 팀에 소속된 회원을 조회한다고 하자.  
이 때에는 서브쿼리에서 전체 팀을 조회한 다음, 각 멤버의 team 속성과 where ANY를 통해 동등한지를 체크하면 된다.

```sql
select m from Member m where m.team = ANY (select t from Team t)
```

이 외에도 IN을 사용해서 서브쿼리의 결과 중 하나라도 같은 것이 있으면 True인 식으로 로직을 구성할 수도 있다.  
EXISTS, IN은 NOT 조건과 함께 사용할 수 있다. (NOT EXISTS, NOT IN)

JPA의 서브쿼리 기능은 WHERE, HAVING 절에서만 사용이 가능하다는 한계가 있다.  
SELECT 절의 경우 하이버네이트에 한에서만 지원을 하고 있다.

```sql
select (select avg(m1.age) From Member m1) as avgAge
```

다음과 같은 FROM 절의 서브 쿼리는 현재 불가능하다.

```sql
select mm.age, mm.username from (select m.age, m.username from Member m) as mm
```

이런 형식의 서브쿼리는 가능하다면 조인으로 풀어서 해결해야 한다.  
조인으로 푸는 것이 불가능하다면 두 번 쿼리를 쪼개서 날린 다음 애플리케이션에서 조립하거나, Native SQL을 사용할 수 있다.

### 7) JPQL 타입 표현과 기타식

JPQL에서 각각의 데이터 타입을 표시하는 방법은 다음과 같다.

먼저 문자의 경우 single quotation을 이용해서 감싸서 표현하면 된다.  
ex) ‘HELLO’, ‘She’, ’s’  
숫자의 경우 10L(Long), 10D(Double), 10F(Float) 처럼 접미사를 이용해서 숫자의 타입을 표현하면 된다.  
Boolean은 TRUE, FALSE를 사용하면 되고, ENUM은 패키지명을 포함한 형태로 `jpabook.MemberType.Admin` 과 같이 작성해야 한다.

문자, Boolean, ENUM 타입을 이용해서 조회하는 예시는 다음과 같다.

```java
String query = "select m.username, 'HELLO', true from Member m " +
        "where m.memberType = hellojpa.MemberType.ADMIN";
List<Object[]> result = em.createQuery(query).getResultList();

for (Object[] objects : result) {
    System.out.println("objects = " + objects[0]);
    System.out.println("objects = " + objects[1]);
    System.out.println("objects = " + objects[2]);
}
```

```bash
objects = member
objects = HELLO
objects = true
```

만약 JPQL 내부에서 Enum의 전체 프로젝트 경로를 삽입하고 싶지 않다면, 다음과 같이 쿼리를 생성한 후 파라미터를 바인딩하는 방식을 사용할 수 있다.

```java
String query = "select m.username, 'HELLO', true from Member m where m.memberType = :type";
List<Object[]> result = em.createQuery(query)
        .setParameter("type", MemberType.ADMIN).getResultList();
```

다음으로 TYPE 키워드를 이용해서 엔티티 타입을 추출해내는 것도 가능하다.  
ex) `TYPE(m) = Member`  
TYPE 키워드는 보통 상속 관계에 있는 엔티티의 타입으로 조건을 적용할 때 사용한다.

```java
List<Item> resultList = em.createQuery("select i from Item i where type(i) = Book", Item.class)
        .getResultList();
```

```bash
Hibernate:
    /* select i from Item i where type(i) = Book */
        select
            item0_.id as id2_3_,
           …
        from
            Item item0_
        where
            item0_.DTYPE='b'  # 지정한 DiscriminatorValue에 따라서 쿼리가 나간다
```

지금까지 배운 문법들 외에도 JPQL은 SQL의 기본 기능을 거의 모두 제공한다.  
AND, OR, NOT 같은 키워드와 =, >, >=, <, <=, <> 같은 비교문도 동일하게 사용 가능하다.  
BETWEEN, LIKE, IS NULL 등의 키워드도 마찬가지로 지원한다.

```sql
select m.username from Member m where m.username is not null
select m.username from Member m where m.age between 0 and 10"
```

### 8) 조건식

JPQL은 다양한 조건식을 제공한다.
먼저 기본적인 CASE 식은 다음과 같이 case - when - else 키워드를 이용해 작성할 수 있다.

```sql
select
    case when m.age <= 10 then '학생요금'
        when m.age >= 60 then '경로요금'
        else '일반요금'
    end
from Member m
```

```bash
    select
        case
            when member0_.age<=10 then '학생요금'
            when member0_.age>=60 then '경로요금'
            else '일반요금'
        end as col_0_0_
    from
        Member member0_
```

만약 switch-case 처럼 각 조건을 단순 동등 비교로 구성한다면, 다음과 같이 CASE 식을 작성할 수도 있다.

```sql
select
    case t.name
        when '팀A' then '인센티브110%'
        when '팀B' then '인센티브120%'
        else '인센티브105%'
    end
from Team t
```

CASE 식 외에도 COALESCE, NULLIF 등의 조건식을 사용할 수 있다.  
COALESCE는 개별 값에 대해 적용하며, 지정한 값이 null인 경우에는 기본값을 반환한다.  
다음은 사용자 이름이 없으면 '이름 없는 회원' 문자열을 반환하는 예시이다.

```sql
select coalesce(m.username, '이름 없는 회원') from Member m
```

```bash
select
    coalesce(member0_.username,
    '이름 없는 회원') as col_0_0_
from
    Member member0_
```

NULLIF는 마찬가지로 개별 값에 적용하여, 지정한 두 값이 같으면 null을 반환하고 다르면 첫번째 값을 반환한다.  
다음은 사용자 이름이 ‘관리자’면 null을 반환하는 예시이다.

```sql
select NULLIF(m.username, '관리자') from Member m
```

```bash
select
    nullif (member0_.username,
    '관리자') as col_0_0_
from Member member0_
```

### 9) JPQL 함수

JPQL에서 제공하는 표준 함수들이 존재한다.  
각각의 SQL문에 매칭되기 때문에 각 db에 맞게 적절히 변환되어 쿼리를 사용할 수 있다.

먼저 CONCAT, SUBSTRING, TRIM, LOWER, UPPER, LENGTH, LOCATE 와 같은 기본 문자열 메서드들이 존재한다.

문자열을 합치는 함수인 CONCAT을 사용하는 예시는 다음과 같다.

```java
String query = "select concat('a', 'b') from Member m";
List<String> result = em.createQuery(query, String.class).getResultList();
for (String s : result) System.out.println("s = " + s);
```

```bash
# 두명의 회원이 등록되어 있음
s = ab
s = ab
```

부분 문자열을 추출하는 SUBSTRING은 다음과 같이 사용할 수 있다.

```java
String query = "select substring(m.username, 2, 3) from Member m";
List<String> result = em.createQuery(query, String.class).getResultList();
for (String s : result) System.out.println("s = " + s);
```

```bash
s = 리자1     # 관리자1
s = 리자2     # 관리자2
```

LOCATE는 특정 문자열의 위치를 찾아주는 함수이다.

```java
String query = "select locate('de', 'abcdefg') from Member m";
List<Integer> result = em.createQuery(query, Integer.class).getResultList();
for (Integer s : result) System.out.println("s = " + s);
```

```bash
s = 4
s = 4
```

이 외에도 ABS, SQRT, MOD 와 같이 숫자를 다루는 함수들도 존재한다.

SIZE, INDEX 함수는 JPA 전용으로 정의된 메서드이다.  
SIZE를 사용하면 특정 컬렉션의 크기를 구할 수 있다.

```java
String query = "select size(t.members) from Team t";
```

```bash
t = 0   # 팀에 등록된 멤버가 없으므로 t=0
```

INDEX는 값 타입 컬렉션에서 @OrderColumn을 지정해서 순서를 적용했을 때 사용할 수 있는 함수이다.  
다만 애초에 값 타입 컬렉션 사용을 권장하지 않기 때문에, 가능하면 INDEX도 사용하지 않는 것이 좋다.

JPQL 기본 함수로 해결이 안 되는 경우에는 직접 함수를 정의해서 호출하는 것도 가능하다.  
사용자 정의 함수를 사용할 때에는 직접 방언을 정의해야 한다.  
사용하는 DB 방언을 상속받고, 원하는 사용자 정의 함수를 등록한다.  
아래 예시에서는 group_concat 함수를 가져와서 방언에 등록한다.

```java
// src/main/java/dialect/MyH2Dialect
public class MyH2Dialect extends H2Dialect {
    public MyH2Dialect() {
        registerFunction("group_concat",
            new StandardSQLFunction("group_concat", StandardBasicTypes.STRING)
        );
    }
}
```

```xml
<!-- src/main/resources/META-INF/persistence.xml -->
<property name="hibernate.dialect" value="dialect.MyH2Dialect"/>
```

```java
// src/main/java/hellojpa/JpaMain
String query = "select function('group_concat', m.username) from Member m";
```

> group_concat은 기본 제공되는 함수가 아니기 때문에 "select group_concat(m.username) from Member m"로 작성하려면 IntelliJ 설정이 필요

```bash
s = 관리자1,관리자2
```
