---
title: Querydsl - 스프링 데이터 JPA가 지원하는 Querydsl 기능 (2)
aliases: 
tags:
  - queryDSL
  - jpa
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-03-11
last_modified_at: 2024-03-11
---
> 인프런 실전! Querydsl 강의 내용 정리

[스프링 데이터 JPA가 지원하는 Querydsl 기능 1](https://iamminseongkim.github.io/jpa/Querydsl-%EC%8A%A4%ED%94%84%EB%A7%81-%EB%8D%B0%EC%9D%B4%ED%84%B0-JPA%EA%B0%80-%EC%A0%9C%EA%B3%B5%ED%95%98%EB%8A%94-Querydsl-%EA%B8%B0%EB%8A%A5-(1)/)

여기서 소개하는 기능은 제약이 커서 복잡한 실무 환경에서 사용하기에는 많이 부족하다.


## 리포지토리 지원 - QuerydslRepositorySupport

**장점**
- `getQuerydsl().applyPagination()`스프링 데이터가 제공하는 페이징을 Querydsl로 편리하게 변환 가능. *(단! Sort는 오류 발생)*
- `from()`으로 시작 가능(최근에는 QueryFactory를 사용해서 `select()`로 시작하는 것이 더 명시적)
- EntityManager 재공

**한계**
- `Querydsl 3.x`버전을 대상으로 만듦
- `Querydsl 4.x`에 나온 JPAQueryFactory 로 시작할 수 없음
	- select로 시작할 수 없음 (from으로 시작해야 함)
- `QueryFactory`를 제공하지 않음
- 스프링 데이터 Sort 기능이 정상 동작 하지 않음.


`MemberRepositoryImpl`
```java
public class MemberRepositoryImpl extends QuerydslRepositorySupport implements MemberRepositoryCustom { 

	private final JPAQueryFactory queryFactory;  
	  
	/*public MemberRepositoryImpl(EntityManager em) {  
	    this.queryFactory = new JPAQueryFactory(em);}*/  
	  
	public MemberRepositoryImpl(EntityManager em) {  
	    super(Member.class);  
	    this.queryFactory = new JPAQueryFactory(em);  
	}
}
```
`extends QuerydslRepositorySupport` 이걸 받아주고 생성자에 

```java
public MemberRepositoryImpl(EntityManager em) {  
	super(Member.class);  
	this.queryFactory = new JPAQueryFactory(em);  
}
```

다음과 같이 받아주면 사용이 가능하다.

```java
@Override  
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

자 이걸 한번 바꿔보자.

```java
List<MemberTeamDto> fetch = from(member)  
	.leftJoin(member.team, team)  
	.where(  
			usernameEq(condition.getUsername()),  
			teamNameEq(condition.getTeamName()),  
			ageGoe(condition.getAgeGoe()),  
			ageLoe(condition.getAgeLoe())  
	)  
	.select(new QMemberTeamDto(  
			member.id.as("memberId"),  
			member.username,  
			member.age,  
			team.id.as("teamId"),  
			team.name.as("teamName")  
	))  
	.fetch();
```

이런 식으로 `from()`부터 바로 시작할 수 있다. `queryFactory` 사용 없이.

흠..

그럼 유일한 장점을 함 보자.

```java
JPQLQuery<MemberTeamDto> jpaQuery = from(member)  
        .leftJoin(member.team, team)  
        .where(  
                usernameEq(condition.getUsername()),  
                teamNameEq(condition.getTeamName()),  
                ageGoe(condition.getAgeGoe()),  
                ageLoe(condition.getAgeLoe())  
        )  
        .select(new QMemberTeamDto(  
                member.id.as("memberId"),  
                member.username,  
                member.age,  
                team.id.as("teamId"),  
                team.name.as("teamName")  
        ));  
  
JPQLQuery<MemberTeamDto> query = getQuerydsl().applyPagination(pageable, jpaQuery);  
  
List<MemberTeamDto> results = query.fetch();
```

페이징이 있는 querydsl 구문인데 다음과 같이 <br>
`JPQLQuery<MemberTeamDto> query = getQuerydsl().applyPagination(pageable, jpaQuery);`

을 사용해서 

```java
.offset(pageable.getOffset())  
.limit(pageable.getPageSize())
```
이걸 쿼리에서 안 넣어 줘도 된다..... 이게 장점이라고 한다..

아무튼 몇 줄 축약할 수 있다..

난 안쓸꺼다.

## Querydsl 지원 클래스 직접 만들기

스프링 데이터가 제공하는 `QuerydslRepositiorySupport`가 지닌 한계를 극복하기 위해 직접 Querydsl 지원 클래스를 만들어 보자.

**장점**
- 스프링 데이터가 제공하는 페이징을 편리하게 변환
- 페이징과 카운트 쿼리 분리 가능
- 스프링 데이터 Sort 지원
- `select()`, `selectFrom()`으로 시작 가능
- `EntityManager`, `QueryFactory` 제공


영한님이 만드신 `Querydsl4RepositorySupport.java` 이다.

```java
package study.querydsl.repository.support;  
  
import com.querydsl.core.types.EntityPath;  
import com.querydsl.core.types.Expression;  
import com.querydsl.core.types.dsl.PathBuilder;  
import com.querydsl.jpa.impl.JPAQuery;  
import com.querydsl.jpa.impl.JPAQueryFactory;  
import jakarta.annotation.PostConstruct;  
import jakarta.persistence.EntityManager;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.data.domain.Page;  
import org.springframework.data.domain.Pageable;  
import org.springframework.data.jpa.repository.support.JpaEntityInformation;  
import org.springframework.data.jpa.repository.support.JpaEntityInformationSupport;  
import org.springframework.data.jpa.repository.support.Querydsl;  
import org.springframework.data.querydsl.SimpleEntityPathResolver;  
import org.springframework.data.support.PageableExecutionUtils;  
import org.springframework.stereotype.Repository;  
import org.springframework.util.Assert;  
  
import java.util.List;  
import java.util.function.Function;  
  
@Repository  
public abstract class Querydsl4RepositorySupport {  
    private final Class domainClass;  
    private Querydsl querydsl;  
    private EntityManager entityManager;  
    private JPAQueryFactory queryFactory;  
  
    public Querydsl4RepositorySupport(Class<?> domainClass) {  
        Assert.notNull(domainClass, "Domain class must not be null!");  
        this.domainClass = domainClass;  
    }  
  
    @Autowired  
    public void setEntityManager(EntityManager entityManager) {  
        Assert.notNull(entityManager, "EntityManager must not be null!");  
        JpaEntityInformation entityInformation =  
                JpaEntityInformationSupport.getEntityInformation(domainClass, entityManager);  
        SimpleEntityPathResolver resolver = SimpleEntityPathResolver.INSTANCE;  
        EntityPath path = resolver.createPath(entityInformation.getJavaType());  
        this.entityManager = entityManager;  
        this.querydsl = new Querydsl(entityManager, new  
                PathBuilder<>(path.getType(), path.getMetadata()));  
        this.queryFactory = new JPAQueryFactory(entityManager);  
    }  
    @PostConstruct  
    public void validate() {  
        Assert.notNull(entityManager, "EntityManager must not be null!");  
        Assert.notNull(querydsl, "Querydsl must not be null!");  
        Assert.notNull(queryFactory, "QueryFactory must not be null!");  
    }  
    protected JPAQueryFactory getQueryFactory() {  
        return queryFactory;  
    }  
    protected Querydsl getQuerydsl() {  
        return querydsl;  
    }  
    protected EntityManager getEntityManager() {  
        return entityManager;  
    }  
    protected <T> JPAQuery<T> select(Expression<T> expr) {  
        return getQueryFactory().select(expr);  
    }  
    protected <T> JPAQuery<T> selectFrom(EntityPath<T> from) {  
        return getQueryFactory().selectFrom(from);  
    }  
    protected <T> Page<T> applyPagination(Pageable pageable,  
                                          Function<JPAQueryFactory, JPAQuery> contentQuery) {  
        JPAQuery jpaQuery = contentQuery.apply(getQueryFactory());  
        List<T> content = getQuerydsl().applyPagination(pageable,  
                jpaQuery).fetch();  
        return PageableExecutionUtils.getPage(content, pageable,  
                jpaQuery::fetchCount);  
    }  
    protected <T> Page<T> applyPagination(Pageable pageable,  
                                          Function<JPAQueryFactory, JPAQuery> contentQuery, Function<JPAQueryFactory,  
            JPAQuery> countQuery) {  
        JPAQuery jpaContentQuery = contentQuery.apply(getQueryFactory());  
        List<T> content = getQuerydsl().applyPagination(pageable,  
                jpaContentQuery).fetch();  
        JPAQuery countResult = countQuery.apply(getQueryFactory());  
        return PageableExecutionUtils.getPage(content, pageable,  
                countResult::fetchCount);  
    }  
}
```

이걸로 뭘 할 수 있는지 보자.
`MemberTestRepository.java`를 만들어 보자.

```java
public class MemberTestRepository extends Querydsl4RepositorySupport { 
	public MemberTestRepository() {  
	    super(Member.class);  
	}
}
```
이렇게 세팅해 놓으면 사용 준비가 끝났다.

먼저 간단하게 리스트 가져오는 메서드를 만들어 보자.

```java
public List<Member> basicSelect() {  
    return select(member)  
            .from(member)  
            .fetch();  
}  
  
public List<Member> basicSelectFrom() {  
    return selectFrom(member)  
            .fetch();  
}
```

많이 줄어들었다.. ㅋㅋ

그 다음 앞에서 `QuerydslRepositiorySupport`를 이용해 만든 페이징 쿼리와 영한님이 만든 support 를 이용한 쿼리를 비교해 보자.

#### QuerydslRepositiorySupport 페이징

```java
public Page<Member> searchPageByApplyPage(MemberSearchCondition condition, Pageable pageable) {  
    JPAQuery<Member> query = selectFrom(member)  
            .leftJoin(member.team, team)  
            .where(usernameEq(condition.getUsername()),  
                    teamNameEq(condition.getTeamName()),  
                    ageGoe(condition.getAgeGoe()),  
                    ageLoe(condition.getAgeLoe())  
            );  
  
    List<Member> content = getQuerydsl().applyPagination(pageable, query).fetch();  
  
    return PageableExecutionUtils.getPage(content, pageable, query::fetchCount);  
}
```
자 전에 배운 걸로 만든 것이고 <br>
`List<Member> content = getQuerydsl().applyPagination(pageable, query).fetch();` 
체인이 한번 끊어져야 하지만  짧아지긴 했다.

#### Querydsl4RepositorySupport (영한) - 페이징 

```java
public Page<Member> applyPagination(MemberSearchCondition condition, Pageable pageable) {  
    return applyPagination(pageable, query ->  
            query.selectFrom(member)  
                    .leftJoin(member.team, team)  
                    .where(  
                            usernameEq(condition.getUsername()),  
                            teamNameEq(condition.getTeamName()),  
                            ageGoe(condition.getAgeGoe()),  
                            ageLoe(condition.getAgeLoe())  
                    )  
    );  
}
```
안 끊기고 한번에 쭉 가는 걸 볼 수 있다. 그 대신 

```java
protected <T> Page<T> applyPagination(Pageable pageable,  
                                      Function<JPAQueryFactory, JPAQuery> contentQuery) {  
    JPAQuery jpaQuery = contentQuery.apply(getQueryFactory());  
    List<T> content = getQuerydsl().applyPagination(pageable,  
            jpaQuery).fetch();  
    return PageableExecutionUtils.getPage(content, pageable,  
            jpaQuery::fetchCount);  
}
```

실제 `applyPagination()` 메서드를 까보면 두 번째 파라미터가 함수다. 

그래서 `query -> query.select...` 이런 식으로 람다식을 넣어줬다.

그리고 applyPagination() 메서드를 잘 보면 QuerydslRepositiorySupport를 그대로 옮겨서 구현해 놓은 걸 볼 수 있다. 이로서 실제 구현 시에는 짧은 코드를 만들 수 있게 된다.

나도 이렇게 라이브러리 들을 내 방식대로 개선해 나갈 수 있는 개발자가 되고 싶다.

마지막으로 페이징 , 카운터 쿼리를 나눠서 하는 메서드를 만들어 보자.

```java
public Page<Member> applyPagination2(MemberSearchCondition condition, Pageable pageable) {  
    return applyPagination(pageable,  
            contentQuery -> contentQuery  
                    .selectFrom(member)  
                    .leftJoin(member.team, team)  
                    .where(  
                            usernameEq(condition.getUsername()),  
                            teamNameEq(condition.getTeamName()),  
                            ageGoe(condition.getAgeGoe()),  
                            ageLoe(condition.getAgeLoe())  
                    ),  
  
            countQuery -> countQuery  
                    .select(member.id)  
                    .from(member)  
                    .leftJoin(member.team, team)  
                    .where(  
                            usernameEq(condition.getUsername()),  
                            teamNameEq(condition.getTeamName()),  
                            ageGoe(condition.getAgeGoe()),  
                            ageLoe(condition.getAgeLoe())  
                    )  
            );  
}
```

이건 다를 거 없고 파라미터가 3개이고, 3번째 파라미터는 카운터 쿼리 함수를 받는다.


>이로써 Querydsl 강의가 끝났다... JPA를 배우면서 정말 재밌었고, 이런 기능이 있다는 거에 신기했고, 벌써 끝나서 아쉽다.. <br>이제 내가 직접 사용해 보면서 실력을 더 쌓아가야 한다. 

