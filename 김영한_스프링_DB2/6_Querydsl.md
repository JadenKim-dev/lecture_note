### 소개1 - 기존 방식의 문제점

컴파일 에러는 빌드 및 실행하기 전에 문제가 있음을 미리 알 수 있지만, 런타임 에러는 직접 실행하고 나서야만 알 수 있다.  
JPQL 쿼리를 직접 작성하면 그 자체로는 문자열이기 때문에, 컨파일 타임에 타입 체크가 불가능하다.   
실제로 해당 로직을 실행하고 나서야 런타임 예외가 발생하기 때문에, 사전에 정상 작동 여부를 확인할 수 없다.  

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

DSL은 도메인 특화 언어로, 특정 도메인으로 표현력이 제한된 프로그래밍 언어의 종류이다.  
QueryDSL은 쿼리에 특화된 언어로, 단순하고 간결하다는 것이 특징이다.

QueryDSL은 자바 문법을 통해 type-safe하게 다양한 저장소(MongoDB, MySQL, Redis)의 쿼리를 생성해주는 것을 목표로 했다.
QueryDSL을 사용하면 엔티티 및 테이블 정보를 읽어서, 코드 생성기가 쿼리용 객체를 생성해준다.
어노테이션 기반으로 동작하기 때문에 Annotation Processing Tool이라고 하는데, JPA에서는 @Entity에 대해서 작동한다.

QueryDSL은 사실상 JPQL을 type-safe하게 작성하는 용도로 많이 사용된다.
위에서와 동일한 조건으로 쿼리를 작성해야 한다고 해보자.
먼저 다음과 같이 테이블을 만들고, 엔티티를 정의한다.

```sql
create table Member (
    id bigint auto primary key,
    age integer not null,
    name varchar(255)
)
```

```java
@Entity public class Member { 
    @Id @GeneratedValue
    private Long id;
    private String name;
    private int age;
    ...
}
```

이러면 APT에 의해서 다음과 같은 쿼리용 객체가 생성된다.

```java
@Generated
public class QMember extends EntityPathBase<Member> {
    public final NumberPath<Long> id = createNumber("id", Long.class); 
    public final NumberPath<Integer> age = createNumber("age", Integer.class); 
    public final StringPath name = createString("name");

    public static final QMember member = new QMember("member"); 
    ... 
}
```

이를 바탕으로 쿼리를 생성하는 코드를 다음과 같이 작성할 수 있다.

```java
JPAQueryFactory query = new JPAQueryFactory(entityManager);
QMember m = QMember. member;
List<Member> list = query
    .select(m)
    .from(m)
    .where(
        m.age.between(20, 40).and(m.name.like("Kim%")) 
    )
    .orderBy(m.age.desc())
    .limit(3)
    .fetch(m);
```

쿼리 기반의 단순하고 직관적인 인터페이스를 사용하고 있다.  
이를 이용하여 JPQL을 빌드하게 되고, 최종적으로 쿼리로 변환되어 db에 전달된다.  
APT를 사용하기 위한 설정만 초기에 하고 나면 손쉽게 라이브러리를 사용할 수 있다.

QueryDSL은 기본적인 JPQL의 기능을 모두 지원하고, 추가로 동적 쿼리를 편리하게 작성하는 기능을 제공한다.

```java
String firstName = "Kim";
int min=20,max=40;

BooleanBuilder builder = new BooleanBuilder();
if (StringUtils. hasText(str)) 
    builder.and(m.name.startsWith(firstName)); 

if (min != 0 && max != 0) 
    builder.and(m.age.between(min, max)); 

List<Member> results = query 
    .select(m) 
    .from(m) 
    .where(builder) 
    .fetch(m);
```

스프링 데이터 JPA는 조회에서 지원되는 기능이 부족한 편이다.  
QueryDSL은 복잡한 쿼리와 동적 쿼리를 편리하게 구성할 수 있게 지원하여 스프링 데이터 JPA를 보완해준다. 

### QueryDSL 설정

QueryDSL을 사용하기 위해서는 다음의 의존성들과 설정을 추가해야 한다.
스프링 설정이나 관련된 라이브러리들의 설정에 따라 추가하는 의존성이 달라질 수 있다.

```groovy
dependencies {
    //Querydsl 추가
    implementation 'com.querydsl:querydsl-jpa' annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jpa"
    annotationProcessor "jakarta.annotation:jakarta.annotation-api"
    annotationProcessor "jakarta.persistence:jakarta.persistence-api"
    ...

    testAnnotationProcessor 'org.projectlombok:lombok'
}

//Querydsl 추가, 자동 생성된 Q클래스 gradle clean으로 제거 clean {
    delete file('src/main/generated')
}
```

