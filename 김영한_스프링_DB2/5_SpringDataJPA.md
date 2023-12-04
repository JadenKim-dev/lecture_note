### 소개1 - 등장 이유

Spring Data는 RDB뿐만 아니라 Redis, Mongo 등 다양한 데이터 저장소 기술이 등장하면서 만들어졌다.  
여러 종류의 저장소를 동일한 인터페이스를 이용하여 접근할 수 있게 추상화했다.  
동일한 인터페이스로 여러 종류의 저장소의 쿼리를 생성하고, 페이징 처리도 지원하다.  
메서드명 만으로 적절한 쿼리를 생성하는 등 상당한 편의성을 제공한다.

스프링 데이터는 원하는 저장소 및 기술에 통합된 버전을 사용하면 된다. (스프링 데이터 JPA, 스프링 데이터 몽고...)  
다만 이들을 제대로 사용하려면 스프링 데이터 뿐만 아니라, 해당 구현 기술 자체도 잘 이해하고 있어야 한다.

### 소개2 - 기능

스프링 데이터 JPA를 사용하면 JpaRepository를 상속 받은 인터페이스를 정의해야 한다.  
이 때 인터페이스 내부에 아무런 메서드를 정의하지 않아도 자동으로 등록, 조회, 삭제, 수정 메서드가 만들어진다.  
이는 프록시을 이용하여 해당 인터페이스의 구현체가 동적으로 생성되어 등록되기 때문에 가능하다.  

또한 기본 등록되는 쿼리 메서드 외에도, 적절하게 메서드명만 인터페이스에 정의해두면 해당 이름에 맞게 쿼리를 작성해준다.  
직접 쿼리를 입력하고 싶다면 @Query 어노테이션을 달아서 JPQL을 직접 작성할 수도 있다.

스프링 데이터 JPA를 통해 컴퓨터가 자동으로 해결해줄 수 있는 부분이 상당 부분 자동화된다.  
JPA - Spring Data JPA - QueryDsl 조합은 통합성 있게 잘 조합되기 때문에 깔끔하게 로직을 구성할 수 있다.  
이를 통해 코드량 자체가 줄어서 생산성이 향상되고, 도메인 클래스 자체에 공을 들여서 짜게 된다.  
직접 쿼리를 확인해야 하는 일이 줄어서 비즈니스 로직 이해가 쉬워지고, sql 의존적인 코드가 줄어들어서 테스트 케이스 작성도 보다 편리해진다.

다만 명심해야 할 것은 Spring Data JPA는 RDB, Jdbc, JPA 기반에 있는 기술이라는 점이다.  
이들을 제대로 이해하지 못한 채로 사용하면 오류가 생겨도 원인 파악이 어렵다.
결국 JPA를 제대로 이해하는 것이 가장 중요하다.

### 주요 기능

Spring Data 자체는 여러 기술들에 공통적인 기능들이 담겨 있고, 스프링 데이터 JPA, 스프링 데이터 몽고 등에는 각 기술에 특화적인 기능이 추가되어 있다.  
제공하는 주요 기술은 두 가지이다 - **공통 인터페이스 기능**, **쿼리 메서드 기능**

#### 1. 공통 인터페이스 기능

JpaRepository에는 공통적으로 사용할 수 있는 기본 기능들이 거의 모두 포함되어 있다.  
등록 조회 수정 삭제 뿐만 아니라 페이징, exists 등 다양한 메서드가 기본 제공된다.

JpaRepository는 다음과 같이 인터페이스를 상속해서 정의한다.  
제네릭에는 관리할 <엔티티, 엔티티ID> 형식으로 주면 된다.

```java
public interface ItemRepository extends JpaRepository<Item, Long> {
}
```

이렇게만 작성해두면 스프링 데이터 JPA에서 해당 인터페이스를 참고하여 동적 프록시 기술을 통해 구현체를 만들어준다.  
해당 구현체는 스프링 컨테이너에 등록되기 때문에, 이를 의존성 주입 받아서 사용하면 된다.

#### 2. 쿼리 메서드 기능

또한 Spring Data JPA는 인터페이스에 메서드명만 적어두면, 이를 바탕으로 쿼리를 자동 생성해주는 기능을 제공한다.

예를 들어 특정 유저네임을 가지고, 나이가 특정 값 이상인 멤버를 조회하고 싶다고 하자.  
순수 JPA로 작성한다면 다음과 같이 JPQL을 직접 작성하고, 파라미터를 바인딩 해야 한다.  

```java
public List<Member> findByUsernameAndAgeGreaterThan(String username, int age) {
      return em.createQuery("select m from Member m where m.username = :username and m.age > :age")
              .setParameter("username", username)
              .setParameter("age", age)
              .getResultList();
```

하지만 스프링 데이터 JPA를 사용하면 다음과 같이 인터페이스에 적절한 메서드명으로 정의만 하면 된다. 
스프링 데이터 JPA 내부에서는 해당 메서드명을 분석해서 적절한 JPQL을 만들어서 실행해준다.
```java
public interface MemberRepository extends JpaRepository<Member, Long> {
    List<Member> findByUsernameAndAgeGreaterThan(String username, int age);
}
```

다음과 같은 다양한 쿼리 메서드 기능을 사용할 수 있다.

- 조회: find...By , read...By , query...By , get...By
       findHelloBy 처럼 식별하기 위한 내용(설명)이 들어가도 된다. 
- COUNT: count...By -> 반환타입 long
- EXISTS: exists...By -> 반환타입 boolean
- 삭제: delete...By , remove...By -> 반환타입 long
- DISTINCT: findDistinct , findMemberDistinctBy 
- LIMIT: findFirst3 , findFirst , findTop , findTop3

하지만 조건이 많이 붙는 복잡한 쿼리를 작성해야 할 때에는 메서드 명이 너무 길어질 수 있다.  
이 때에는 @Query에 실행할 JPQL을 직접 작성할 수도 있다.

```java
public interface SpringDataJpaItemRepository extends JpaRepository<Item, Long> {
    //쿼리 메서드 기능
    List<Item> findByItemNameLike(String itemName);

    //쿼리 직접 실행
    @Query("select i from Item i where i.itemName like :itemName and i.price <= :price")
        List<Item> findItems(@Param("itemName") String itemName, @Param("price") Integer price);
    }
}
```





