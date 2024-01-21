### JPA 소개

JPA는 자바 엔터프라이즈 시장에서 주된 데이터 접근 기술이다.  
JPA는 ORM 기술로 객체에 맞게 SQL을 짜는 부분까지 자동화해주는 기술이다.  
보통 편의성이 더해진 스프링 데이터 JPA, QueryDsl을 사용하는 편이다.
다만 각 기술들 자체가 내용이 방대하기 때문에 제대로 활용하기 위해서는 학습이 필요하다.

### ORM 개념1 - SQL 중심 개발의 문제점

웹 어플리케이션 서버를 구성할 때 데이터 저장은 주로 RDB를 사용한다.  
이 때 어플리케이션의 객체를 DB에 저장하고, 또 DB의 내용을 읽어와서 객체로 만들어 사용하기 위해서는 그 중간에서 Mapper 역할을 하도록 코드를 개발해야 한다.

문제는 WAS에서 채택한 객체지향 패러다임과 RDB에서 사용하는 SQL의 패러다임이 불일치한다는 점에 있다.  
객체답게 모델링할 수록 Mapper 역할을 하는 코드가 복잡해진다.

대표적으로 객체의 상속 관계는 RDB에 잘 맞아떨어지지 않는다.  
슈퍼타입 - 서브타입 관계로 매핑하더라도 복잡하게 JOIN 쿼리를 작성해서 객체에 매핑해주는 코드를 작성해야 한다.  
결국 엔티티 객체의 경우 상속 관계로 잘 구현하지 않게 된다.

객체 연관관계 매핑 시에도 객체 참조로 연관관계를 맺고, 자유롭게 객체 그래프를 탐색할 수 있는 것이 적절하다.  
`member.getOrder().getDelivery();`  
하지만 SQL 에서는 SQL을 실행할 때 어디까지 JOIN 하느냐에 따라서 탐색 가능한 범위에 제한이 생긴다.  
그렇다고 객체 연관관계를 전부 매핑해서 불러올 수 없기 때문에, 필요한 조회 메서드를 여러벌로 중복해서 만들게 된다.

```java
memberDAO.getMember();
memberDAO.getMemberWithTeam();
memberDAO.getMemberWithOrderWithDelivery();
```

결국 서비스 단에서는 엔티티 객체를 신뢰할 수 없게 되고, 레포지토리 단에서 어디까지 SQL을 Join하는지를 직접 확인해보고 이에 맞게 로직을 구현해야 한다.  
이로 인해 계층 구분이 제대로 될 수 없다.

### ORM 개념2 - JPA 소개

JPA는 Java Persistence API로, 자바 진영의 ORM 표준 기술이다.  
JPA는 하나의 인터페이스 표준안이고, 자바에서 ORM을 사용할 때에는 JPA를 구현한 구현 기술을 사용하게 된다.

ORM은 Object Relational Mapping의 약자로, 객체는 객체대로 DB는 DB대로 각각 설계하고, ORM 기술이 이를 중간에서 매핑해주는 기술이다.  
JAVA 어플리케이션에서는 DB를 직접 다루는 대신 JPA를 사용하게 되고, JPA 내부에서 JDBC API를 사용해서 SQL을 실행하고 결과를 매핑하게 된다.

데이터 저장 시 어플리케이션은 엔티티 객체를 JPA에 전달하게 된다.  
JPA에서는 전달받은 객체를 분석해서 SQL을 만들고, JDBC API를 이용해서 SQL을 실행하게 된다.

데이터 조회 시에는 id 등의 식별자를 JPA에 넘기면 내부에서 Select SQL을 생성해서 JDBC API를 이용해 실행한다.  
조회 결과의 ResultSet은 엔티티 객체에 적절히 매핑해서 반환한다.

JPA를 사용할 경우 컬렉션을 이용해 데이터를 다루는 것처럼 매우 간단하게 db 작업을 할 수 있기 때문에 생산성이 향상된다.

```java
jpa.persist(member) // 저장
Member member = jpa.find(memberId) // 조회
member.setName(“변경할 이름”) // 수정
jpa.remove(member) // 삭제
```

또한 테이블에 필드를 추가하는 등 유지보수가 필요할 때에도, 이에 맞게 SQL을 매핑하는 작업은 JPA가 모두 수행하기 때문에 매우 편리하다.

