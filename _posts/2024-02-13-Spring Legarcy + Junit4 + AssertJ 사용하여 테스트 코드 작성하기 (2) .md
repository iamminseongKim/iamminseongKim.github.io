---
title: Spring Legarcy + Junit4 + AssertJ 사용하여 테스트 코드 작성하기 (2)
aliases: 
tags:
  - test
  - spring
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-02-13
last_modified_at: 2024-02-13
---
이번엔 서비스 테스트 코드를 작성해 보자.

> 참고 : 이전 글에 세팅 방법이 있다. [Spring Legarcy + Junit4 + AssertJ 사용하여 테스트 코드 작성하기](https://iamminseongkim.github.io/spring/Spring-Legarcy-+-Junit4-+-AssertJ-%EC%82%AC%EC%9A%A9%ED%95%98%EC%97%AC-%ED%85%8C%EC%8A%A4%ED%8A%B8-%EC%BD%94%EB%93%9C-%EC%9E%91%EC%84%B1%ED%95%98%EA%B8%B0/) 

자 먼저 서비스단 코드를 보자.

```java
@Service  
public class TestService {  
    private final TestDao testDao;  
  
    @Autowired  
    public TestService(TestDao testDao) {  
        this.testDao = testDao;  
    }  
  
    public int service1() {  
        int result = testDao.getTotalCount();  
        return result * 2;  
    }  
}
```

자 우리는 service1() 이라는 메서드를 테스트 할 것이다. service1() 메서드는 DB에서 전체 숫자를 가져와서 2배를 해주는 메서드다. 이를 위해서 testDao와 연관 관계를 맺었다.

이제 단위 테스트와 스프링을 이용해서 통합 테스트를 진행해보자.

## 1. 단위 테스트

먼저 `TestServiceTest.java`를 만들어 주자.

```java
@RunWith(MockitoJUnitRunner.class)  
public class TestServiceTest {  
    @Test  
    public void service1Test() throws Exception {  
        //given  
        //when        
        //then    
    }  
}
```
다음과 같이 뼈대를 잡았다. `@RunWith(MockitoJUnitRunner.class)` 을 사용한다.

이제 전체 코드를 작성해 보자.
먼저 DAO를 의존하니깐 @Mock 어노테이션을 이용해서 가짜 DAO를 만들고, 
그 다음 @InjectMocks을 이용해서 TestService를 가져오면서 DAO를 심어준다.

```java
public class TestServiceTest {  
    @Mock TestDao testDao;  
    @InjectMocks TestService testService;
    ...
}
```

그 다음 이제 service1() 메서드를 테스트 한다.  필요한 걸 적어보자.

1. `testDao.getTotalCount()`는 뭐가 나올지 모른다. 
2. 그럼 가짜 값이 필요하다. `when().thenReturn()` 을 사용하자.
3. `service1()`을 호출해서 값을 받자.
4. `assertThat().isEqualTo()`를 사용해서 값을 검증하자.

이렇게 코드를 작성하면 될 거 같다.

```java
@Test  
public void service1Test() throws Exception {  
    //given  
    int totalCount = 10;  
    when(testDao.getTotalCount()).thenReturn(totalCount);  
    //when  
    int result = testService.service1();  
    //then  
    assertThat(result).isEqualTo(totalCount*2);  
}
```
나는 다음과 같이 작성했다.

어쨋든 service1() 메서드는 totalCount를 2배 해주는 메서드 이기 때문에 내가 설정한 가짜 값을 2배 한 값과 일치하면 된다.

![](https://i.imgur.com/o0SBsQp.png)

결과를 보면 깔끔하게 통과한 걸 볼 수 있다.

## 2. Spring을 사용한 통합(?) 테스트 

이젠 Spring을 사용해서 DAO에서 직접 값을 가져와서 service1() 메서드를 테스트 해보자

먼저 뼈대를 잡자. `TestServiceTestWithSpring.java`를 만들겠다.

```java
@RunWith(SpringJUnit4ClassRunner.class)  
@ContextConfiguration(locations = {"classpath*:spring/root-context.xml", "classpath*:spring/appServlet/*.xml"})  
@WebAppConfiguration  
public class TestServiceTestWithSpring {  
  
    @Autowired TestService testService;  
  
    @Test  
    public void service1Test() throws Exception {  
        //given  
        //when        
        //then    
    }  
}
```

컨트롤러 테스트 때와 마찬가지로 `@RunWith(SpringJUnit4ClassRunner.class)`, `@ContextConfiguration`, `@WebAppConfiguration`어노테이션을 사용한다.

그 후 `@Autowired TestService testService;`로 서비스를 테스트 해보자.

1. `//given`에선 줄 게 없다. 줄 게 있으면 여기서 세팅을 하자.
2. `//when` 에선 `service1()` 메서드를 실행하자.
3. `//then`에선 `service1()` 에서 가져온 값을 검증 하자.

참고로 testDAO에서 getTotalCount() 메소드는 다음과 같다.

```java
@Repository  
public class TestDao {  
    public int getTotalCount() {  
        return 10;  
    }  
}
```
10을 리턴 해주고 있다. (원래라면 DB 값을 가져온다.)

이제 test 코드를 작성하면

```java
@Test  
public void service1Test() throws Exception {  
    //given  
    //when    
    int result = testService.service1();  
    //then  
    assertThat(result).isEqualTo(20);  
}
```

DB에서 10을 주기로 했고, 2배 해준 20을 검증해 보면 된다.
더 정확히 하고 싶으면 getTotalCount를 따로 실행해서 미리 변수로 빼놓고 예상 값에 x2 해주면 될 거 같다. 
아무튼 실행해보면

![](https://i.imgur.com/cuzRIgi.png)

잘 통과 된 걸 볼 수 있다.

그런데 스프링을 이용해서 테스트 한 코드는 무슨 실행할 때 6초나 걸린다. 이는 스프링을 먼저 올려 놓고 그 다음에 테스트를 작동하므로 스프링 올린 시간이 포함 돼서 그런 것 같다.

아무튼 서비스단도 단위 테스트 및 Spring을 사용한 통합 테스트도 진행해 보았다. 


