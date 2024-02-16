---
title: 스프링 데이터 JPA - Auditing
aliases: 
tags:
  - jpa
  - spring
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-02-16
last_modified_at: 2024-02-16
---
- 엔티티를 생성, 변경할 때 변경한 사람과 시간을 추적하고 싶다면?
	- 등록일
	- 수정일
	- 등록자
	- 수정자

### 순수 JPA 사용
우선 등록일, 수정일 적용

```java
@MappedSuperclass  
@Getter  
public class JpaBaseEntity {  
  
    @Column(updatable = false)  
    private LocalDateTime createdDate;  
    private LocalDateTime updatedDate;  
  
    @PrePersist  
    public void prePersist() {  
        LocalDateTime now = LocalDateTime.now();  
        createdDate = now;  
        updatedDate = now;  
    }  
  
    @PreUpdate  
    public void preUpdate() {  
        updatedDate = LocalDateTime.now();  
    }  
}
```

> `@MappedSuperclass`를 반드시 붙혀 줘야 한다. 그 후 엔티티에 가서 extends JpaBaseEntity 하면 된다.
 
**JPA 주요 이벤트 어노테이션**
- @PrePersist, @PostPersist
- @PreUpdate, @PostUpdate

### 스프링 데이터 JPA 사용

**설정**
`@EnableJpaAuditing` → 스프링 부트 설정 클래스에 적용해야 함
`@EntityListeners(AuditingEntityListener.class)` → 엔티티에 적용.

```java
@EnableJpaAuditing  
@SpringBootApplication  
public class DataJpaApplication {  
  
    public static void main(String[] args) {  
       SpringApplication.run(DataJpaApplication.class, args);  
    }  
  
}
```

```java
@EntityListeners(AuditingEntityListener.class)  
@MappedSuperclass  
@Getter  
public class BaseEntity {  
  
    @CreatedDate  
    @Column(updatable = false)  
    private LocalDateTime createdDate;  
    @LastModifiedDate  
    private LocalDateTime lastModifiedDate;  
}
```

여기서 등록자 수정자를 추가하고 싶다면? 
- 그런데 모든 테이블 입장에서 이 공통 엔티티가 전부 필요한 데이터는 아닐 수도 있다.
- 시간은 웬만하면 필요 하고, 등록자 수정자는 필요 없을 수도 있다.
- 그럼 등록자 수정자 엔티티를 만들고, 시간 엔티티를 상속 받으면 될 거 같다.


```java
@EntityListeners(AuditingEntityListener.class)  
@MappedSuperclass  
@Getter  
public class BaseByEntity extends BaseEntity{  
    @CreatedBy  
    @Column(updatable = false)  
    private String createdBy;  
  
    @LastModifiedBy  
    private String lastModifiedBy;  
}
```

이렇게 만들었다. 이제 등록자, 수정자가 필요한 엔티티들은 BaseByEntity를 받으면 되고, 
필요 없는 곳은 BaseEntity를 상속 받으면 될  것이다.

그런데 createdBy, lastModifiedBy를 무슨 수로 값을 할당 해 줄 것인가?
 
스프링에 `Bean`을 등록해 줘야 한다.

```java
@EnableJpaAuditing  
@SpringBootApplication  
public class DataJpaApplication {  
  
    public static void main(String[] args) {  
       SpringApplication.run(DataJpaApplication.class, args);  
    }  
  
    @Bean  
    public AuditorAware<String> auditorProvider() {  
       return () -> Optional.of(UUID.randomUUID().toString());  
    }  
  
}
```

메인 클래스에 `AuditorAware`를 리턴 하는 `auditorProvider` 메서드를 만들고 `@Bean`에 등록해줬다.
값은 그냥 랜덤 아이디 뱉도록 해 놨는데, 실제 사용할 땐 session에서 아이디를 가져 오는 등 
동작이 들어가게 될 것이다.

이제 세팅이 끝났고 

![](https://i.imgur.com/Bz8xGoG.png)

다음과 같이 상속 받았으면 테스트를 진행 해보자.

```java
@Test  
public void jpaEventBaseEntity() throws Exception {  
    //given  
    Member member = new Member("member1");  
    memberRepository.save(member); // @PrePersist  
    Thread.sleep(100);  
    member.setUsername("member2");  
  
    em.flush(); // @PreUpdate  
    em.clear();  
    //when  
    Member findMember = memberRepository.findById(member.getId()).get();  
    //then  
    System.out.println("findMember.createdDate = " + findMember.getCreatedDate());  
    System.out.println("findMember.updatedDate = " + findMember.getLastModifiedDate());  
    System.out.println("findMember.createdBy = " + findMember.getCreatedBy());  
    System.out.println("findMember.updatedBy = " + findMember.getLastModifiedBy());  
}
```

다음과 같이 찍어 봤고, 결과는 

![](https://i.imgur.com/38GztFG.png)
다음과 같이 잘 나왔다.

![](https://i.imgur.com/VaQ0QQo.png)

DB에서도 같은 값을 확인 할 수 있다.