그리고 JPA의 주요한 장점 중 하나가 객체 지향과 SQL 사이의 패러다임 불일치를 해결해준다는 점이다.  
대표적으로 상속 관계의 경우, 슈퍼타입 - 서브타입 관계로 매핑 시 조회/저장 시 자동으로 필요한 SELECT 문과 JOIN 문을 작성해준다.

```
jpa.persist(album);
-> INSERT INTO ITEM ...
   INSERT INTO ALBUM ...

Album album = jpa.find(Album.class, albumId);
-> SELECT I.*, A.*
    FROM ITEM I
    JOIN ALBUM A ON I.ITEM_ID = A.ITEM_ID
```

또한 연관관계 매핑의 경우에도, 객체 참조로 연관관계를 맺으면 자동으로 외래키 값으로 테이블 간 관계를 맺어주는 식으로 매핑해준다.  
추가적으로 자유로운 객체 그래프 탐색을 지원해서, 엔티티를 신뢰할 수 있게 한다.

```java
// 연관관계 저장
member.setTeam(team);
jpa.persist(member);

// 객체 그래프 탐색
Member member = jpa.find(Member.class, memberId);
Team team = member.getTeam();
Delivery delivery = member.getOrder().getDelivery();
```

객체 비교의 경우에도, JPA는 동일한 트랜잭션 내에서 조회한 동일한 데이터의 경우 같은 객체로 인식하도록 지원한다.  
이는 캐시를 이용해서 객체를 반환하기 때문에 가능한 것으로, 성능상 이점도 있다.

```java
Long memberId = 100L;
Member member1 = jpa.find(Member.class, memberId); // SQL 실행
Member member2 = jpa.find(Member.class, memberId); // 캐시에서 조회

member1 == member2; // true
```

성능 관점에서 JPA는 Insert에서 쓰기 지연을 지원한다.  
트랜잭션을 커밋하기 전까지는 insert 쿼리를 모아두고, JDBC BATCH SQL 기능을 사용해서 한번에 SQL을 실행한다.

```java
transaction.begin(); // [트랜잭션] 시작

em.persist(memberA);
em.persist(memberB);
em.persist(memberC);
// 여기까지 INSERT SQL을 데이터베이스에 보내지 않는다.

// 커밋하는 순간 데이터베이스에 INSERT SQL을 모아서 보낸다.
transaction.commit(); // [트랜잭션] 커밋
```

또한 JPA는 지연 로딩, 즉시 로딩을 둘 다 지원한다.
지연 로딩의 경우 연관관계의 객체 참조 시 해당 객체를 실제로 사용할 때에 SQL을 실행해서 데이터를 가져온다.  
즉시 로딩의 경우 성능 최적화가 필요할 때 사용하며, 처음부터 JOIN을 통해 연관관계의 객체를 함께 조회해온다.

```java
// 지연로딩
Member member = memberDAO.find(memberId); // -> SELECT * FROM MEMBER
Team team = member.getTeam();
String teamName = team.getName(); // -> SELECT * FROM TEAM

// 즉시로딩
Member member = memberDAO.find(memberId);
// -> SELECT M.*, T.* FROM MEMBER M JOIN TEAM T ON M.TEAM_ID = T.TEAM_ID
Team team = member.getTeam();
String teamName = team.getName();
```

### JPA 설정

spring-boot-starter-data-jpa를 사용하면 스프링 부트에 JPA, 스프링 데이터 JPA를 통합해서 사용할 수 있다.  
build.gradle에 다음과 같이 의존성을 추가한다.

```groovy
// JPA, 스프링 데이터 JPA 추가
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
```

이제 다음과 같은 라이브러리들이 추가되면서 JPA를 사용할 수 있게 된다.

- hibernate-core : JPA 구현체인 hibernate 라이브러리
- jakarta.persistence-api : JPA 인터페이스
- spring-data-jpa : 스프링 데이터 JPA 라이브러리

또한 application.properties에 다음 설정들을 추가해서 로깅을 남긴다.  
logging.level.org.hibernate.SQL은 실행되는 SQL에 대한 로깅을, logging.level.org.hibernate.type.descriptor.sql.BasicBinder는 바인딩되는 파라미터에 대한 로깅을 남기는 설정이다.

```properties
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

### JPA 적용1 - 개발

JPA를 사용하려면 먼저 객체와 테이블을 매핑해야 한다.  
JPA가 제공하는 어노테이션을 사용하여 매핑을 진행한다.  
JPA의 엔티티에는 public/protected 기본 생성자가 필수 이므로 추가한다.

```java
package hello.itemservice.domain;

