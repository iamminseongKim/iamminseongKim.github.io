---
title: Querydsl - 프로젝션과 결과 반환 - @QueryProjection
aliases: 
tags:
  - queryDSL
  - jpa
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-02-27
last_modified_at: 2024-02-27
---

> 인프런 실전! Querydsl 강의 내용 정리

## 생성자  + @QueryProjection

먼저 DTO 생성자에 `@QueryProjection`를 붙혀준다.

![](https://i.imgur.com/e6yCRKU.png){: .align-center}

그 다음 gradle compile 해주면

![](https://i.imgur.com/LR5L9Or.png){: .align-center}

다음과 같이 QDTO(?) 가 생긴다.

```java
@Test  
public void findDtoByQueryProjection() throws Exception {  
    List<MemberDto> result = queryFactory  
            .select(new QMemberDto(member.username, member.age))  
            .from(member)  
            .fetch();  
  
    for (MemberDto memberDto : result) {  
        System.out.println("memberDto = " + memberDto);  
    }  
  
}
```

그럼 다음과 같이 테스트 코드를 작성할 수 있다.

![](https://i.imgur.com/SLx2ojm.png){: .align-center}

결과도 잘 나온다.

이 코드의 장점은 런타임 시점이 아니라 컴파일 시점에서

생성자에 없는 컬럼이 추가된다 거나 하는 오류를 잡아 낼 수 있다는 점이다.


![](https://i.imgur.com/LN7O0No.png){: .align-center}

그냥 생성자를 사용한 테스트 코드인데 지금 DTO에는 team이 없다. 

그래서 이건 지금 시점에선 에러가 안 나는데 실행 시키면 에러가 난다.

![](https://i.imgur.com/XiQhuPZ.png){: .align-center}

다음과 같이 말이다.

하지만 `@QueryProjection`은 아니다.

![](https://i.imgur.com/kxeaN0p.png){: .align-center}

다음과 같이 컴파일 시점에서 오류가 이미 발견 된다.

이걸 보면 단점이 없어 보인다.

하지만 딱 한 가지 고려할 점이 있다.

`@QueryProjection` 이거 하나 때문에 DTO에서 `QueryDSL`에 대한 의존 관계가 생겨버린다.<br>
그리고 이에 따라 DTO까지 Q파일을 생성해야 하는 단점.

![](https://i.imgur.com/Sa93kCd.png)

