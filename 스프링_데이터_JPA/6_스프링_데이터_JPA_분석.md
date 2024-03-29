### 스프링 데이터 JPA 구현체 분석

스프링 데이터 JPA에서 사용하는 내부 구현체를 뜯어보면서 메커니즘을 확인해보자.
공통 인터페이스 기능의 경우 JpaRepository의 구현체인 SimpleJpaRepository에서 확인할 수 있다.  
기본적인 조회, count, 저장 메서드가 JPA의 entityManager를 이용하여 구현되어 있다.

```java
@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> ... {
    ...
}
```

먼저 레포지토리 클래스에 @Repository가 붙어있는 것을 확인할 수 있다.  
이로 인해 해당 클래스가 컴포넌트 스캔의 대상이 되어 빈으로 등록된다.  
또한 jpa 등 세부 구현체의 예외를 스프링 예외로 변환해서 발생시킨다.

레포지토리 클래스에는 @Transactional(readOnly = true)도 함께 붙어있다.  
이로 인해 스프링 데이터 JPA 레포지토리의 모든 메서드에는 기본적으로 트랜잭션이 적용된다.  
따라서 메서드를 호출하는 서비스 계층에서 트랜잭션이 적용되어 있다면 이어 받고, 적용되지 않았다면 자체적으로 트랜잭션을 돌린다.  
전역적으로는 readOnly로 트랜잭션이 적용되기 때문에, save 처럼 데이터 수정이 필요한 메서드에는 readOnly가 제외된 트랜잭션을 적용한다.  
JPA의 데이터 수정(등록, 수정, 삭제)은 트랜잭션 안에서만 가능하다.  
위와 같은 기본 설정 덕분에 특별히 직접 트랜잭션을 설정하지 않아도 정상적으로 데이터 수정이 될 수 있다.

참고로, readOnly로 Transactional을 설정하면 set autocommit false를 통해 트랜잭션을 시작하는 것은 동일하다.  
다만 영속성 컨텍스트를 flush하지 않기 때문에 변경 감지 등이 발생하지 않아서 데이터 수정이 이루어지지 않게 최적화된다.

또한 save 메서드의 구현에 대해서 자세히 살펴볼 필요가 있다.  
SimpleJpaRepository에서는 새로운 엔티티이면 em.persist를 통해 영속화하고, 기존에 존재하는 엔티티이면 merge하도록 구현되어 있다.  

```java
@Repository
@Transactional(readOnly = true)
public class SimpleJpaRepository<T, ID> ... {
    @Transactional
    public <S extends T> S save(S entity) {
        if (entityInformation.isNew(entity)) {
            em.persist(entity);
            return entity;
        } else {
            return em.merge(entity);
        }
    }
}
```

이 때 merge를 호출하면 해당하는 데이터를 db에서 조회한 후, 넘겨받은 엔티티 정보로 덮어씌워서 db에 저장하는 식으로 동작한다.  
이로 인해 조회 쿼리가 한 번 추가 되기 때문에 성능에 악영향을 준다.  
따라서 데이터 수정이 필요할 경우 save를 호출하는게 아니라, jpa의 변경 감지를 사용하는 것이 적절하다.

### 새로운 엔티티를 구별하는 방법

스프링 데이터 JPA의 레포지토리에서는 전달 받은 엔티티가 새로운 엔티티인지를 구분해야 한다.  
이 때 SimpleJpaRepository의 isNew 메서드에서는 식별자 값을 기준으로 새로운 엔티티인지를 판단한다.  
식별자가 객체 타입(Long)일 경우 null인지를 체크하고, 원시값 타입(long)이면 0인지를 체크한다.

보통 식별자에는 @GeneratedValue를 붙이는데, 이 경우 em.persist를 통해 영속회하는 시점에 식별자 값이 들어간다.  
따라서 아직 영속화 되지 않은 엔티티의 경우 식별자에 null 또는 0이 들어가고, 이 값을 통해 새로운 엔티티인지를 구분할 수 있다.

다만 식별자에 @GeneratedValue를 붙이지 않고 직접 값을 넣어주는 경우에는 문제가 있다.  
이 때에는 영속화하기 전부터 식별자에 값이 들어있기 때문에, 이미 존재하는 엔티티로 판별되고 merge가 발생한다.  
이러한 경우에는 엔티티에서 Persistable을 구현해야 한다.  
Persistable은 getId, isNew 메서드를 정의하고 있는 인터페이스이다.

```java
package org.springframework.data.domain;

public interface Persistable<ID> {
    ID getId();
    boolean isNew();
}
```

isNew 메서드에 새로운 엔티티인지를 판단하는 로직을 구현하면 된다.  
이 때 @CreatedDate를 붙여서 생성일을 필드로 가지고 있으면 영속화 시점에 생성일이 삽입된다.  
실무에서는 대부분의 엔티티에 생성일 필드가 포함되기 때문에, 보통 생성일을 기준으로 새로운 엔티티인지를 판단한다.

```java
package study.datajpa.entity;

@Entity
@EntityListeners(AuditingEntityListener.class)
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Item implements Persistable<String> {

    @Id
    private String id;

    @CreatedDate
    private LocalDateTime createdDate;

    public Item(String id) {
        this.id = id;
    }

    @Override
    public String getId() {
        return id;
    }

    @Override
    public boolean isNew() {
        return createdDate == null;
    }
}
```
