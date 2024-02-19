---
title: 스프링 데이터 JPA - JPA Hint and  Lock
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
### JPA Hint 
JPA 쿼리 힌트 (SQL 힌트가 아니라 JPA 구현체에게 제공하는 힌트)

**쿼리 힌트 사용**
```java
@QueryHints(value = @QueryHint(name="org.hibernate.readOnly", value = "true"))  
Member findReadOnlyByUsername(String username);
```

다음과 같이 사용하면 여기서 생성된 엔티티는 readOnly가 된다.

```java
@Test  
public void queryHint() throws Exception {  
    //given  
    Member member1 = memberRepository.save(new Member("member1", 10));  
    em.flush(); // 인서트  
    em.clear(); // 영속성 컨텍스트 날림  
    //when  
    Member findMember = memberRepository.findById(member1.getId()).get();  
    findMember.setUsername("member2");  
    //then  
  
    em.flush(); // 업데이트 (변경감지)  
}
```
이런 테스트가 있다고 보면 쿼리가 인서트 제외하고, 2번 나가게 될 것이다.  `findById()`에서 select

setUsername 때문에 `em.flush()` 에서 업데이트 

![](https://i.imgur.com/Z7rleZ8.png)

만약에 가져온 엔티티를 업데이트를 막기 위해선 JPA 쿼리 힌트를 사용하면 된다.


```java
@Test  
public void queryHint2() throws Exception {  
    //given  
    Member member1 = memberRepository.save(new Member("member1", 10));  
    em.flush(); // 인서트  
    em.clear(); // 영속성 컨텍스트 날림  
    //when  
    Member findMember = memberRepository.findReadOnlyByUsername("member1");  
    findMember.setUsername("member2");  
    //then  
  
    em.flush(); // 업데이트 (변경감지)  
}
```
메서드만 `findReadOnlyByUsername`로 바꿔서 동일하게 테스트를 진행해보면

![](https://i.imgur.com/1iKghVH.png)

바뀌지 않고 끝나버린 것을 볼 수 있다.

### JPA Lock


```java
@Lock(LockModeType.PESSIMISTIC_WRITE)  
List<Member> findLockByUsername(String username);
```
이런 식으로 스프링 데이터 JPA에서는 JPA에 Lock 기능을 다음과 같이 사용할 수 있다.

잘 사용하지 않아서, 사용하려면 제대로 찾아보고 사용하자.

아무튼 
![](https://i.imgur.com/npdxJMQ.png)

select 절에 for update 이런 식으로 나간다. 이러면 그 세션 에서 업데이트 하지 않는 이상 다른 곳에서 
그  특정 데이터를 못 건드리게 된다.


