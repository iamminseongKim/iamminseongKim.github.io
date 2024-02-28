---
title: Querydsl - 동적 쿼리와 성능 최적화 조회 - Builder 사용
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
- <font color="#92d050">동적 쿼리 Builder 적용</font>
- [동적 쿼리 Where 적용](https://iamminseongkim.github.io/jpa/Querydsl-%EB%8F%99%EC%A0%81-%EC%BF%BC%EB%A6%AC-Where-%EC%A0%81%EC%9A%A9/)
- 조회 API 컨트롤러 개발

--- 
## 동적 쿼리와 성능 최적화 조회 - Builder 사용

### MebmerTeamDto - 조회 최적화용 DTO 추가

```java
@Data  
public class MemberTeamDto {  
  
    private Long memberId;  
    private String username;  
    private int age;  
    private Long teamId;  
    private String teamName;  
  
    @QueryProjection  
    public MemberTeamDto(Long memberId, String username, int age, Long teamId, String teamName) {  
        this.memberId = memberId;  
        this.username = username;  
        this.age = age;  
        this.teamId = teamId;  
        this.teamName = teamName;  
    }  
}
```
다음과 같이 만들고 Qdto 만들기

![](https://i.imgur.com/GxyqrbK.png)


그 다음 검색 조건 클래스를 만들었다.

MemberSearchCondition.java
```java
@Data  
public class MemberSearchCondition {  
    // 회원명, 팀명, 나이(ageGoe, ageLoe)  
    private String username;  
    private String teamName;  
    private Integer ageGoe;  
    private Integer ageLoe;  
}
```

이제 리포지토리 메서드를 설계해 보자.

MemberJpaRepository.java
```java
public List<MemberTeamDto> searchByBuilder(MemberSearchCondition condition) {  
  
    BooleanBuilder builder = new BooleanBuilder();  
  
    if (hasText(condition.getUsername())) {  
        builder.and(member.username.eq(condition.getUsername()));  
    }  
    if (hasText(condition.getTeamName())) {  
        builder.and(team.name.eq(condition.getTeamName()));  
    }  
  
    if (condition.getAgeGoe() != null) {  
        builder.and(member.age.goe(condition.getAgeGoe()));  
    }  
  
    if (condition.getAgeLoe() != null) {  
        builder.and(member.age.loe(condition.getAgeLoe()));  
    }  
  
    return queryFactory  
            .select(new QMemberTeamDto(  
                    member.id.as("memberId"),  
                    member.username,  
                    member.age,  
                    team.id.as("teamId"),  
                    team.name.as("teamName")  
            ))  
            .from(member)  
            .where(builder)  
            .leftJoin(member.team, team)  
            .fetch();  
}
```

전에 배운 `BooleanBuilder` 를 사용해서 검색 조건을 동적으로 만들었다.
`hasText` 는 `StringUtils.hasText()` 이다.

이제 테스트 코드를 작성해 보자.


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
    List<MemberTeamDto> result = memberJpaRepository.searchByBuilder(condition);  
    //then  
    assertThat(result).extracting("username").containsExactly("member4");  
}
```

딱 35~40살에 Team B인 사람만 찾아보자.
```java
MemberSearchCondition condition = new MemberSearchCondition();  
condition.setAgeGoe(30);  
condition.setAgeLoe(40);  
condition.setTeamName("teamB");
```
아까 만든 검색 조건을 세팅한 것이다.

![](https://i.imgur.com/pLq59sZ.png)
{: .align-center}

쿼리도 내가 원하는 대로 조건 3개가 잘 나간 걸 볼 수 있다.