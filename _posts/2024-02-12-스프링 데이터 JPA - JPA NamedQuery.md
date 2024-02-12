---
title: 스프링 데이터 JPA - JPA NamedQuery
aliases: 
tags:
  - jpa
  - spring
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-02-12
last_modified_at: 2024-02-12
---
JPA의 NamedQuery를 호출 할 수 있다. (**잘 안쓴다.**)

먼저 순수 JPA NamedQuery는 다음과 같이 사용했다.
```java
@Entity  
@Getter @Setter  
@NoArgsConstructor(access = PROTECTED)  
@ToString(of = {"id", "username", "age"})  
@NamedQuery(  
        name="Member.findByUsername",  
        query = "select m from Member m where m.username = :username"  
)  
public class Member { 
	...
}
```
다음과 같이 엔티티에 직접 쿼리 이름과 내용을 정의했다. `@NamedQuery` 어노테이션을 이용하여.

그다음에 리포지토리 단에서
```java
public List<Member> findByUsername(String username) {  
    return em.createNamedQuery("Member.findByUsername", Member.class)  
            .setParameter("username", username)  
            .getResultList();  
}
```
다음과 같이 `em.createNamedQuery`를 사용하여 사용하면 된다.

이제 스프링 데이터 JPA 에선 엔티티나 XML 선언하는 건 똫같지만, 리포지토리단에서 구현하는건 @Query어노테이션을 이용한다.

```java
@Query(name = "Member.findByUsername")  
List<Member> findByUsername(@Param("username") String username);
```

다음과 같이 `@Query`어노테이션에 엔티티에서 작성한 NamedQuery이름을 적어주고 중요한건
NamedQuery에서 username파라미터를 정의해놨기 때문에 `@Param`어노테이션으로 지정해주어야 한다.
그래서 최종 코드가 저렇게 나오게 된다. 

아무튼 NamedQuery는 실무에서 잘 사용 안한다고 한다. 그냥 리포지토리단에 Query를 직접 작성하는 방법이...

> NamedQuery의 장점은 애플리케이션 로딩 시점에 오류가 있으면 에러가 발생 한다.
> 정적인 쿼리이기 때문에 애플리케이션 로딩 시점에 미리 다 파싱을 한다. 그래서 문법적으로 오류가 있다면 
> 오류가 발생하게 된다.





