### 0) 개요

다양한 검색조건을 사용해서 데이터를 가져오기 위해서는 복잡한 쿼리를 짤 수 있어야 한다.  
JPA는 다양한 기술들을 통해서 쿼리 방법을 지원한다.

- JPQL(Java Persistence Query Language)
  - 직접 사용하거나, JPA Criteria/QueryDSL 등의 JPQL 빌더를 사용
- 네이티브 SQL
  - 특정 데이터베이스 vendor에 종속되는 쿼리를 작성해야 하는 경우 사용
- JDBC API, JdbcTemplate, MyBatis 등을 함께 사용하는 것도 가능

### 1) JPQL 소개

JPA를 사용하면 EntityManager.find()를 사용하거나, a.getB().getC() 처럼 객체 그래프를 탐색하여 데이터를 가져올 수 있다.  
이를 통해 엔티티 객체를 중심으로 개발하는 것이 가능하다.

다만 위 방법들만으로는 `나이가 18살 이상인 회원` 처럼 특정 조건이 적용된 검색 쿼리를 구성하는 것은 불가능하다.  
필요한 데이터만 DB에서 불러오려면 WHERE나 GROUPBY 등의 구문이 적용된 SQL이 필요하다.  
JPA는 검색을 할 때에도 테이블이 아닌 엔티티 객체를 대상으로 검색해야 하는데, 그렇다고 해서 모든 DB 데이터를 불러와서 객체로 변환해서 검색할 수는 없다.

JPA는 SQL을 추상화한 JPQL이라는 객체 지향 쿼리 언어를 제공한다.  
SQL 문법과 유사하게 SELECT, FROM, WHERE, GROUP BY, HAVING, JOIN 등의 기능을 지원한다.  
이 때 SQL은 데이터베이스 테이블을 대상으로 쿼리를 하는 반면에, JPQL은 엔티티 객체를 대상으로 쿼리를 한다.  
SQL을 추상화했기 때문에 JPA처럼 특정 데이터베이스 SQL에 의존하지 않는다.

```java
/**
 * JPQL을 이용한 검색
 */
// JPQL에서 Member는 MEMBER 테이블이 아닌, 엔티티 Member를 가리킴
// m: alias로 member 객체 자체를 가리킴
String jpql = "select m From Member m where m.username like '%kim%'";
List<Member> result = em.createQuery(jpql,Member.class).getResultList();
for (Member member : result) {
    System.out.println("member = " + member);
}
```

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
Criteria를 통해 문자가 아닌 자바코드로 JPQL을 작성할 수 있다.

다만 너무 복잡하고 유지보수가 어려워서 실용성이 떨어진다.  
보통은 Criteria 대신에 QueryDSL을 사용하는 것을 권장한다.

```java
//Criteria 사용 준비
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Member> query = cb.createQuery(Member.class);
//루트 클래스 (조회를 시작할 클래스)
Root<Member> m = query.from(Member.class);
//쿼리 생성
CriteriaQuery<Member> cq =query.select(m).where(cb.equal(m.get("username"), "kim"));
List<Member> resultList = em.createQuery(cq).getResultList();
```

#### QueryDSL 소개

QueryDSL은 Criteria 같은 JPQL 빌더로, 문자가 아닌 자바코드로 JPQL을 작성할 수 있도록 지원한다.  
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
String sql = "SELECT ID, AGE, TEAM_ID, NAME FROM MEMBER WHERE NAME = 'kim'";  // 쌩 쿼리를 작성
List<Member> resultList = em.createNativeQuery(sql, Member.class).getResultList();
```

#### JDBC 직접 사용, JdbcTemplate

JPA를 사용하는 동시에, JDBC 커넥션을 직접 다루거나 스프링 JdbcTemplate, MyBatis등을 함께 사용하는 것도 가능하다.

단 외부 기술을 사용하기 전에 영속성 컨텍스트를 강제로 플러시해야 할 수 있다.  
JPA는 플러시하기 전이라도 1차 캐시를 통해 영속성 컨텍스트에 존재하는 데이터를 조회 할 수 있지만, JPA에서 관리하지 않는 기술들은 접근이 불가능하다.  
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

JPQL SQL을 추상화한 것으로, 결국 SQL로 변환되어 실행된다.  
JPQL의 예시는 다음과 같다.

`select m from Member as m where m.age > 18`

JPQL 작성 시 엔티티와 속성은 대소문자를 구분하고(Member, age), JPQL 키워드는 대소문자를 구분하지 않는다. (SELECT, FROM, where)
또한 테이블 이름이 아닌 엔티티 이름(Member)을 사용하고, 엔티티에 대한 별칭은 필수이다.(m)  
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
update_절 [where_절]

# DELETE 문
delete_문 :: = delete_절 [where_절]
```