@Data
@Entity
public class Item {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "item_name", length = 10)
    private String itemName;
    private Integer price;
    private Integer quantity;

    public Item() {
    }

    public Item(String itemName, Integer price, Integer quantity) {
        this.itemName = itemName;
        this.price = price;
        this.quantity = quantity;
    }
}
```

@Entity 어노테이션을 붙이면 JPA에서 해당 객체를 엔티티로 인식하게 된다.  
@Id를 붙여서 테이블의 pk를 해당 필드와 매핑하고, @GeneratedValue(strategy = GenerationType.IDENTITY)로 지정해서 db에서 자동으로 생성된 값을 사용하도록 한다.  
@Column(name = "item_name", length = 10)을 통해서는 객체의 필드와 테이블의 컬럼을 매핑했다.  
length = 10 과 같은 정보를 지정하면 create table시 해당 정보를 반영하여 테이블을 생성하게 된다.

이제 JPA를 사용해서 데이터를 관리하는 Repository를 구현해보자.  
JPA를 사용하기 위해서는 생성자에서 EntityManager를 주입받아야 한다.  
스프링 컨테이너에서는 데이터소스를 사용해서 EntityManager를 빈으로 등록하게 되고, 레포지토리에서는 이를 이용해서 데이터를 조회, 수정, 삭제할 수 있다.

또한 아래의 예시에서는 레포지토리 객체 자체에 @Transactional을 적용해두었다.  
JPA의 데이터 수정(등록, 수정, 삭제)는 모두 트랜잭션 내에서 이루어지기 때문에 트랜잭션 적용이 필수적이다.  
다만 일반적인 경우에는 서비스 단에서 트랜잭션을 적용하는 것이 좋다.

```java
package hello.itemservice.repository.jpa;

@Slf4j
@Repository
@Transactional
public class JpaItemRepository implements ItemRepository {

    private final EntityManager em;

    public JpaItemRepository(EntityManager em) {
        this.em = em;
    }

    @Override
    public Item save(Item item) {
        em.persist(item);
        return item;
    }

    @Override
    public void update(Long itemId, ItemUpdateDto updateParam) {
        Item findItem = em.find(Item.class, itemId);
        findItem.setItemName(updateParam.getItemName());
        findItem.setPrice(updateParam.getPrice());
        findItem.setQuantity(updateParam.getQuantity());
    }

    @Override
    public Optional<Item> findById(Long id) {
        Item item = em.find(Item.class, id);
        return Optional.ofNullable(item);
    }

    @Override
    public List<Item> findAll(ItemSearchCond cond) {
        String jpql = "selectxxx i from Item i";

        Integer maxPrice = cond.getMaxPrice();
        String itemName = cond.getItemName();

        if (StringUtils.hasText(itemName) || maxPrice != null) {
            jpql += " where";
        }

        boolean andFlag = false;
        List<Object> param = new ArrayList<>();
        if (StringUtils.hasText(itemName)) {
            jpql += " i.itemName like concat('%',:itemName,'%')";
            param.add(itemName);
            andFlag = true;
        }

        if (maxPrice != null) {
            if (andFlag) {
                jpql += " and";
            }
            jpql += " i.price <= :maxPrice";
            param.add(maxPrice);
        }

        log.info("jpql={}", jpql);

        TypedQuery<Item> query = em.createQuery(jpql, Item.class);
        if (StringUtils.hasText(itemName)) {
            query.setParameter("itemName", itemName);
        }
        if (maxPrice != null) {
            query.setParameter("maxPrice", maxPrice);
        }
        return query.getResultList();
    }
}
```

JPA를 사용하기 위한 Config를 다음과 같이 작성한다.

```java
package hello.itemservice.config;

@Configuration
public class JpaConfig {

    private final EntityManager em;

    public JpaConfig(EntityManager em) {
        this.em = em;
    }

    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }

    @Bean
    public ItemRepository itemRepository() {
        return new JpaItemRepository(em);
    }
}
```

EntityManager를 주입받아서 JpaItemRepository를 생성하고 있다.  
JpaTransactionManager, Datasource를 이용해서 EntityManagerFactory를 이용해서 EntityManager를 생성하는 작업은 스프링 컨테이너 내부에서 자동으로 수행된다.

이제 해당 Config가 적용되도록 SpringApplication에 어노테이션을 추가한다.

```java
package hello.itemservice;

@Slf4j
@Import(JpaConfig.class)
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

### 레포지토리 코드 분석

