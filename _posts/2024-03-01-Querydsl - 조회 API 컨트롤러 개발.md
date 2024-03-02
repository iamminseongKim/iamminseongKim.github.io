---
title: Querydsl - 조회 API 컨트롤러 개발
aliases: 
tags:
  - jpa
  - queryDSL
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-03-01
last_modified_at: 2024-03-01
---
> 인프런 실전! Querydsl 강의 내용 정리

- [순수 JPA 리포지토리와 Querydsl](https://iamminseongkim.github.io/jpa/Querydsl-%EC%8B%A4%EB%AC%B4-%ED%99%9C%EC%9A%A9-%EC%88%9C%EC%88%98-JPA-%EB%A6%AC%ED%8F%AC%EC%A7%80%ED%86%A0%EB%A6%AC%EC%99%80-Querydsl/)
- [동적 쿼리 Builder 적용](https://iamminseongkim.github.io/jpa/Querydsl-%EB%8F%99%EC%A0%81-%EC%BF%BC%EB%A6%AC%EC%99%80-%EC%84%B1%EB%8A%A5-%EC%B5%9C%EC%A0%81%ED%99%94-%EC%A1%B0%ED%9A%8C-Builder-%EC%82%AC%EC%9A%A9/)
- [동적 쿼리 Where 적용](https://iamminseongkim.github.io/jpa/Querydsl-%EB%8F%99%EC%A0%81-%EC%BF%BC%EB%A6%AC-Where-%EC%A0%81%EC%9A%A9/)
- <font color="#92d050">조회 API 컨트롤러 개발</font>

--- 
## 조회 API 컨트롤러 개발

편리한 데이터 확인을 위해 샘플 데이터를 추가하자.

샘플 데이터 추가가 테스트 케이스 실행에 영향을 주지 않도록 다음과 같이 `프로파일`을 설정하자.

`src/main/resources/application.yml`
```yaml
spring:  
  profiles:  
    active: local
```

이제 메인(테스트 말고) 에서 가짜 데이터를 스프링 시작 시점에 넣도록 설정해 보자.

`src/main/java/study/querydsl/controller/InitMember.java` 
```java
@Profile("local")  
@Component  
@RequiredArgsConstructor  
public class InitMember {  
  
    private final InitMemberService initMemberService;  
    @PostConstruct  
    public void init() {  
        initMemberService.init();  
    }  
  
    @Component  
    static class InitMemberService {  
        @PersistenceContext  
        EntityManager em;  
  
        @Transactional  
        public void init() {  
            Team teamA = new Team("teamA");  
            Team teamB = new Team("teamB");  
            em.persist(teamA);  
            em.persist(teamB);  
            for (int i=0; i< 100; i++) {  
                Team selectedTeam = i % 2 ==0 ? teamA:teamB;  
                em.persist(new Member("member" + i, i, selectedTeam));  
            }  
        }  
    }  
  
}
```

`@Profile("local")` 

여기있는 이거 때문에 테스트 때는 스프링을 돌려도 실행되지 않는다.

뭐 100명 멤버 넣는 로직이다.

![](https://i.imgur.com/aJFq0cU.png)

다음과 같이 `The following 1 profile is active: "local"`이라고 로그에 나온다.


샘플 데이터를 만들었다면 이제 컨트롤러를 만들어 보자.

`src/main/java/study/querydsl/controller/MemberController.java`
```java
@RestController  
@RequiredArgsConstructor  
public class MemberController {  
  
    private final MemberJpaRepository memberJpaRepository;  
  
    @GetMapping("/v1/members")  
    public List<MemberTeamDto> searchMemberV1(MemberSearchCondition condition) {  
        return memberJpaRepository.search(condition);  
    }  
}
```

우리가 이전에 만들어 놨던 `MemberTeamDto`로 반환을 하고


`MemberSearchCondition`으로 검색 조건을 완성한다.

![](https://i.imgur.com/COH3UTV.png){: .align-center}

결과가 잘 나온다. 

검색 조건을 추가해 보자.

`?teamName=teamB&ageGoe=31&ageLoe=35&username=member31`

![](https://i.imgur.com/xB2K7sx.png){: .align-center}

검색 조건을 추가해도 아주 잘 나오는 걸 볼 수 있다.



