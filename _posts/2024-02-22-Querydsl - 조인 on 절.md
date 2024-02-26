---
title: Querydsl - 조인 on 절
aliases: 
tags:
  - jpa
  - queryDSL
categories:
  - jpa
toc: true
toc_label: "목차" 
date: 2024-02-22
last_modified_at: 2024-02-22
---
> 인프런 실전! Querydsl 강의 내용 정리

## 기본 조인

조인의 기본 문법은 첫 번째 파라미터에 조인 대상을 지정하고, 두 번째 파라미터에 별칭(alias)으로 사용할 Q 타입을 지정하면 된다.

```java
join(조인 대상, 별칭으로 사용할 Q 타입)
```

```java
/**  	
 * 팀 A에 소속된 모든 회원  
 * */  
@Test  
public void join() throws Exception {  
    //given  
    //when    
    List<Member> result = queryFactory  
            .selectFrom(member)  
            .join(member.team, team)  
            .where(team.name.eq("teamA"))  
            .fetch();  
    //then  
    assertThat(result)  
            .extracting("username")  
            .containsExactly("member1", "member2");  
}
```
다음과 같이 join 을 해주면 된다.

```sql
select
        m1_0.member_id,
        m1_0.age,
        m1_0.team_id,
        m1_0.username 
    from
        member m1_0 
    join
        team t1_0 
            on t1_0.team_id=m1_0.team_id 
    where
        t1_0.name=?
```
쿼리도 잘 나간다.

`leftJoin()` 이런 것도 다 가능하다.

![](https://i.imgur.com/yPQUFLR.png){: .align-center}

left join으로 나간 걸 볼 수 있다.

![](https://i.imgur.com/XP2xLyA.png){: .align-center}

inner, right 가능하다 ~


## 조인 - On 절

- ON절을 활용한 조인 (JPA 2.1 부터 지원)
1. 조인 대상 필터링
2. 연관관계 없는 엔티티 외부 조인 

### 1. 조인 대상 필터링

> 회원과 팀을 조인하면서, 팀 이름이 teamA인 팀만 조인, 회원은 모두 조회


```
/**  
 * JPQL : select m, t from Member m left join m.team t on t.name = 'teamA'  
 * */
@Test  
public void join_on_filtering() throws Exception {  
    List<Tuple> result = queryFactory  
            .select(member, team)  
            .from(member)  
            .leftJoin(member.team, team).on(team.name.eq("teamA"))  
            .fetch();  
  
    for (Tuple tuple : result) {  
        System.out.println("tuple = "+ tuple);  
    }  
}
```

![](https://i.imgur.com/nnLoSxj.png){: .align-center}

on절에 team 맞추고 and 로 이름까지 맞춰서 가져 왔다.

left 조인 이기 때문에 teamB애들은 team이 null로 가져왔다.

> 참고 : on 절을 활용해 조인 대상을 필터링 할 때, 외부 조인이 아니라 내부 조인 (inner join)을 사용하면,<br> where 절에서 필터링 하는 것과 기능이 동일하다.<br> 따라서 on 절을 활용한 조인 대상 필터링을 사용할 때, 내부 조인 이면<br> `익숙한 where 절로 해결하고`, 정말 `외부 조인`이 필요한 경우에만 이 기능을 사용하자.



### 2. 연관 관계가 없는 엔티티 **외부 조인**

> 회원의 이름과 팀의 이름이 같은 대상 `외부 조인`

이름바 막 조인, 먼저 팀 이름으로 멤버를 만들어 보자.

```java
/**  
 * 연관 관계가 업는 엔티티 외부 조인  
 * 회원의 이름이 팀 이름과 같은 대상 외부 조인  
 * */  
@Test  
public void join_on_no_relation() throws Exception {  
    //given  
    em.persist(new Member("teamA"));  
    em.persist(new Member("teamB"));  
    em.persist(new Member("teamC"));  
  
    //when  
  
    List<Tuple> fetch = queryFactory  
            .select(member, team)  
            .from(member)  
            .leftJoin(team).on(member.username.eq(team.name))  
            .fetch();  
    //then  
    for (Tuple tuple : fetch) {  
        System.out.println("tuple = " + tuple);  
    }  
}
```

주목 해야 할 점은
```java
.leftJoin(team).on(member.username.eq(team.name))  
```
여기서 잘 보면 
`.leftJoin(member.team, team)` 이 형태가 아니라 그냥 team만 넣었다.

멤버와 팀의 연관 관계는 모르겠고 그냥 이름만 같은 걸 가져오라는 막 조인이다.

그래서 on절에서 meber.username = team.name 으로 필터링 하고 있다.


```sql
select
        m1_0.member_id,
        m1_0.age,
        m1_0.team_id,
        m1_0.username,
        t1_0.team_id,
        t1_0.name 
    from
        member m1_0 
    left join
        team t1_0 
            on m1_0.username=t1_0.name
```

쿼리는 다음과 같이 나왔다. `on m1_0.username=t1_0.name` 이렇게 비교하게 된다.
 
결과는 

![](https://i.imgur.com/DKHPy7m.png){: .align-center}

다음과 같이 teamA씨, teamB씨만 team 데이터가 나왔고, 

나머지는 member 정보만 나온 걸 볼 수 있다.

- 하이버네이트 5.1 부터 `on`절을 사용해서 서로 관계가 없는 필드로 외부 조인하는 기능이 추가 되었다. 물론 내부 조인도 가능하다.
- 주의 ! 문법을 잘 봐야 한다. `lefJoin()` 부분에 일반 조인과 다르게 엔티티가 하나만 들어간다.
	- 일반 조인 : `leftJoin(member.team, team)`
	- on 조인 : from(member).`leftJoin(team)`.on(xxx)
