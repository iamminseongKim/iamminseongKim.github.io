---
title: 스프링 데이터 JPA - Specifications (명세)
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
> 나머지 기능들. 거의 안쓰는 기능 (알고만 있자.)
- **<font color="#92d050">Specifications</font>**
- [Query By Example](https://iamminseongkim.github.io/jpa/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-Query-By-Example/)
- [Projection](https://iamminseongkim.github.io/jpa/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA-Projections/)
- [네이티브 쿼리]()

## Specifications (명세)

책 도메인 주도 설계 (Domain Driven Design)는 Specifications(명세) 라는 개념을 소개
스프링 데이터 JPA는 *JPA Criteria*를 활용해서 이 개념을 사용할 수 있도록 지원

**술어(predicate)**
- 참 또는 거짓으로 평가
- AND OR 같은 연산자로 조합해서 다양한 검색조건을 쉽게 생성(컴포지트 패턴)
- 예) 검색 조건 하나하나
- 스프링 데이터 JPA는 `org.springframework.data.jpa.domain.Specification`클래스로 정의

**명세 기능 사용 방법**
`JpaSepcificationExecutor`인터페이스 상속
```java
public interface MemberRepository extends JpaRepository<Member, Long>, MemberRepositoryCustom  
        , JpaSpecificationExecutor<Member> { 
        
        ...
  }
```


```java
@Test  
public void specBasic() throws Exception {  
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
    Specification<Member> spec = MemberSpec.username("m1").and(MemberSpec.teamName("teamA"));  
    List<Member> members = memberRepository.findAll(spec);  
    //then  
    Assertions.assertThat(members.size()).isEqualTo(1);  
}
```
다음과 같은 테스트 코드를 작성 할 때 `Specification`을 이용해서 다음과 같이 조건문을 사용하도록 해보자. 그러면  Specification을 구현해 보러 가자.

```java
public class MemberSpec {  
    public static Specification<Member> teamName(final String teamName) {  
        return new Specification<Member>() {  
            @Override  
            public Predicate toPredicate(Root<Member> root, CriteriaQuery<?> query, CriteriaBuilder criteriaBuilder) {  
  
                if (StringUtils.isEmpty(teamName)) {  
                    return null;  
                }  
  
                Join<Member, Team> t = root.join("team", JoinType.INNER);// 회원과 조인  
                return criteriaBuilder.equal(t.get("name"), teamName);  
            }  
        };  
    }  
  
    public static Specification<Member> username(final String username) {  
        return (Specification<Member>) (root, query, builder) -> {  
            return builder.equal(root.get("username"), username);  
        };  
    }  
}
```
진짜 너무 어렵다.. `criteria`를 이용하여 만들었다.

![](https://i.imgur.com/N5cPa7b.png)

결과는 잘 나왔다.

- `Specification`을 구현하면 명세들을 조립할 수 있음. `where()`, `and()`, `or()`, `not()`제공
- `findAll()`을 보면 회원 이름 명세 (`username`)와 팀 이름 명세 (`teamName`)를 `and()`로 조합해서 검색 조건 으로 사용.

- 명세를 정의하려면 `Specification`인터페이스를 구현
- 명세를 정의할 땐 `toPredicate(...)`메서드만 구현하면 되는데 JPA Criteria의 `Root`, `CriteriaQuery`, `CriteriaBuilder`클래스를 파라미터 제공
- 예제에선 편의상 람다 사용

> **참고 : 실무에서 JPA Critreria 안쓴다! QueryDSL 사용하자.** 