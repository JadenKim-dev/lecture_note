### 스프링 데이터 JPA 구현체 분석

스프링 데이터 JPA에서 사용하는 내부 구현체를 뜯어보면서 메커니즘을 확인해보자.
공통 인터페이스 기능의 경우 JpaRepository의 구현체인 SimpleJpaRepository에서 확인할 수 있다.  
기본적인 조회, count, 저장 메서드가 JPA의 entityManager를 이용하여 구현되어 있다.

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
    ...
}
```

먼저 레포지토리 클래스에 @Repository가 붙어있는 것을 확인할 수 있다.  
이로 인해 해당 클래스가 컴포넌트 스캔의 대상이 되어 빈으로 등록된다.  
또한 jpa 등 세부 구현체의 예외를 스프링 예외로 변환해서 발생시킨다.

### 새로운 엔티티를 구별하는 방법

```java
package org.springframework.data.domain;

public interface Persistable<ID> {
    ID getId();
    boolean isNew();
}
```

```java
package study.datajpa.entity;

import lombok.AccessLevel;
import lombok.NoArgsConstructor;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.domain.Persistable;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import javax.persistence.Entity;
import javax.persistence.EntityListeners;
import javax.persistence.Id;
import java.time.LocalDateTime;

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
