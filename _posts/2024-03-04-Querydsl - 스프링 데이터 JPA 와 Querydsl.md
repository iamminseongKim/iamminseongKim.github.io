---
title: Querydsl - 스프링 데이터 JPA 와 Querydsl
aliases: 
tags:
  - queryDSL
  - jpa
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-03-04
last_modified_at: 2024-03-04
---
> 인프런 실전! Querydsl 강의 내용 정리

## 스프링 데이터 JPA 리포지토리로 변경

기존에 만들었던 `MemberJpaRepository`를 스프링 데이터 JPA로 동작하도록 바꿔보자.

자 **인터페이스** `MemberRepository`를 하나 만들어 보자.

그리고 MemberJpaRepository에 있는 기능들을 구현해야 하는데, 사실 스프링 데이터 JPA는 웬만하면 다 있다.

[기존 순수 JPA 리포지토리](https://iamminseongkim.github.io/jpa/Querydsl-%EC%8B%A4%EB%AC%B4-%ED%99%9C%EC%9A%A9-%EC%88%9C%EC%88%98-JPA-%EB%A6%AC%ED%8F%AC%EC%A7%80%ED%86%A0%EB%A6%AC%EC%99%80-Querydsl/#%EC%88%9C%EC%88%98-jpa-%EB%A6%AC%ED%8F%AC%EC%A7%80%ED%86%A0%EB%A6%AC%EC%99%80-querydsl)

그래서 내가 따로 만들어 줘야 할 메서드는 `findByUsername(String username)` 정도인 것 같다.

```java
public interface MemberRepository extends JpaRepository<Member, Long> {  
    // select m from Member m where m.username = ?  
    List<Member> findByUsername(String name);  
}
```
끝. ㅋㅋ

이제 테스트 해보자.

```java
@Test  
public void basicTest() throws Exception {  
    //given  
    Member member = new Member("member1", 10);  
    memberRepository.save(member);  
    //when  
    Member findMember = memberRepository.findById(member.getId()).get();  
    //then  
    assertThat(findMember).isEqualTo(member);  
  
    List<Member> result1 = memberRepository.findAll();  
    assertThat(result1).containsExactly(member);  
  
    List<Member> result2 = memberRepository.findByUsername("member1");  
    assertThat(result2).containsExactly(member);  
}
```

기존에 순수 JPA 테스트 때와 같은 코드이다. 문제 없이 잘 통과 한다.

이제 중요한 건 Querydsl을 어떻게 스프링 데이터 JPA에서 사용할 것 인가 이다.

## 사용자 정의 리포지토리

**사용자 정의 리포지토리 사용법**

1. 사용자 정의 인터페이스 작성
2. 사용자 정의 인터페이스 구현
3. 스프링 데이터 리포지토리에 사용자 정의 인터페이스 **상속**

핵심 원리는 [스프링 데이터 JPA - 확장기능](https://iamminseongkim.github.io/jpa/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-%ED%99%95%EC%9E%A5-%EA%B8%B0%EB%8A%A5/) 을 참고.

![](https://i.imgur.com/5PvIRHe.png){: .align-center}

자 먼저 `MemberRepositoryCustom`인터페이스를 만들자. 나는 Querydsl로 만든 search 메서드를 다시 사용하고 싶다.

![](https://i.imgur.com/Z3FhCd9.png){: .align-center}

얘를 사용하기 위해서 인터페이스에 다음과 같이 작성한다.

```java
List<MemberTeamDto> search(MemberSearchCondition condition);
```

그 후 `MemberRepositoryImpl` 구현체를 만들자.

```java
public class MemberRepositoryImpl implements MemberRepositoryCustom {  
  
    private final JPAQueryFactory queryFactory;  
  
    public MemberRepositoryImpl(EntityManager em) {  
        this.queryFactory = new JPAQueryFactory(em);  
    }  
  
    @Override  
    public List<MemberTeamDto> search(MemberSearchCondition condition) {  
        return queryFactory  
                .select(new QMemberTeamDto(  
                        member.id.as("memberId"),  
                        member.username,  
                        member.age,  
                        team.id.as("teamId"),  
                        team.name.as("teamName")  
                ))  
                .from(member)  
                .where(  
                        usernameEq(condition.getUsername()),  
                        teamNameEq(condition.getTeamName()),  
                        ageGoe(condition.getAgeGoe()),  
                        ageLoe(condition.getAgeLoe())  
                )  
                .leftJoin(member.team, team)  
                .fetch();  
    }  
  
    private BooleanExpression usernameEq(String username) {  
        return hasText(username) ? member.username.eq(username) : null;  
    }  
  
    private BooleanExpression teamNameEq(String teamName) {  
        return hasText(teamName) ? team.name.eq(teamName) : null;  
    }  
  
    private BooleanExpression ageGoe(Integer ageGoe) {  
        return ageGoe != null ? member.age.goe(ageGoe) : null;  
    }  
  
    private BooleanExpression ageLoe(Integer ageLoe) {  
        return ageLoe != null ? member.age.loe(ageLoe) : null;  
    }  
}
```

다음과 같이 가져왔다. 이제 이걸 `MemberRepository`에 상속 시키면 그대로 사용 가능하다.

```java
public interface MemberRepository extends JpaRepository<Member, Long>, MemberRepositoryCustom {  
	 ...
}
```

이제 테스트를 해보자.

```java
@Test  
public void searchTest() throws Exception {  
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
  
    MemberSearchCondition condition = new MemberSearchCondition();  
    condition.setAgeGoe(35);  
    condition.setAgeLoe(40);  
    condition.setTeamName("teamB");  
    //when  
    List<MemberTeamDto> result = memberRepository.search(condition);  
    //then  
    assertThat(result).extracting("username").containsExactly("member4");  
}
```

![](https://i.imgur.com/fLVr1X7.png){: .align-center}

잘 작동 하고 `memberRepository.search()`가 사용 중인 걸 볼 수 있다.

>참고 : 항상 **사용자 정의 리포지토리**가 필요한 것은 아니다.  그냥 임의의 리포지토리를 만들어도 된다. <br>예를 들어 MemberQueryRepository를 인터페이스가 아닌 클래스로 만들고 <br>`스프링 빈`으로 등록해서 그냥 직접 사용해도 된다. <br>물론 이 경우 스프링 데이터 JPA와는 아무런 관계 없이 별도로 동작한다.