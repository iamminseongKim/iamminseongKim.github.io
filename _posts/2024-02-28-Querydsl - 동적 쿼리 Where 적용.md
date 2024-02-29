---
title: Querydsl - 동적 쿼리 Where 적용
aliases: 
tags:
  - queryDSL
  - jpa
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-02-28
last_modified_at: 2024-02-28
---
> 인프런 실전! Querydsl 강의 내용 정리

- [순수 JPA 리포지토리와 Querydsl](https://iamminseongkim.github.io/jpa/Querydsl-%EC%8B%A4%EB%AC%B4-%ED%99%9C%EC%9A%A9-%EC%88%9C%EC%88%98-JPA-%EB%A6%AC%ED%8F%AC%EC%A7%80%ED%86%A0%EB%A6%AC%EC%99%80-Querydsl/)
- [동적 쿼리 Builder 적용](https://iamminseongkim.github.io/jpa/Querydsl-%EB%8F%99%EC%A0%81-%EC%BF%BC%EB%A6%AC%EC%99%80-%EC%84%B1%EB%8A%A5-%EC%B5%9C%EC%A0%81%ED%99%94-%EC%A1%B0%ED%9A%8C-Builder-%EC%82%AC%EC%9A%A9/)
- <font color="#92d050">동적 쿼리 Where 적용</font>
- 조회 API 컨트롤러 개발

--- 
## 동적 쿼리와 성능 최적화 조회 - Where절 파라미터 사용

기존이랑 크게 다를건 없고 리포지토리단 메서드만 새로 만들어 보자.

MemberJpaRepository.java
```java
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
```

자 여기서 해줘야 할 것은 `where()`안에 메서드들을 만들어 주는 것이다.

```java
usernameEq(condition.getUsername()),  
teamNameEq(condition.getTeamName()),  
ageGoe(condition.getAgeGoe()),  
ageLoe(condition.getAgeLoe())
```
이 4개 말이다.

```java
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
```
이렇게 간단하게 만들면 될 것 같다.


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
    List<MemberTeamDto> result = memberJpaRepository.search(condition);  
    //then  
    assertThat(result).extracting("username").containsExactly("member4");  
}
```

저번 BooleanBuilder 와 동일한 테스트 이다.

![](https://i.imgur.com/F6GuKcs.png){: .align-center}

테스트도 문제 없이 통과가 됐다.

`BooleanBuilder`보다 보기는 더 좋고, <br>검증 메서드들 4개를 `재활용` 할 수도 있기 때문에<br> 이쪽이 좀 더 선호 될 것 같다.