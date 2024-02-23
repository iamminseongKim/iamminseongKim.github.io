---
title: Querydsl - Case 문, 상수 문자 더하기
aliases: 
tags:
  - jpa
  - queryDSL
categories:
  - jpa
toc: true
toc_label: 목차
date: 2024-02-23 14:00:48
last_modified_at: 2024-02-23
---
> 인프런 실전! Querydsl 강의 내용 정리

## Case 문

**select, 조건(where) 절에서 사용 가능**

### 간단한 case문

```java
@Test  
public void basicCase() throws Exception {  
    //given  
    //when    
    List<String> result = queryFactory  
            .select(member.age  
                    .when(10).then("열살")  
                    .when(20).then("스무살")  
                    .otherwise("기타"))  
            .from(member)  
            .fetch();  
  
    //then  
    for (String s : result) {  
        System.out.println("s = " + s);  
    }  
}
```

select 절에서 간단하게 `when().then()` `otherwise()` 직관적 이게 사용 가능 하다.

![](https://i.imgur.com/KZPZVse.png){: .align-center}

원하는 대로 잘 나온다.

### 복잡한 case

```java
@Test  
public void complexCase() throws Exception {  
    //given  
    //when    
    List<String> result = queryFactory  
            .select(new CaseBuilder()  
                    .when(member.age.between(0, 20)).then("0~20살")  
                    .when(member.age.between(21, 30)).then("21~30살")  
                    .otherwise("기타"))  
            .from(member)  
            .fetch();  
    //then  
    for (String s : result) {  
        System.out.println("s = " + s);  
    }  
}
```

`new CaseBuilder()`를 사용하고, `between` 같은 기능을 사용했다.

![](https://i.imgur.com/bZcbXD7.png){: .align-center}

다음과 같이 잘 나왔다. 
그런데 굳이 이렇게 DB에서 가공할 필요가 있나 싶다.


## 상수, 문자 더하기

1. 상수 더하기

```java
@Test  
public void constant() throws Exception {  
    //given  
    //when  
    List<Tuple> fetch = queryFactory  
            .select(member.username, Expressions.constant("A"))  
            .from(member)  
            .fetch();  
  
    //then  
    for (Tuple tuple : fetch) {  
        System.out.println("tuple = " + tuple);  
    }  
}
```

다음과 같이 조회할 데이터 , `Expressions.constant("상수")`  를 넣어주면 된다.

![](https://i.imgur.com/mMWifqF.png){: .align-center}

다음과 같이 튜플로 A 가 같이 붙어 나오는 걸 볼 수 있다.

2. 문자 더하기

사실상 이건 좀 쓰긴 할 것 같다.

```java
@Test  
public void concat() throws Exception {  
    //given  
    //when  
    List<String> member1 = queryFactory  
            .select(member.username.concat("-").concat(member.age.stringValue()))  
            .from(member)  
            .where(member.username.eq("member1"))  
            .fetch();  
    //then  
    for (String s : member1) {  
        System.out.println("s = " + s);  
    }  
}
```

원하는 데이터`.concat()` 이런 식으로 붙혔다.

그리고 나이도  추가 하려 하니 나이는 int형이라 또  String 으로 변환해 줘야 한다.

그래서 meber.age.`stringVaule()`를 사용해서 나이를 문자열로 바꾼 후 다시 `.concat()` 을 사용하였다.

![](https://i.imgur.com/20oGSzR.png){: .align-center}

재밌게도 `cast(age as varchar)` 이렇게 바뀐 걸 볼 수 있다. (하이버네이트 방언 마다 바뀔 꺼 같긴 함.)




