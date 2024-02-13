---
title: Spring Legarcy + Junit4 + AssertJ 사용하여 테스트 코드 작성하기
aliases: 
tags:
  - spring
  - test
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-02-13
last_modified_at: 2024-02-13
---
회사 프로젝트 상 Spring 4.xx 를 사용해야 하는 경우가 많고, 또 Junit5는 올릴 수 없는 버전 이라서 Junit4 및 AssertJ를 사용해서 스프링 레거시 프로젝트 테스트를 해보자.

> 참고 : 본인은 인텔리제이를 사용함.

## 1. 라이브러리 의존성 설정
먼저 spring-test, junit4, assertj, mockito를 사용했다.
```xml
<dependency>  
    <groupId>junit</groupId>  
    <artifactId>junit</artifactId>  
    <version>4.13.2</version>  
    <scope>test</scope>  
</dependency>  
<!--assertj 로 테스트 검증 -->  
<dependency>  
    <groupId>org.assertj</groupId>  
    <artifactId>assertj-core</artifactId>  
    <!-- use 2.9.1 for Java 7 projects -->  
    <version>3.11.1</version>  
    <scope>test</scope>  
</dependency>  
<dependency>  
    <groupId>org.springframework</groupId>  
    <artifactId>spring-test</artifactId>  
    <version>4.0.7.RELEASE</version>  
    <scope>test</scope>  
</dependency>  
<!-- https://mvnrepository.com/artifact/org.mockito/mockito-core -->  
<dependency>  
    <groupId>org.mockito</groupId>  
    <artifactId>mockito-core</artifactId>  
    <version>3.12.4</version>  
    <scope>test</scope>  
</dependency>  
<!-- https://mvnrepository.com/artifact/org.mockito/mockito-inline -->  
<dependency>  
    <groupId>org.mockito</groupId>  
    <artifactId>mockito-inline</artifactId>  
    <version>3.12.4</version>  
    <scope>test</scope>  
</dependency>
```


이제 컨트롤러, 서비스 두 가지 예시로 테스트 코드를 작성해 보자.

## 2. 컨트롤러 테스트 

TestController.java
```java
@Controller  
public class TestController {  
    private final TestService testService;  
  
    @Autowired  
    public TestController(TestService testService) {  
        this.testService = testService;  
    }  
  
    @RequestMapping("/test/test")  
    public String test(){  
        int test = testService.test();  
        return "success";  
    }  
}
```
이렇게 간단한 컨트롤러를 만들었다.

그럼 테스트 코드를 만들어 보자

