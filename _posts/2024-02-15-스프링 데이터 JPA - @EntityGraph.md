---
title: 스프링 데이터 JPA - @EntityGraph
aliases: 
tags:
  - jpa
  - spring
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-02-15
last_modified_at: 2024-02-15
---
연관된 엔티티들을 SQL 한번에 조회하는 방법.

member → team은 지연 로딩 관계이다. 따라서 다음과 같이 team의 데이터를 조회할 때 마다 쿼리가 실행된다. **(N + 1 문제 발생)**


```java
@Test  
public void findMemberLazy() throws Exception {  
    //given  
    Team teamA = new Team("teamA");  
    Team teamB = new Team("teamB");  
    teamRepository.save(teamA);  
    teamRepository.save(teamB);  
  
    Member member1 = new Member("member1", 10, teamA);  
    Member member2 = new Member("member2", 10, teamB);  
    memberRepository.save(member1);  
    memberRepository.save(member2);  
    em.flush();  
    em.clear();  
    //when  
    List<Member> members = memberRepository.findAll();  
  
    //then  
    for (Member member : members) {  
        System.out.println("member = " + member);  
    }  
}
```
자 여기서 
member에 결과를 보면 

![](https://i.imgur.com/htGPOlP.png)

쿼리 1개 결과 2 개가 잘 나오게 됐다.

여기서 Member에 연관된 team의 이름을 알고 싶다면?
```java
System.out.println("member.team = " + member.getTeam().getName());
```
member → team은 지연 로딩 관계, 그럼 그전까지 프록시 객체였던 팀이 조회 됐기 때문에 team을 찾는 쿼리가 또 나갈 것이다.

![](https://i.imgur.com/t3f437Q.png)

이렇게 **N + 1**이 발생 되었다.

순수 JPA에선 `fetch join` 을 사용했었다.

```sql
select m from Member m left join fetch m.team
```
이런 식으로 

![](https://i.imgur.com/lH7c6L7.png)

쿼리도 1개만 나간 것을 볼 수 있다.

스프링 데이터 JPA에서는 어떻게 해결할까? 그때 사용하는 것이 `@EntityGraph`이다.

일단 findAll()함수를 오버라이딩 해서 사용하겠다.

```java
@Override  
@EntityGraph(attributePaths = {"team"})  
List<Member> findAll();
```

그 후 `@EntityGraph(attributePaths = {"team"})` 를 추가하면 fetch join한 효과를 받을 수 있다.

![](https://i.imgur.com/KqkGxPf.png)

다음과 같이 결과도 잘 나왔다.


```java
@EntityGraph(attributePaths = {"team"})  
List<Member> findEntityGraphByUsername(@Param("username") String name);
```
다음과 같이 NamedQuery에서도 얼마든지 사용 가능하다.

![](https://i.imgur.com/TRgtfta.png)

내가 생각한 것처럼 join, where 문이 잘 나갔다.

> NamedEntityGraph라는 기능도 있는데, [여기를 한번 참고 해보자..](https://docs.spring.io/spring-data/jpa/docs/current-SNAPSHOT/reference/html/#jpa.entity-graph) 
> 복잡하면 JPQL에 fetch join을 쓰자.



