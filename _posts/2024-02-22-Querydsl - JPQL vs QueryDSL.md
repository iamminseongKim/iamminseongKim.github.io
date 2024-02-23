---
title: Querydsl - JPQL vs QueryDSL
aliases: 
tags:
  - jpa
  - queryDSL
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-02-22
last_modified_at: 2024-02-22
---
> 인프런 실전! Querydsl 강의 내용 정리

오늘은 JPQL과 QueryDSL 코드가 어떻게 다른지 같은 동작하는 메서드를 각각 만들어서 비교해 보자.


먼저 간단하게 테스트 코드를 작성해 보자.
`QuerydslBasicTest.java`
```java
@SpringBootTest  
@Transactional  
public class QuerydslBasicTest {  
  
    @Autowired  
    EntityManager em;  
  
    @BeforeEach  
    public void before() {  
        //given  
        Team teamA = new Team("teamA");  
        Team teamB = new Team("teamB");  
        em.persist(teamA);  
        em.persist(teamB);  
  
        Member member1 = new Member("member1", 10, teamA);  
        Member member2 = new Member("member2", 20, teamA);  
  
        Member member3 = new Member("member3", 30, teamB);  
        Member member4 = new Member("member4", 40, teamB);  
        em.persist(member1);  
        em.persist(member2);  
        em.persist(member3);  
        em.persist(member4);  
        em.flush();  
        em.clear();  
    }
}  
```
이렇게 미리 사전 작업 해 놓고 이제 JPQL vs Querydsl 을 비교해 보자.

#### 1. JPQL
```java
@Test  
public void startJPQL() throws Exception {  
    //given  
    //when    
    String qlString =  
            "select m from Member m " +  
            "where m.username = :username";  
            
    Member findMember = em.createQuery(qlString, Member.class)  
            .setParameter("username", "member1")  
            .getSingleResult();  
    //then  
    assertThat(findMember.getUsername()).isEqualTo("member1");  
}
```


#### 2. Querydsl
```java 
@Test  
public void startQuerydsl() throws Exception {  
    //given  
    JPAQueryFactory queryFactory = new JPAQueryFactory(em);  
    QMember m = new QMember("m");  
  
    //when  
    Member findMember = queryFactory  
            .select(m)  
            .from(m)  
            .where(m.username.eq("member1"))  
            .fetchOne();  
  
    //then  
    assertThat(findMember.getUsername()).isEqualTo("member1");  
}
```

두 개 코드가 어떻게 보이는가? 
내가 처음에 느낀 건 `querydsl`이 초기 세팅 (Q클래스 build) 같은 것만 넘어가면 진짜 사기 같다.

```java
.select(m)  
.from(m)  
.where(m.username.eq("member1"))  
.fetchOne();  
```
이건 그냥 SQL 이 아닌가? ㄷㄷ;

그리고 또 하나 좋게 본 건 에러 발생 시점이다. 

JPQL에서 만약에 누가 
```java
String qlString =  
            "select m from Member m" +  
            "where m.username = :username";  
```
이렇게 써 놓으면 오류를 찾을 수 있을 것 같은가?

그런데 querydsl은  무슨 자바 함수 쓰듯이 오타가 나면 바로 IDE 에서 발견 해주고, 놀랍다.

![](https://i.imgur.com/YwYGziR.png){: .align-center}

바로 저렇게 나온다.


### Querydsl 코드 간소화

이제 Querydsl 코드를 좀 더 줄여 보자.
```java
JPAQueryFactory queryFactory = new JPAQueryFactory(em);
```
이건 메서드 밖으로 던져 버리자.

![](https://i.imgur.com/nrFCFMH.png){: .align-center}


이렇게 필드로 던졌고, 멀티 쓰레드에 대해 걱정하지 않아도 된다고 한다.

> [스프링이 어떻게 EntityManager를 관리하는가?](https://doanduyhai.wordpress.com/2011/11/21/spring-persistencecontext-explained/)  더 궁금하면 이 글을 한번 참고해 보자.
> 여기서 하는 말을 요약해 보자면, 
> 
> Spring은 `@PersistenceContext`를 통해 주입 된 `EntityManager`에 대한 프록시를 제공하여 이 문제를 해결한다고 한다. 
> 
> Spring은 `EntityManager` 빈에 대한 프록시를 생성하고 해당 범위를 관리. 
> 
> 이 프록시는 `@PersistenceContext` 주석을 사용할 때 각 스레드가 `EntityManager`의 자체 인스턴스를 얻도록 보장함.


이제 더 줄여 보자.

#### 기본 Q-Type 활용 

```java
@Test  
public void startQuerydsl() throws Exception {  
    //given  
    QMember m = new QMember("m");  
    //when  
    Member findMember = queryFactory  
            .select(m)  
            .from(m)  
            .where(m.username.eq("member1"))  
            .fetchOne();  
  
    //then  
    assertThat(findMember.getUsername()).isEqualTo("member1");  
}
```
지금 코드는 다음과 같다.

![](https://i.imgur.com/s2vIgR7.png){: .align-center}

과감하게 `QMember m = new QMember("m");`를 지워버리고 `QMember.member`를 바로 사용해 버리자.

![](https://i.imgur.com/Z7FdsLP.png){: .align-center}

자 이제 `QMember`를 static 처리 해버리면 최종 결과는 다음과 같다.

```java
@Test  
public void startQuerydsl() throws Exception {  
    //when  
    Member findMember = queryFactory  
            .select(member)  
            .from(member)  
            .where(member.username.eq("member1"))  
            .fetchOne();  
    //then  
    assertThat(findMember.getUsername()).isEqualTo("member1");  
}
```

진짜 깔끔해 졌다 ㄷㄷ.

