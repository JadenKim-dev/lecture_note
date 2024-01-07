### 프로젝트 생성

JPA를 사용할 때 persistence.xml 파일을 작성하여 관련 설정을 할 수 있다.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence version="2.2"
             xmlns="http://xmlns.jcp.org/xml/ns/persistence" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence http://xmlns.jcp.org/xml/ns/persistence/persistence_2_2.xsd">
    <persistence-unit name="hello">  <!--persistence unit 이름-->
        <properties>
            <property name="javax.persistence.jdbc.driver" value="org.h2.Driver"/>
            <property name="javax.persistence.jdbc.user" value="sa"/>
            <property name="javax.persistence.jdbc.password" value=""/>
            <property name="javax.persistence.jdbc.url" value="jdbc:h2:tcp://localhost/~/test"/>
            <property name="hibernate.dialect" value="org.hibernate.dialect.H2Dialect"/>
            <!-- 옵션 -->
            <property name="hibernate.show_sql" value="true"/>  <!-- db에 나가는 쿼리를 보여줌 -->
            <property name="hibernate.format_sql" value="true"/>
            <property name="hibernate.use_sql_comments" value="true"/>
            <!--<property name="hibernate.hbm2ddl.auto" value="create" />-->
        </properties>
    </persistence-unit>
</persistence>
```

#### 데이터베이스 방언 (dialect)

데이터베이스 방언은 SQL 표준을 지키지 않는 특정 데이터베이스만의 고유한 기능이다.  
이로 인해 각각의 데이터베이스가 제공하는 SQL 문법과 함수는 조금씩 다르다.

- 가변 문자: MySQL VARCHAR, Oracle VARCHAR2
- 페이징: MySQL LIMIT , Oracle ROWNUM
- 문자열을 자르는 함수: SQL SUBSTRING(), Oracle SUBSTR()

하지만 JPA는 특정 데이터베이스에 종속되지 않는다.  
hibernate.dialect 속성에 지정 시 알아서 해당하는 db의 방언에 맞춰서 바꿔준다.  
`<property name="hibernate.dialect" value="org.hibernate.dialect.H2Dialect"/>`

<img src="./images/2_JPA_Start1.png" width="400">

하이버네이트는 40가지 이상의 데이터베이스 방언을 지원한다.

- H2 : `org.hibernate.dialect.H2Dialect`
- Oracle 10g : `org.hibernate.dialect.Oracle10gDialect`
- MySQL : `org.hibernate.dialect.MySQL5InnoDBDialect`

### 애플리케이션 개발

<img src="./images/2_JPA_Start2.png" width="400">

Persistence는 persistence.xml에 작성한 설정정보를 조회해서, EntityManagerFactory를 생성한다.  
해당 EntityManagerFactory에서 EntityManager를 생성하게 되고, 개발자는 EntityManager를 이용하여 데이터를 다루게 된다.

예제 확인을 위해 먼저 다음의 sql 문으로 테이블을 생성한다.

```sql
create table Member (
  id bigint not null,
  name varchar(255),
  primary key (id)
);
```

이제 엔티티 객체를 정의해서 객체와 테이블을 매핑한다.

```java
// src/main/java/hellojpa/Member
@Entity
public class Member {
    @Id
    private Long id;

    private String name;

    // JPA 엔티티는 기본 생성자 작성이 필수
    public Member() {
    }
    public Member(Long id, String name) {
        this.id = id;
        this.name = name;
    }

    public void setId(Long id) { this.id = id; }
    public void setName(String name) { this.name = name; }
    public Long getId() { return id; }
    public String getName() { return name; }
}
```

이제 해당 엔티티를 이용해서 회원을 저장, 조회, 삭제, 수정해보자.

```java
// src/main/java/hellojpa/JpaMain

public class JpaMain {
    public static void main(String[] args) {
        // EntityManagerFactory는 애플리케이션 로딩시점에 하나만 만들어야 함
        // 읽어올 설정정보(persistence-unit)를 지정해서 생성
        EntityManagerFactory emf = Persistence.createEntityManagerFactory("hello");

        // EntityManager는 하나의 Transaction마다 새롭게 생성되어야 함
        // 따라서 고객의 요청이 올 때마다 생성 필요
        EntityManager em = emf.createEntityManager();

        // 모든 데이터 변경은 Transaction 안에서 수행
        // db transaction 시작
        EntityTransaction tx = em.getTransaction();
        tx.begin();

        try {
            /** 회원 등록 */
            Member member = new Member();
            member.setId(1L);
            member.setName("helloA");
            em.persist(member);

            /** 회원 조회 */
            Member findMember = em.find(Member.class, 1L);
            System.out.println("findMember.id = " + findMember.getId());
            System.out.println("findMember.name = " + findMember.getName());

            /** 회원 삭제 */
            em.remove(findMember);

            /** 회원 수정 */
            // JPA를 통해 조회, JPA에서 findMember를 관리한다
            Member findMember = em.find(Member.class, 1L);
            findMember.setName("helloJPA");

            // 엔티키가 변경 됐는지를 commit시점에서 체크하고, 변경이 있으면 update 뭐리를 날린다.
            tx.commit();

        } catch(Exception e) {
            tx.rollback();
        } finally {
            // entity manager 사용 후 close해야 db connection이 끊어진다.
            em.close();
        }
        emf.close();
    }
}
```

이 때 주의할 점이 있다.
EntityManagerFactory는 하나만 생성해서 애플리케이션 전체에서 공유해야 한다 
이와 달리 EntityManager는 thread 간에 공유되면 안 되고, 각 요청을 처리한 후 버려야 한다.  
또한 JPA의 모든 데이터 변경은 트랜잭션 안에서 실행해야 한다.

**JPQL 소개**

JPA를 사용하면 엔티티 객체를 중심으로 개발하게 된다.  
다만 검색 쿼리를 작성하는 것이 문제가 되는데, 필요한 데이터를 DB에서 불러오려면 검색 조건이 포함된 SQL이 필요하다.  
여러 테이블을 조인하고, 원하는 데이터를 최적화해서 가져오고, 필요하면 통계성 쿼리를 날리는 등의 복잡한 작업이 필요할 때가 있다.  
JPA의 JPQL을 사용하면 검색을 할 때도 테이블이 아닌 엔티티 객체를 대상으로 검색할 수 있다.  

```java
※ src/main/java/hellojpa/JpaMain
public class JpaMain {
    public static void main(String[] args) {
        try {
            // 테이블이 아닌 Member 엔티티 객체를 대상으로 쿼리를 구성한다.
            List<Member> result = em.createQuery("select m from Member as m", Member.class).getResultList();
            for(Member member: result) {
                System.out.println("member.name = " + member.getName());
            }
            tx.commit();
        } catch(Exception e) {
            tx.rollback();
        } finally {
            em.close();
        }
    }
}

```

JPA는 SQL을 추상화한 JPQL이라는 객체 지향 쿼리 언어 제공한다.  
JPQL은 SQL과 문법이 유사하고 SELECT, FROM, WHERE, GROUP BY, HAVING, JOIN등의 대부분의 쿼리 기능을 지원한다.

JPQL은 테이블이 아닌 객체를 대상으로 검색하는 객체 지향 쿼리이다. (객체 지향 SQL)  
JPQL은 SQL을 추상화했기 때문에 특정 데이터베이스의 SQL에 의존하지 않는다.  
JPQL 실행 시 해당 데이터베이스의 방언에 맞게 변형되어 쿼리가 날라간다.