이제 Q타입 파일을 생성해야 하는데, 프로젝트 빌드 옵션에 따라 생성 방법이 달라진다.  
먼저 IntelliJ에서 프로젝트 빌드 방식은 다음의 설정에서 확인할 수 있다.  
Preferences - Build, Execution, Deployment - Build - Tools - Gradle

먼저 Gradle을 통해서 빌드하는 경우, Gradle -> Tasks -> build -> clean을 통해 기존 Q파일을 지워준다.  
그 후에 Gradle -> Tasks -> other -> compileJava를 통해 자바 파일을 컴파일하면 Q파일이 생성된다.
IntelliJ를 통해 직접 빌드하는 경우 Build -> Build Project 또는 Build -> Rebuild 통해 프로젝트를 빌드해야 한다.
또는 main() 함수를 직접 실행해도 Q파일이 생성된다.  
Q파일을 삭제하고 싶을 때에는 gradle clean 명령어를 실행하여 지울 수 있다.  

> Querydsl은 IntelliJ가 버전업하거나, 스프링이 버전업 하면서 환경 구성 방법이 달라지기도 한다.  
> 각 버전에 맞는 방법은 직접 찾아서 해야 한다.

### 프로젝트 적용

Querydsl을 사용하는 레포지토리를 정의해보자.  
기본적인 등록, 수정, 단건 조회는 기본 JPA 방식을 사용하고, 동적 쿼리를 적용해야 하는 목록 조회에 Querydsl을 사용한다.

```java
package hello.itemservice.repository.jpa;

import static hello.itemservice.domain.QItem.*;

@Repository
@Transactional
public class JpaItemRepositoryV3 implements ItemRepository {

    private final EntityManager em;
    private final JPAQueryFactory query;

    public JpaItemRepositoryV3(EntityManager em) {
        this.em = em;
        this.query = new JPAQueryFactory(em);
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

    public List<Item> findAllOld(ItemSearchCond cond) {

        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();

        QItem item = QItem.item;
        BooleanBuilder builder = new BooleanBuilder();
        if (StringUtils.hasText(itemName)) {
            builder.and(item.itemName.like("%" + itemName + "%"));
        }
        if (maxPrice != null) {
            builder.and(item.price.loe(maxPrice));
        }

        List<Item> result = query
                .select(item)
                .from(item)
                .where(builder)
                .fetch();

        return result;
    }

    @Override
    public List<Item> findAll(ItemSearchCond cond) {

        String itemName = cond.getItemName();
        Integer maxPrice = cond.getMaxPrice();

        return query
                .select(item)
                .from(item)
                .where(likeItemName(itemName), maxPrice(maxPrice))
                .fetch();
    }

    private BooleanExpression likeItemName(String itemName) {
        if (StringUtils.hasText(itemName)) {
            return item.itemName.like("%" + itemName + "%");
        }
        return null;
    }

    private BooleanExpression maxPrice(Integer maxPrice) {
        if (maxPrice != null) {
            return item.price.loe(maxPrice);
        }
        return null;
    }
}
```
Querydsl울 사용하기 위해서는 EntityManager를 주입받고, 이를 이용해서 JPAQueryFactory 객체를 생성해야 한다.
동적 쿼리 작성에서 BooleanBuilder를 이용해서 조건에 따라 where 문을 다르게 구성할 수 있다.  
다만 이 경우 직접 조건에 따라 판단을 하고, builder.and를 호출해서 where 조건을 추가해나가야 한다.  

이를 좀 더 최적화하면 itemName, price에 대한 BooleanExpression을 반환하는 메서드를 각각 정의할 수 있다.  
조건을 추가할 때는 해당하는 함수형 값을, 추가하지 않을 때에는 null을 반환하도록 구현하고, 해당 메서드의 반환값을 where에 그대로 사용하면 된다.

이제 해당 레포지토리를 사용하도록 Config를 작성하고, 어플리케이션에 설정한다.

```java
package hello.itemservice.config;

@Configuration
@RequiredArgsConstructor
public class QuerydslConfig {

    private final EntityManager em;

    @Bean
    public ItemService itemService() {
        return new ItemServiceV1(itemRepository());
    }

    @Bean
    public ItemRepository itemRepository() {
        return new JpaItemRepositoryV3(em);
    }

}
```

```java
package hello.itemservice;

import hello.itemservice.config.*;
import hello.itemservice.repository.ItemRepository;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Import;
import org.springframework.context.annotation.Profile;
import org.springframework.jdbc.datasource.DriverManagerDataSource;

import javax.sql.DataSource;


@Slf4j
@Import(QuerydslConfig.class)
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
