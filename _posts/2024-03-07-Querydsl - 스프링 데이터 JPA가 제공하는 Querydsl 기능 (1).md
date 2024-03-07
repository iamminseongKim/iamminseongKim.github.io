---
title: Querydsl - 스프링 데이터 JPA가 제공하는 Querydsl 기능 (1)
aliases: 
tags:
  - queryDSL
  - jpa
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-03-07
last_modified_at: 2024-03-07
---
> 인프런 실전! Querydsl 강의 내용 정리

여기서 소개하는 기능은 제약이 커서 복잡한 실무 환경에서 사용하기에는 많이 부족하다.


## 인터페이스 지원 - QuerydslPredicateExecutor

- [스프링 JPA 공식 문서](https://docs.spring.io/spring-data/jpa/docs/2.2.3.RELEASE/reference/html/#core.extensions.querydsl)

### 적용 법

먼저 `Repository` 인터페이스에 `QuerydslPredicateExecutor<Entity>` 를 상속 시켜야 한다.

```java
public interface MemberRepository extends JpaRepository<Member, Long>, MemberRepositoryCustom, QuerydslPredicateExecutor<Member> {  
  
    // select m from Member m where m.username = ?  
    List<Member> findByUsername(String name);  
}
```

그 후 이제 기능들을 사용하면 되는데 

바로 테스트 코드를 보자.

```java
@Test  
public void querydslPredicateExecutorTest() throws Exception {  
  
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
  
  
    QMember member = QMember.member;  
    Iterable<Member> result = memberRepository.findAll(  
            member.age.between(10, 40)  
                    .and(member.username.eq("member1")));  
  
    for (Member findMember : result) {  
        System.out.println("findMember = " + findMember);  
    }  
}
```

신기하게도 `QMember.member` 를 사용해서 `findAll()`에다가 넣어 버렸다. 심지어 조건도 같이 넣는다.

그러면 결과는

![](https://i.imgur.com/bL4tEsv.png){: .align-center}

내가 원하는 쿼리가 나가는 걸 볼 수 있다. between이랑 username.


### 한계 점

- 조인 X (묵시적 조인은 가능하지만 `left join`이 불가능 하다.)
- 클라이언트가 Querydsl에 의존해야 한다. 서비스 클래스가 Querydsl이라는 구현 기술에 의존해야 한다.
- 복잡한 실무 환경에서 사용하기에는 한계가 명확하다.

> 참고 : `QuerydslPredicateExecutor`는 Pageable, Sort를 모두 지원하고 정상 동작한다.


## Querydsl Web 지원

- [스프링 JPA 공식 문서](https://docs.spring.io/spring-data/jpa/docs/2.2.3.RELEASE/reference/html/#core.web.type-safe)

```java
@Controller  
class UserController {  
  
    @Autowired UserRepository repository;  
  
    @RequestMapping(value = "/", method = RequestMethod.GET)  
    String index(Model model, @QuerydslPredicate(root = User.class) Predicate predicate,  
                 Pageable pageable, @RequestParam MultiValueMap<String, String> parameters) {  
  
        model.addAttribute("users", repository.findAll(predicate, pageable));  
  
        return "index";  
    }  
}
```

아무튼 컨트롤러 단에서 `findAll()`을 사용해서 바로 url 로 들어온 값을 바인딩 해주는 기능.

### 한계점

- 진짜 단순한 조건만 가능
- 조건을 커스텀하는 기능이 복잡하고 명시적이지 않음
- 컨트롤러가 Querydsl에 의존
- 복잡한 실무 환경에서 사용하기에는 한계가 명확
