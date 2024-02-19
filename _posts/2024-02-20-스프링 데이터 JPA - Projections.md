---
title: 스프링 데이터 JPA - Projections
aliases: 
tags:
  - jpa
  - spring
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-02-20
last_modified_at: 2024-02-20
---
### 나머지 기능
> 거의 안쓰는 기능 (알고만 있자.)
- [Specifications](https://iamminseongkim.github.io/jpa/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-Specifications-(%EB%AA%85%EC%84%B8)/)
- [Query By Example](https://iamminseongkim.github.io/jpa/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-Query-By-Example/)
- <font color="#92d050">Projection</font>
- [네이티브 쿼리]()

엔티티 대신에 DTO를 편리하게 조회할 때 사용
전체 엔티티가 아니라 만약 회원 이름만 딱 조회하고 싶으면?

먼저 repository패키지에 `UsernameOnly` 인터페이스를 만들었다.
```java
public interface UsernameOnly {  
    String getUsername();  
}
```
내가 필요한 username을 getter 만드는 느낌으로 이름을 `getUsername()`으로 한다. 만약에 Id 가 필요했다면 `Long getId();` 이런 걸 추가 하면 되지 않을까?


그 후 `MemberRepository`에서 다음과 같이 사용 했다.
```java
List<UsernameOnly> findProjectionsByUsername(@Param("username") String username);
```

이러고 테스트를 진행해 보자.

```java
@Test  
public void projections() throws Exception {  
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
    List<UsernameOnly> result = memberRepository.findProjectionsByUsername("m1");  
    //then  
    assertThat(result.get(0).getUsername()).isEqualTo("m1");  
}
```

![](https://i.imgur.com/AijKzxr.png)

정말 재밌게도 엔티티를 조회할 땐 엔티티에 속성들이 모두 select 절에 들어갔지만, 지금은 `UsernameOnly`인터페이스에 정의된 `getUsername` 만 select 절에 가져오는 걸 볼 수 있다. 

또 스프링이 알아서 인터페이스로 만든 `UsernameOnly`을 알아서 객체로 구현해서 리턴 해주는 것도 알 수 있다.

### OpenProjection

```java
public interface UsernameOnly {  
    @Value("#{target.username + ' ' + target.age}")  
    String getUsername();  
}
```

이런 것도 가능하다. userName에 원하는 데이터를 만들 수도 있다.

하지만 왜 `OpenProjection`이냐면, 이게 실행 될 때 Entity를 조회하듯이 모든 걸 다 조회한 후에  
`@Value("#{target.username + ' ' + target.age}")` 여기서 정의한 대로 만들어 주기 때문이다.

이렇게 바꿔 놓고 테스트를 또 진행해보면

![](https://i.imgur.com/seukmI9.png)

이름 + 나이 가 이름으로 출력 된걸 볼 수 있다.

### 클래스 형식 Projections

다음과 같은 형태도 가능하다.

`UsernameOnlyDto` 클래스를 만들어 보자.

```java
public class UsernameOnlyDto {  
    private final String username;  
  
    public UsernameOnlyDto(String username) {  
        this.username = username;  
    }  
  
    public String getUsername() {  
        return username;  
    }  
}
```
다음과 같은 클래스를 만들었고,이때 중요한 건 생성자에 들어가는 파라미터 이름이다. 
이게 우리가 조회하려는 **엔티티**의 이름과 **일치해야** 조회가 잘 이루어 진다.

그 다음 getter를 세팅해 주면 된다.

이제 `MemberRepository`에서 조회 메서드를  살짝 수정해 보자

```java
<T> List<T> findProjectionsByUsername(@Param("username") String username, Class<T> type);
```
다음과 같이 아무 타입을 받아서 넘기겠다는 의미로 작성했다.

그럼 테스트에선

```java
@Test  
public void projections() throws Exception {  
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
    List<UsernameOnlyDto> result = memberRepository.findProjectionsByUsername("m1", UsernameOnlyDto.class);  
    //then  
    assertThat(result.get(0).getUsername()).isEqualTo("m1");  
}
```
`when`절에서 호출하는 방식으로 호출 하면 원하는 클래스로 받을 수 있게 되는 것이다.

![](https://i.imgur.com/j4ZNpr2.png)

쿼리도 내가 원하는 방식으로 조회가 됐다. 
>혹시 여기서 JPA 3.0 이상이라 오류가 난다면 인텔리제이에서 gradle 설정에서 빌드를 gradle로 바꾸면 될 거다.

아무튼 이렇게 해놓으면, 동적으로 Projection을 할 수 있게 된다.

### 중첩 구조 Projections

멤버랑 팀을 함께 Projections를 해보자.


```java
public interface NestedClosedProjections {  
    String getUsername();  
    TeamInfo getTeam();  
    interface TeamInfo {  
        String getName();  
    }  
}
```
다음과 같이 인터페이스를 만들었고 그 안에 Team에 대한 인터페이스를 만들었다.

일단 동적으로 `MemberRepository`에 `findProjectionsByUsername()` 메서드를 만들어 놨기 때문에 
테스트 코드만 살짝 수정해 보자.

```java
@Test  
public void projections() throws Exception {  
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
    List<NestedClosedProjections> result = memberRepository.findProjectionsByUsername("m1", NestedClosedProjections.class);  
    //then  
    assertThat(result.get(0).getUsername()).isEqualTo("m1");  
    assertThat(result.get(0).getTeam().getName()).isEqualTo("teamA");  
}
```

자 돌려보자.

![](https://i.imgur.com/bNhId3j.png)

통과는 잘 됐는데, Team에 대한 컬럼들은 최적화가 안된다.

**주의**
- 프로젝션 대상이 ROOT 엔티티면, JPQL SELECT절 최적화 가능
- 프로젝션 대상이 ROOT 가 아니라면
	- LEFT OUTER JOIN 처리
	- 모든 필드를 SELECT해서 엔티티로 조회한 다음 계산

**정리**
- **프로젝션 대상이 root 엔티티면 유용하다.**
- **프로젝션 대상이 root 엔티티를 넘어가면 JPQL SELECT 최적화가 안된다.**
- **실무의 복잡한 쿼리를 해결하기에는 한계가 있다.**
- **실무에서는 단순할 때만 사용하고, 조금만 복잡해지면 QueryDSL을 사용하자..**