![](https://i.imgur.com/2PBUMsF.png)

이렇게 만들거나 `Ctrl + Shift + t` 를 눌러서 테스트 java파일을 만들자 

![](https://i.imgur.com/xaEFIij.png)

그럼 가장 중요하게 볼 게 JUnit4로 설정 되어 있는지 확인. 이상 없으면 OK.

![](https://i.imgur.com/69TpF5Q.png)

그럼 다음과 같이 src/test 하위에 main과 마찬가지로 같은 경로에 파일이 생성 된다.

#### mockito를 이용한 단위 테스트
그럼 이제 mockito를 이용해서 단위 테스트를 진행해보자.

여기서 가장 중요한 어노테이션은 다음과 같다.

1. `@RunWith(MockitoJUnitRunner.class)`
2. `@Test`

**@RunWith**은 junit4에서 테스트를 시작할 때 같이 돌아가야 하는 클래스 즉, ApplicationContext를 만들고 관리하는 작업을 @RunWith에 설정된 class로 이용하겠다는 뜻.

**@Test** 는 Junit이 실행시킬 메서드라는 걸 알리기 위한 어노테이션.


그럼 기본 뼈대는 다음과 같다.

```java
import org.junit.Test;  
import org.junit.runner.RunWith;  
import org.mockito.junit.MockitoJUnitRunner;  
  
@RunWith(MockitoJUnitRunner.class)  
public class TestControllerTest {  
  
    @Test  
    public void testTest() throws Exception {  
        //given  
        //when  
        //then  
    }  
}
```

이제 컨트롤러에 test 메소드를 테스트 해보자.

작성한 전체 코드는 다음과 같다.
```java
import org.junit.Test;  
import org.junit.runner.RunWith;  
import org.mockito.InjectMocks;  
import org.mockito.Mock;  
import org.mockito.junit.MockitoJUnitRunner;  
import org.springframework.test.web.servlet.MockMvc;  
  
import static org.assertj.core.api.Assertions.assertThat;  
import static org.mockito.Mockito.when;  
  
@RunWith(MockitoJUnitRunner.class)  
public class TestControllerTest {  
  
    @Mock  
    private TestService testService;  
    @InjectMocks  
    private TestController testController;  
    
    @Test  
    public void testTest() throws Exception {  
        //given  
        int serviceResult = 1;  
        when(testService.test()).thenReturn(serviceResult);  
        //when  
        String result = testController.test();  
        //then  
        assertThat(result).isEqualTo("success");  
    }  
  
}
```

여기서 테스트 코드를 실행하기 전에 먼저 @Mock, @InjectMocks 어노테이션을 작성해줬다. 
이걸 왜 사용하냐면 testController에서 우리가 테스트할 코드는 
```java
@RequestMapping("/test/test")  
public String test(){  
	int test = testService.test();  
	return "success";  
}  
```
다음과 같고, 이때 `testService.test()` 라는 연관 관계가 있기 때문에 이걸 대체 해주기 위함이다.

 `testService.test()`  가 무슨 역할인지는 모르겠지만, **컨트롤러 만을 테스트** 하기 위해서 가짜 Mock객체로 바꿔주려면 `@Mock` 어노테이션이 필요하고, 이 @Mock을 받을 컨트롤러에서는 `@InjectMocks`을 사용하는 것이다.

그래서 다음과 같이 

```java
@Mock  
private TestService testService;  
@InjectMocks  
private TestController testController;  
```
가짜 서비스를 만들고 컨트롤러에 삽입하면 된다.

이제 테스트 코드를 알아보자

```java
@Test  
public void testTest() throws Exception {  
	//given  
	int serviceResult = 1;  
	when(testService.test()).thenReturn(serviceResult);  
	//when  
	String result = testController.test();  
	//then  
	assertThat(result).isEqualTo("success");  
}
```
보면 먼저 `@Test`로 이 메서드가 테스트에서 실행돼야 함을 알리고, Junit4에선 반드시 public을 해줘야 한다. 

그리고 테스트 패턴은 여러가지가 있지만 `Given-When-Then`을 사용하겠다. 간단하게 설명하면 given에선 데이터 세팅 when에선 테스트 실행 then에선 테스트 검증이다.

그럼 given 에선 먼저 testService에 test()메서드를 가짜 값으로 바꿔주기 위한 코드를 작성한다.

```java
//given
int serviceResult = 1;  
when(testService.test()).thenReturn(serviceResult);  
```
when(메서드 이름).thenReturn(대체 값) 이렇게 보면 된다.


그 다음 when에선 이제 내가 테스트 하고자 하는 메서드를 테스트 한다.

```java
//when  
String result = testController.test();  
```
실행해서 결과 값을 저장.


이제 결과를 검증만 하면 테스트 가 완료 된다.
```java
//then  
assertThat(result).isEqualTo("success");
```
then에선 assertThat을 사용해서 검증을 하게 된다.
assertThat에 여러 기능들이 있겠지만, 여기선 assertThat().isEqualTo(); 로 쓰겟다.
**assertThat(받은 결과 값).isEqualTo(내가 예상하는 값);** 이렇게  작성하면 된다.

![](https://i.imgur.com/Eh7Fw0r.png)

최종적으로 실행 해보면 다음과 같이 초록 체크 불이 들어오면 테스트가 통과된 것이다.
만약에 기대한 값이 아니라 다른 값이 나오게 되어 테스트가 깨지면 다음과 같이 나온다.

![](https://i.imgur.com/nsVvp2D.png)

나는 false를 기대했지만 실제 값은 success 라고 친절하게 코드가 알려준다.


#### Spring를 이용한 통합(?) 테스트
그럼 전체적으로 이게 잘 돌아가는지 봐야 한다. Spring을 사용하면 설정 해줘야 할 게 쫌 많다.
먼저 테스트 파일을 하나 또 만들자. `TestControllerTestWithSpring.java`로 만들었다.

```java
import org.junit.Before;  
import org.junit.Test;  
import org.junit.runner.RunWith;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.test.context.ContextConfiguration;  
import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;  
import org.springframework.test.context.web.WebAppConfiguration;  
import org.springframework.test.web.servlet.MockMvc;  
import org.springframework.test.web.servlet.setup.MockMvcBuilders;  
  
import static org.junit.Assert.*;  
  
@RunWith(SpringJUnit4ClassRunner.class)  
@ContextConfiguration(locations = {"classpath*:spring/root-context.xml", "classpath*:spring/appServlet/*.xml"})  
@WebAppConfiguration  
public class TestControllerTestWithSpring {  
    private MockMvc mockMvc;  
    
    @Autowired TestController testController;  
    
    @Before  
    public void setUp() {  
        this.mockMvc = MockMvcBuilders.standaloneSetup(testController).build();  
    }  
    
    @Test  
    public void testTestWithSpringMockMvc() throws Exception {  
        //given  
        //when        
        //then    
    }  
}
```
뼈대는 다음과 같다.. 매우 길어졌다.

먼저 하나하나 설명 해 보자면 
1. `@RunWith(SpringJUnit4ClassRunner.class)`  
	이제 스프링을 이용해 테스트를 진행하겠다는 어노테이션이다.
2. `@ContextConfiguration(locations = {"classpath*:spring/root-context.xml", "classpath*:spring/appServlet/*.xml"})` 
	이건 스프링에 bean같은 미리 설정해 놓은 파일들을 불러오라는 뜻인데, 설정 법이 다양하니 좀 더 알아보길 추천한다.
3. `@WebAppConfiguration` 
	[스프링 공식 문서](https://docs.spring.io/spring-framework/reference/testing/annotations/integration-spring/annotation-webappconfiguration.html) 참고해보길 바란다. 간단하게 웹 동작을 하기 위해 필요한 것들을 가져오기 위한 어노테이션.
4. `private MockMvc mockMvc;` 
	어플리케이션을 서버에 배포하지 않고도 스프링 MVC의 테스트를 진행할 수 있게 도와주는 클래스이다.
5.  `@Autowired TestController testController;` 
	테스트 할 컨트롤러
6.  `@Before`및  `setUp()` 메서드
	- Before은 Test가 실행 되기 전에 먼저 실행 돼야 할 메서드를 알려주는 어노테이션
	- setUp 메서드는 mockMvc에 testController를 담기 위한 사전 정의 메서드

이제 테스트 코드를 작성 할 준비가 되었다. test() 메서드를 테스트 해보자.

```java
@Test  
public void testTestWithSpringMockMvc() throws Exception {  
    mockMvc.perform(get("/test/test"))   
		.andExpect(status().isOk())   
		.andDo(print());  
    
    /*  
    * get() : MockMvcRequestBuilders.get() get 호출 하겠다.  
    * MockMvcResultMatchers.status(), isOk()  : 통신이 성공적인가 ? 200    
    * MockMvcResultHandlers.print() : 결과 출력  
    * */
}
```

이제 미리 만들어 놓은  mockMvc에 perform() 으로 특정 url을 호출 해보자. 우린 test 컨트롤러를 테스트 해야 하므로 get방식에 "/test/test"를 호출 했다. 그리고 `andExpect(status().isOk())`로 응답이 올바르게 왔는지 검증했고, 마지막으로 내용을 `andDo(print())`로 출력하였다.

결과를 보자.

![](https://i.imgur.com/2IwlC6Q.png)
![](https://i.imgur.com/KRVhqY6.png)

다음과 같이 테스트가 통과 하였고, Request 정보 및 Response 정보를 출력해 주고 있다.

이로써 컨트롤러에 대한 단위 테스트 및 스프링을 사용한 통합 테스트를 알아보았다.

서비스 단은 다음에 보자..

> [서비스단 테스트 코드 작성법 보러가기](https://iamminseongkim.github.io/spring/Spring-Legarcy-+-Junit4-+-AssertJ-%EC%82%AC%EC%9A%A9%ED%95%98%EC%97%AC-%ED%85%8C%EC%8A%A4%ED%8A%B8-%EC%BD%94%EB%93%9C-%EC%9E%91%EC%84%B1%ED%95%98%EA%B8%B0-(2)/)



