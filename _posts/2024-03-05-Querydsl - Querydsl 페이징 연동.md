---
title: Querydsl - Querydsl 페이징 연동
aliases: 
tags:
  - queryDSL
  - jpa
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-03-05
last_modified_at: 2024-03-05
---
> 인프런 실전! Querydsl 강의 내용 정리

- 스프링 데이터의 Page, Pageable을 활용해 보자.  [스프링 데이터 JPA - 페이징](https://iamminseongkim.github.io/jpa/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-%ED%8E%98%EC%9D%B4%EC%A7%95/)
- 전체 카운트를 한번에 조회하는 단순한 방법
- 데이터 내용과 전체 카운트를 별도로 조회하는 방법

먼저 간단한 페이지를 써 보자.
`MemberRepositoryCustom`인터페이스에 메서드를 정의하자.

```java
Page<MemberTeamDto> searchPageSimple(MemberSearchCondition condition, Pageable pageable);
```

구현 해 보자. 

`MemberRepositoryImpl`
```java
@Override  
public Page<MemberTeamDto> searchPageSimple(MemberSearchCondition condition, Pageable pageable) {  
    QueryResults<MemberTeamDto> results = queryFactory  
            .select(new QMemberTeamDto(  
                    member.id.as("memberId"),  
                    member.username,  
                    member.age,  
                    team.id.as("teamId"),  
                    team.name.as("teamName")  
            ))  
            .from(member)  
            .leftJoin(member.team, team)  
            .where(  
                    usernameEq(condition.getUsername()),  
                    teamNameEq(condition.getTeamName()),  
                    ageGoe(condition.getAgeGoe()),  
                    ageLoe(condition.getAgeLoe())  
            )  
            .offset(pageable.getOffset())  
            .limit(pageable.getPageSize())  
            .fetchResults();  
  
    List<MemberTeamDto> content = results.getResults();  
    long total = results.getTotal();  
  
    return new PageImpl<>(content, pageable, total);  
}

```

Querydsl 구문은 크게 다를 건 없고, 이전 검색 쿼리를 그대로 사용했다. 여기서 다른 건 

```java
.offset(pageable.getOffset())  
.limit(pageable.getPageSize())
```
파라미터로 받아온 pageable을 사용해서 페이징 처리를 한다는 것과 `.fetchResults();`  로 페이징 객체를 반환 받는다는 것이다. 하지만 `.fetchResults();`는 현재 

![](https://i.imgur.com/nr44ZRn.png)


`deprecated`다... 이게 카운트 쿼리가 정확히 안나오는 경우도 있기 때문에 카운트 쿼리를 따로 날리는게 요즘엔 맞는 방법 인 것 같다.


아무튼 페이징 객체를 받아서 

```java
List<MemberTeamDto> content = results.getResults();  
long total = results.getTotal();  
  
return new PageImpl<>(content, pageable, total);
```
다음과 같이 넘겨 줬다.

이제 테스트를 해보자.
```java
@Test  
public void searchPageSimpleTest() throws Exception {  
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
    PageRequest pageRequest = PageRequest.of(0, 3);  
  
    //when  
    Page<MemberTeamDto> result = memberRepository.searchPageSimple(condition, pageRequest);  
    //then  
    assertThat(result.getSize()).isEqualTo(3);  
    assertThat(result.getContent()).extracting("username").containsExactly("member1", "member2", "member3");  
}
```
테스트 자체도 기존 search 메서드 테스트 했을 때랑 비슷한데, 

```java
PageRequest pageRequest = PageRequest.of(0, 3);
```
페이징 객체를 만들어서 넘겨준다는 차이가 있다.

그래서 검증 해 보면

```sql
select
        m1_0.member_id,
        m1_0.username,
        m1_0.age,
        t1_0.team_id,
        t1_0.name 
    from
        member m1_0 
    left join
        team t1_0 
            on t1_0.team_id=m1_0.team_id 
    offset
        ? rows 
    fetch
        first ? rows only
```
`offset`과 `fetch`가 쿼리로 나가는 걸 볼 수 있다.


이제 복잡한 페이징 쿼리를 사용해 보자.

### 데이터 내용과 전체 카운트를 별도로 조회하는 방법

`searchPageComplex()`메서드 구현

```java
Page<MemberTeamDto> searchPageComplex(MemberSearchCondition condition, Pageable pageable);
```
인터페이스에 다음과 같이 정의하고

구현 해보자.

```java
@Override  
public Page<MemberTeamDto> searchPageComplex(MemberSearchCondition condition, Pageable pageable) {  
    List<MemberTeamDto> contents = queryFactory  
            .select(new QMemberTeamDto(  
                    member.id.as("memberId"),  
                    member.username,  
                    member.age,  
                    team.id.as("teamId"),  
                    team.name.as("teamName")  
            ))  
            .from(member)  
            .leftJoin(member.team, team)  
            .where(  
                    usernameEq(condition.getUsername()),  
                    teamNameEq(condition.getTeamName()),  
                    ageGoe(condition.getAgeGoe()),  
                    ageLoe(condition.getAgeLoe())  
            )  
            .offset(pageable.getOffset())  
            .limit(pageable.getPageSize())  
            .fetch();  
  
    long total = queryFactory  
            .select(member)  
            .from(member)  
            .leftJoin(member.team, team)  
            .where(  
                    usernameEq(condition.getUsername()),  
                    teamNameEq(condition.getTeamName()),  
                    ageGoe(condition.getAgeGoe()),  
                    ageLoe(condition.getAgeLoe())  
            )  
            .fetchCount();  
  
    return new PageImpl<>(contents, pageable, total);  
}
```
사실 뭐 크게 다르진 않지만, 

장점은 카운트 쿼리는 사실 실제 가져오는 쿼리보다 간단한 경우가 종종 있다.

지금 여기서도 솔직히 `team`에 대해 조인할 필요가 있는지 생각해 볼 수 있다.

하지만! `fetchCount()` 이것도 `deprecated`다 ㅋㅋ

```java
Long total = queryFactory  
        .select(member.count())  
        .from(member)  
        .leftJoin(member.team, team)  
        .where(  
                usernameEq(condition.getUsername()),  
                teamNameEq(condition.getTeamName()),  
                ageGoe(condition.getAgeGoe()),  
                ageLoe(condition.getAgeLoe())  
        )  
        .fetchOne();
```
따라서 다음과 같이 작성 하는 것이 바람직 하다.

테스트 해보자.

```java
//when  
Page<MemberTeamDto> result = memberRepository.searchPageComplex(condition, pageRequest);
```
이전 테스트랑 동일하게 세팅하고 메서드만 Complex로 바꿔줬다.

![](https://i.imgur.com/WxF8atz.png)

통과는 잘 됐고, 

또 카운트 쿼리도 따로 나간 걸 볼 수 있다.