먼저 객체 저장 시에는 em.persist()를 사용한다.  
실행되는 sql을 확인해보면 id 값이 빠진 채로 insert 쿼리가 실행되고 있다.  
엔티티에 지정한 전략에 따라 id 값은 db에서 자동으로 생성되어 삽입되고, jpa 단에서 쿼리 실행 후에 id 값을 db에서 받아와서 객체에 넣어준다.

```java
public Item save(Item item) {
    em.persist(item);
    return item;
}

insert into item (id, item_name, price, quantity) values (null, ?, ?, ?) 또는
insert into item (id, item_name, price, quantity) values (default, ?, ?, ?) 또는
insert into item (item_name, price, quantity) values (?, ?, ?)
```

객체 수정 시에는 엔티티 매니저를 통해 객체를 조회해서 값을 변경해주면 된다.  
JPA에서는 트랜잭션이 커밋되는 시점에 영속성 컨텍스트에서 변경된 엔티티 객체가 있는지 확인하게 되고, 변경이 존재한다면 update 쿼리를 실행하는 식으로 동작한다.

> 테스트 환경에서는 트랜잭션이 롤백되기 때문에 @Commit을 붙여야 update 쿼리를 확인할 수 있다.

```java
public void update(Long itemId, ItemUpdateDto updateParam) {
    Item findItem = em.find(Item.class, itemId);
    findItem.setItemName(updateParam.getItemName());
    findItem.setPrice(updateParam.getPrice());
    findItem.setQuantity(updateParam.getQuantity());
}

update item set item_name=?, price=?, quantity=? where id=?
```

JPA에서 PK를 이용해 단건 조회를 할 때에는 em.find()를 사용한다.  
실행된 sql에서는 Join이 적용된 복잡한 상황에서도 잘 조회를 하기 위해서 컬럼명, 테이블명 등에 별칭을 적용된다.

```java
public Optional<Item> findById(Long id) {
    Item item = em.find(Item.class, id);
    return Optional.ofNullable(item);
}

select
  item0_.id as id1_0_0_,
  item0_.item_name as item_nam2_0_0_,
  item0_.price as price3_0_0_,
  item0_.quantity as quantity4_0_0_
from item item0_
where item0_.id=?
```

여러 조건을 적용해서 목록 조회를 할 떄에는 JPQL을 사용해서 조회해야 한다.  
JPQL은 객체지향 쿼리 언어로써, 엔티티 객체를 대상으로 SQL을 실행하는 개념이다.  
엔티티 테이블이 아닌 객체의 이름과 컬렴명을 이용하여 JPQL을 작성해야 한다.

```java
public List<Item> findAll(ItemSearchCond cond) {
    String jpql = "select i from Item i";
    //동적 쿼리 생략
    TypedQuery<Item> query = em.createQuery(jpql, Item.class);
    return query.getResultList();
}

// JPQL
select i from Item i
where i.itemName like concat('%',:itemName,'%')
  and i.price <= :maxPrice

// SQL
select
  item0_.id as id1_0_,
  item0_.item_name as item_nam2_0_,
  item0_.price as price3_0_,
  item0_.quantity as quantity4_0_
from item item0_
where (item0_.item_name like ('%'||?||'%'))
  and item0_.price<=?
```

> JPA를 직접 사용하면 동적 쿼리를 구성하기가 여전히 복잡한 편이다.  
> QueryDSL을 사용하면 보다 편리하게 복잡한 쿼리를 구성할 수 있다.

### JPA 적용3 - 예외 변환

위 예시에서는 EntityManager를 직접 이용해서 JPA를 사용했는데, 이러면 JPA 관련 예외가 발생된다.  
JPA는 PersistenceException, IllegalStateException, IllegalArgumentException을 발생시킬 수 있는데, 이러한 예외들을 레포지토리에서 그대로 사용하면 서비스 단에서도 예외 핸들링 과정에서 JPA에 종속된 코드를 작성하게 된다.

이러한 문제를 해결하기 위해 레포지토리에는 @Repository 어노테이션을 달아야 한다.  
@Repository가 붙은 객체는 스프링 빈으로 등록될 뿐만 아니라, 예외 변환 AOP의 적용 대상이 된다.  
레포지토리 단에서 JPA 예외가 발생하면, 스프링과 JPA를 사용할 때 자동으로 등록되는 PersistenceExceptionTranslator를 통해 스프링 데이터 접근 예외로 변환되어 상위 계층으로 던져진다.
