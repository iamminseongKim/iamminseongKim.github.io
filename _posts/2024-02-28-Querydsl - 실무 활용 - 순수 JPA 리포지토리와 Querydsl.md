---
title: Querydsl - 실무 활용 - 순수 JPA 리포지토리와 Querydsl
aliases: 
tags:
  - queryDSL
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-02-28
last_modified_at: 2024-02-28
---
> 인프런 실전! Querydsl 강의 내용 정리

- <font color="#92d050">순수 JPA 리포지토리와 Querydsl</font>
- [동적 쿼리 Builder 적용](https://iamminseongkim.github.io/jpa/Querydsl-%EB%8F%99%EC%A0%81-%EC%BF%BC%EB%A6%AC%EC%99%80-%EC%84%B1%EB%8A%A5-%EC%B5%9C%EC%A0%81%ED%99%94-%EC%A1%B0%ED%9A%8C-Builder-%EC%82%AC%EC%9A%A9/)
- [동적 쿼리 Where 적용](https://iamminseongkim.github.io/jpa/Querydsl-%EB%8F%99%EC%A0%81-%EC%BF%BC%EB%A6%AC-Where-%EC%A0%81%EC%9A%A9/)
- 조회 API 컨트롤러 개발

--- 
## 순수 JPA 리포지토리와 Querydsl

### 순수 JPA 리포지토리

![](https://i.imgur.com/bFCD8jl.png){: .align-center}

다음과 같이 패키지와 파일을 만들어 줬다.

그리고 기본적인 `save`, `findById`, `findAll`, `findByUsername` 메서드를 만들어 보자.

```java
@Repository  
public class MemberJpaRepository {  
  
    private final EntityManager em;  
    private final JPAQueryFactory queryFactory;  
      
    public MemberJpaRepository(EntityManager em) {  
        this.em = em;  
        this.queryFactory = new JPAQueryFactory(em);  
    }   
    
    public void save(Member member) {  
        em.persist(member);  
    }
        
    public Optional<Member> findById(Long id) {  
        Member findMember = em.find(Member.class, id);  
        return Optional.ofNullable(findMember);  
    }
        
    public List<Member> findAll() {  
        return em.createQuery("select m from Member m", Member.class)  
                .getResultList();  
    }
      
    public List<Member> findByUsername(String username) {  
        return em.createQuery("select m from Member m where m.username = :username", Member.class)  
                .setParameter("username", username)  
                .getResultList();  
    }  
}
```

평범하게 JPQL로 만든 순수 JPA 리포지토리다. <br>queryDsl 도 여기서 사용할 거기 때문에 

```java
private final EntityManager em;  
private final JPAQueryFactory queryFactory;  
  
public MemberJpaRepository(EntityManager em) {  
	this.em = em;  
	this.queryFactory = new JPAQueryFactory(em);  
}  
```

다음과 같이 **생성자**를 만들어 줬다.

참고로 QueryDsl 은 Bean으로 등록 해서 사용해도 된다. 

그럼 Main 클래스에서 

```java
@Bean  
JPAQueryFactory jpaQueryFactory(EntityManager em) {  
    return new JPAQueryFactory(em);  
}
```
다음과 같이 등록하고

```java
private final EntityManager em;  
private final JPAQueryFactory queryFactory;  
  
public MemberJpaRepository(EntityManager em, JPAQueryFactory queryFactory) {  
    this.em = em;  
    this.queryFactory = queryFactory;  
}
```

다음과 같이 사용하면 된다. 동시성 문제 없다.

또 롬복 사용해서 `@NoArgsConstructor` 사용할 수도 있다.

그런데 단점이 의존성을 2개 주입해야 하기 때문에 테스트 코드가 조금 더 복잡해 지긴 할 것 같다.

난 처음 방식을 쓰겠다 일단.


이제 만든 걸 테스트 해 보자.

```java
@Test  
public void basicTest() throws Exception {  
    //given  
    Member member = new Member("member1", 10);  
    memberJpaRepository.save(member);  
    //when  
    Member findMember = memberJpaRepository.findById(member.getId()).get();  
    //then  
    assertThat(findMember).isEqualTo(member);  
  
    List<Member> result1 = memberJpaRepository.findAll();  
    assertThat(result1).containsExactly(member);  
  
    List<Member> result2 = memberJpaRepository.findByUsername("member1");  
    assertThat(result2).containsExactly(member);  
}
```

뭐 간단한 테스트 이다.

![](https://i.imgur.com/HiNCYYw.png){: .align-center}

통과도 잘 됬다.

이제 QueryDsl을 사용해서 `findAll`, `findByUsername` 메서드를 다시 만들어 보자.

```java
public List<Member> findAll_Querydsl() {  
    return queryFactory  
            .selectFrom(member)  
            .fetch();  
}

public List<Member> findByUsername_Querydsl(String username) {  
    return queryFactory  
            .selectFrom(member)  
            .where(member.username.eq(username))  
            .fetch();  
}

```

매우 깔끔해 졌다.

또 장점은 이거다.

![](https://i.imgur.com/tVTa1nD.gif){: .align-center}

딸깍 메타가 가능하다...

그에 반해 원래 순수 JPA는 어떤지 보자.

![](https://i.imgur.com/nLXWLSs.gif)

저기서 오타 나도 모른다.. ㅋㅋ

테스트 해보자.

```java
@Test  
public void basicQuerydslTest() throws Exception {  
    Member member = new Member("member1", 10);  
    memberJpaRepository.save(member);  
  
    List<Member> result1 = memberJpaRepository.findAll_Querydsl();  
    assertThat(result1).containsExactly(member);  
  
    List<Member> result2 = memberJpaRepository.findByUsername_Querydsl("member1");  
    assertThat(result2).containsExactly(member);  
  
}
```
간단하게 member를 가지고 있나 체크해 봤다.


![](https://i.imgur.com/X94dXEq.png){: .align-center}

테스트도 잘 통과가 된 걸 볼 수 있다.

아무튼 QueryDsl이 초기 세팅만 좀 많이 어렵지, 세팅만 되면... 진짜 편리하다.



