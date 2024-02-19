---
title: 스프링 데이터 JPA - Query By Example
aliases: 
tags:
  - jpa
  - spring
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-02-19
last_modified_at: 2024-02-19
---
### 나머지 기능
> 거의 안쓰는 기능 (알고만 있자.)
- [Specifications](https://iamminseongkim.github.io/jpa/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-Specifications-(%EB%AA%85%EC%84%B8)/)
- **<font color="#92d050">Query By Example</font>**
- [Projection](https://iamminseongkim.github.io/jpa/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-Projections/)
- [네이티브 쿼리]()

[스프링 공식 문서](https://docs.spring.io/spring-data/jpa/reference/repositories/query-by-example.html)

예제로 가보자. 


다음과 같이 테스트 코드를 작성해 보자
```java
@Test  
public void queryByExample() throws Exception {  
    //given  
    Team teamA = new Team("teamA");  
    em.persist(teamA);  
  
    Member m1 = new Member("m1", 0, teamA);  
    Member m2 = new Member("m2", 0, teamA);  
    em.persist(m1);  
    em.persist(m2);  
  
    em.flush();  
    em.clear();  

    //when  
    // Probe    
    Member member = new Member("m1");  
    ExampleMatcher matcher = ExampleMatcher.matching()  
            .withIgnorePaths("age");  
  
    Example<Member> memberExample = Example.of(member, matcher);  
    List<Member> members = memberRepository.findAll(memberExample);  
  
    //then  
    assertThat(members.get(0).getUsername()).isEqualTo("m1");  
}
```
여기서 볼건 

```java
	Member member = new Member("m1");  
    ExampleMatcher matcher = ExampleMatcher.matching()  
            .withIgnorePaths("age");  
  
    Example<Member> memberExample = Example.of(member, matcher); 
```
이 부분이다. 이건 검색할 조건을 객체로 검색하겠다는 의미이다. 
이때 나이는 검색 조건에서 빼기 위해 다음과 같이 작성했고, 
결과를 Example객체의 memberExample로 만들어서 `findAll()`에 넘겨버렸다.

JpaRepository 인터페이스에 다음과 같은 게 있다.
```java
<S extends T> List<S> findAll(Example<S> example);
```

![](https://i.imgur.com/5q17DCf.png)

결과적으로 username이 where 문에 들어간 걸 볼 수 있다.
상당히 신박해 보인다. 하지만.. join관련 문제가 있다. (inner 조인만 가능하다.)


```java
// Probe  
Member member = new Member("m1");  
Team team = new Team("teamA");  
member.setTeam(team);
```

다음과 같이 검색조건 객체를 만들 때 team도 세팅해 주면 
![](https://i.imgur.com/2tsTgYM.png)
join도 가능하긴 하다.

- Probe : 필드에 데이터가 있는 실제 도메인 객체
- ExampleMatcher : 특정 필드를 일치 시키는 상세한 정보 제공, 재사용 가능
- Example : Probe와 ExampleMatcher로 구성, 쿼리를 생성하는데 사용

**장점**
- 동적 쿼리를 편리하게 처리
- 도메인 객체를 그대로 사용
- 데이터 저장소를 RDB에서 NOSQL로 변경해도 코드 변경이 없게 추상화 되어 있음
- 스프링 데이터 JPA `JpaRepository`인터페이스에 이미 포함

**단점**
- 조인은 가능하지만, 내부 조인 (INNER JOIN)만 가능함 외부 조인(LEFT JOIN)안됨.
- 다음과 같은 중첩 제약 조건 안됨
	- `firstname = ?0 or (firstname =?1 and lastname = ?2)`
- 매칭 조건이 매우 단순함
	- 문자는 `starts/contains/ends/regex`
	- 다른 속성은 정확한 매칭(=)만 지원

**정리**
- **실무에서 사용하기에는 매칭 조건이 너무 단순하고, LEFT 조인이 안됨**
- **실무에서는 QueryDSL을 사용하자.....**
