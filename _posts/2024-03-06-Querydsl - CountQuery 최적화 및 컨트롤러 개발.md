---
title: Querydsl - CountQuery 최적화 및 컨트롤러 개발
aliases: 
tags:
  - queryDSL
  - jpa
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-03-06
last_modified_at: 2024-03-06
---
> 인프런 실전! Querydsl 강의 내용 정리

## CountQuery 최적화

CountQuery를 최적화 하는 방법에 대해 알아보자.

- 스프링 데이터 라이브러리가 제공
- count 쿼리가 생략 가능한 경우 생략해서 처리
	- 페이지 시작이면서 컨텐츠 사이즈가 페이지 사이즈 보다 작을 때
	- 마지막 페이지 일 때 (offset + 컨텐츠 사이즈를 더해서 전체 사이즈 구함)

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
}    
```
어제 다음과 같은 페이징 메서드를 만들었다.

이제 스프링이 지원하는 페이징 카운트 최적화 방법을 알아보자.

먼저 카운트 쿼리에 `.fetchOne()` 을 빼고 다시 변수를 추출 한다.

```java
JPAQuery<Long> countQuery = queryFactory  
        .select(member.count())  
        .from(member)  
        .leftJoin(member.team, team)  
        .where(  
                usernameEq(condition.getUsername()),  
                teamNameEq(condition.getTeamName()),  
                ageGoe(condition.getAgeGoe()),  
                ageLoe(condition.getAgeLoe())  
        );
```

그 후 메서드의 리턴을 `PageableExecutionUtils.getPage(contents, pagable, 람다식)`

이런 형태로 리턴 해주면 된다.
그럼 전체 코드는 다음과 같다.
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
            
    JPAQuery<Long> countQuery = queryFactory  
            .select(member.count())  
            .from(member)  
            .leftJoin(member.team, team)  
            .where(  
                    usernameEq(condition.getUsername()),  
                    teamNameEq(condition.getTeamName()),  
                    ageGoe(condition.getAgeGoe()),  
                    ageLoe(condition.getAgeLoe())  
            );  
  
    return PageableExecutionUtils.getPage(contents, pageable, countQuery::fetchOne);  
    //return new PageImpl<>(contents, pageable, total);  
}    
```

이렇게 하면 만약 조건이 성립하면 `카운트 쿼리`가 안 나가게 된다.

테스트로 보자.
```java
@Test  
public void searchPageTest() throws Exception {  
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
    PageRequest pageRequest = PageRequest.of(0, 10);  
  
    //when  
    Page<MemberTeamDto> result = memberRepository.searchPageComplex(condition, pageRequest);  
    //then  
    assertThat(result.getContent()).extracting("username").containsExactly("member1", "member2", "member3", "member4");  
}
```

다음과 같이 페이지가 0부터 10까지 인데 데이터가 4건이면 

테스트를 돌려보면

![](https://i.imgur.com/C6aL0FQ.png){: .align-center}

딱 컨텐츠를 가져오는 쿼리만 나가는 걸 볼 수 있다.

그럼 조건이 안 맞으면?  (페이징 0~3, 데이터 4개) 

![](https://i.imgur.com/eXW8pGO.png){: .align-center}

이땐 아까 생략되던 카운트 쿼리가 다시 나오게 된다.


## 컨트롤러 개발

이제 지금까지 만든 걸 컨트롤러까지 만들어서 이용해 보자.

`MemberController`
```java
@RestController  
@RequiredArgsConstructor  
public class MemberController { 
 ...
	 private final MemberRepository memberRepository;

 ...
 @GetMapping("/v2/members")  
	public Page<MemberTeamDto> searchMemberV2(MemberSearchCondition condition, Pageable pageable) {  
	    return memberRepository.searchPageSimple(condition, pageable);  
	}  
	  
	@GetMapping("/v3/members")  
	public Page<MemberTeamDto> searchMemberV3(MemberSearchCondition condition, Pageable pageable) {  
	    return memberRepository.searchPageComplex(condition, pageable);  
	}
}
```


다음과 같이 컨트롤러를 만들어 줬다.

![](https://i.imgur.com/mlTgC5d.png){: .align-center}

다음과 같이 페이징 정보랑 잘 나온다. 

또 v3컨트롤러 같은 경우에는 데이터 보다 페이징 카운트가 크면 카운트 쿼리도 안나간다.

### 스프링 데이터 정렬(Sort)

스프링 데이터 JPA는 자신의 정렬(Sort)을 Querydsl의 정렬(OrderSpecifier)로 편리하게 변경하는 기능을 제공한다. 이 부분은 뒤에 스프링 데이터 JPA가 제공하는 Querydsl 기능에서 살펴 보자.

**스프링 데이터 Sort를 Querydsl의 OrderSpecifier 로 변환**
```java
JPAQuery<Member> query = queryFactory.selectFrom(member);  
  
for (Sort.Order o : pageable.getSort()) {  
    PathBuilder pathBuilder = new PathBuilder(member.getType(), member.getMetadata());  
    query.orderBy(new OrderSpecifier<Comparable>(o.isAscending() ? Order.ASC : Order.DESC,  
            pathBuilder.get(o.getProperty())));  
}  
List<Member> result = query.fetch();
```

> 참고 : 정렬(`Sort`)은 조건이 조금만 복잡해져도 `Pageable`의 `Sort`기능을 사용하기 어렵다. <br>루트 엔티티 범위를 넘어가는 동적 정렬 기능이 필요하면 스프링 데이터 페이징이 제공하는<br>`Sort`를 사용하기 보다는 파라미터를 받아서 직접 쿼리 처리를 하는 것을 권장한다.



