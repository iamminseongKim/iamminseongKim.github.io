---
title: 스프링 MVC - 1편 - HttpServletRequest
aliases: 
tags:
  - java
  - spring
  - servlet
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-03-15
last_modified_at: 2024-03-18
---
>  인프런 스프링 MVC 1편 - 백엔드 웹 개발 핵심 기술편을 학습하고 정리한 내용 입니다.


HTTP 요청 메세지를 통해 클라이언트에서 서버로 데이터를 전달하는 방법을 알아보자.

**주로 3가지 방법을 사용한다**

## GET - 쿼리 파라미터
- /url<span style="background:rgba(54, 188, 155, 0.3)"><font color="#ff0000">?username=hello&age=20</font></span>
-  메세지 바디 없이, URL의 쿼리 파라미터에 데이터를 포함해서 전달
- 예) 검색, 필터, 페이징 등에서 많이 사용하는 방식

다음 데이터를 클라이언트에서 서버로 전송 해 보자.

전달 데이터 : username=hello, age=20

메시지 바디 없이, URL의 **쿼리 파라미터**를 사용해서 데이터를 전달하자.

쿼리 파라미터는 URL에 다음과 같이 `?`를 시작으로 보낼 수 있다. 추가 파라미터는 `&`로 구분하면 된다.

> http://localhost:8080/request-param?username=hello&age=20

서버에서는 `HttpServletRequest`가 제공하는 다음 메서드를 통해 쿼리 파라미터를 편리하게 조회할 수 있다.

먼저 `java/hello/servlet/basic/request/RequestParamServlet.java` 생성

```java
@WebServlet(name="requestParamServlet", urlPatterns = "/request-param")  
public class RequestParamServlet extends HttpServlet {  
  
    @Override  
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {  
        System.out.println("[전체 파라미터 조회] - start ");  
  
        request.getParameterNames().asIterator()  
                        .forEachRemaining(paramName -> System.out.println(paramName + "=" + request.getParameter(paramName)));  
  
        System.out.println("[전체 파라미터 조회] - end ");  
        System.out.println();  
  
        System.out.println("[단일 파라미터 조회] - start ");  
        String username = request.getParameter("username");  
        String age = request.getParameter("age");  
  
        System.out.println("username = " + username);  
        System.out.println("age = " + age);  
        System.out.println("[단일 파라미터 조회] - end ");  
        System.out.println();  
  
        System.out.println("[이름이 같은 복수 파라미터 조회]");  
        String[] usernames = request.getParameterValues("username");  
        for (String name : usernames) {  
            System.out.println("username = " + name);  
        }  
  
        response.getWriter().write("ok!");  
    }  
}
```

전체 코드는 다음과 같다. 하나 하나 보자.

#### 전체 파라미터 조회

이건 이전에 헤더를 본 거와 같이 `request.getParameterNames()` 을 이용해서 모든 request 를 가져오는 방식이였다. [참고 - 헤더 정보 가져오기](https://iamminseongkim.github.io/spring/%EC%8A%A4%ED%94%84%EB%A7%81-MVC-1%ED%8E%B8-HttpServletRequest-%EC%82%AC%EC%9A%A9%EB%B2%95/#2-header%EC%9D%98-%EB%AA%A8%EB%93%A0-%EC%A0%95%EB%B3%B4)
 
#### 단일 파라미터 조회 
```java
System.out.println("[단일 파라미터 조회] - start ");  
String username = request.getParameter("username");  
String age = request.getParameter("age");  

System.out.println("username = " + username);  
System.out.println("age = " + age);  
System.out.println("[단일 파라미터 조회] - end ");  

```

가장 많이 쓰는 `request.getParameter(name);` 으로 값을 가져오는 방식이다.

#### 동일 파라미터  조회
혹시 http://localhost:8080/request-param?username=hello&age=20&username=hello2 

다음과 같이 요청하면 단일에선 어떻게 처리되는지 아는가? 

단일에선 `hello` 로 찍힐 것이다. 첫 번째 값을 반환한다.

모두 가져오려면 

```java
System.out.println("[이름이 같은 복수 파라미터 조회]");  
String[] usernames = request.getParameterValues("username");  
for (String name : usernames) {  
	System.out.println("username = " + name);  
}  
```
이런 식으로 조회 해야 한다.

#### 쿼리 파라미터 조회 메서드 

```java
String username = request.getParameter("username"); //단일 파라미터 조회 
Enumeration parameterNames = request.getParameterNames(); //파라미터 이름들 모두 조회 
Map parameterMap = request.getParameterMap(); //파라미터를 Map으로 조회 
String[] usernames = request.getParameterValues("username"); //복수 파라미터 조회
```

## POST - HTML Form
- content-type : application/x-www-form-urlencoded
- **메세지 바디**에 쿼리 파라미터 형식으로 전달 username=hello&age=20
- 예) 회원 가입, 상품 주문, HTML Form 사용

![](https://i.imgur.com/4aMXitd.png){: .align-center}

`src/main/webapp/basic/hello-form/html` 생성
```html
<!DOCTYPE html>  
<html>  
<head>  
    <meta charset="UTF-8">  
    <title>Title</title>  
</head>  
<body>  
<form action="/request-param" method="post">  
    username: <input type="text" name="username" />  
    age:      <input type="text" name="age" />  
    <button type="submit">전송</button>  
</form>  
</body>  
</html>
```

![](https://i.imgur.com/SMRYjUM.png){: .align-center}

다음과 같이 전송해 보자 form action 이 `/request-param` 즉 get방식에서 하던 서블릿 클래스와 같은 곳이다.

![](https://i.imgur.com/hiMRkft.png){: .align-center}

![](https://i.imgur.com/Gw1CCV0.png){: .align-center}

일단 브라우저 상에선 잘 넘어갔다.

그럼 서버를 보자.

![](https://i.imgur.com/e7QeVhO.png){: .align-center}

서버에서도 잘 넘어 왔다! 

> **메세지 바디**에 쿼리 파라미터 형식으로 전달 username=hello&age=20 

이런 방식이기 때문에 GET방식으로 만들어 놓은 곳이랑 같이 호환 된다.

 즉 `request.getParameter` 가 get 이든 post든 잘 가져온다는 것 이다.

> 참고 <br>content-type 은 HTTP 메세지 바디의 데이터 형식을 지정한다. <br>**GET URL 쿼리 파라미터 형식**으로 클라이언트에서 서버로 데이터를 전달할 때는 HTTP 메시지 바디를 사용하지 않기 때문에 content-type이 없다. <br>**POST HTML Form 형식**으로 데이터를 전달하면 HTTP 메시지 바디에 해당 데이터를 포함해서 보내기 때문에 바디에 포함된 데이터가 어떤 형식인지 content-type을 꼭 지정해야 한다. <br>이렇게 폼으로 데이터를 전홍하는 형식을 `application/x-www-form-urlencoded`라 한다.

##### Postman을 사용한 테스트
이런 간단한 테스트에 Html 폼 만들어서 하기는 너무 귀찮다. 
이럴땐 포스트맨을 사용하자.

![](https://i.imgur.com/5NGMNuf.png)
{: .align-center}

다음과 같이 url과 POST를 세팅해 주고 Body 탭에 `x-www-form-urlencoded`를 체크하고 key value에 값을 넣어서 보내면 된다.

## HTTP message body 에 데이터 직접 담아서 요청
- HTTP API에서 주로 사용, JSON, XML, TEXT
- 데이터 형식은 주로 JSON 사용
- POST, PUT, PATCH

먼저 가장 단순한 텍스트 메시지를 HTTP 메시지 바디에 담아서 전송하고, 읽어보자.
HTTP 메시지 바디의 데이터를 InputStream을 사용해서 읽을 수 있다.

`java/hello/servlet/basic/request/RequestBodyStringServlet.java`를 만들자
```java
@WebServlet(name = "requestBodySpringServlet", urlPatterns = "/request-body-string")  
public class RequestBodyStringServlet extends HttpServlet {  
    @Override  
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {  
        ServletInputStream inputStream = request.getInputStream();  
        String messageBody = StreamUtils.copyToString(inputStream, StandardCharsets.UTF_8);  
  
        System.out.println("messageBody = " + messageBody);  
        response.getWriter().write("ok!");  
    }  
}
```

- `request.getInputStream()` 으로 메시지 바디를 Byte로 가져온다.
- 이를 `StreamUtils.copyToString()` 등 여러가지 방법으로 String으로 변환한다.
- 이 때 Byte 를 String 으로 변환 하는 과정에서 `Charset(문자표)`를 반드시 지정 해 줘야 한다.
- 여기선 UTF-8 을 사용했다

결과를 Postman으로 보자.

![](https://i.imgur.com/zyCO1dP.png){: .align-center}

결과가 잘 나온 걸 볼 수 있다.

### JSON 

이번에는 HTTP API 에서 주로 사용하는 JSON 형식으로 데이터를 전달해 보자.

**JSON 형식 전송**
- POST http://localhost:8080/request-body-json
- content-type: **application/json**
- message body: {"username" : "hello", "age" : 20}
- 결과 : message body = {"username" : "hello", "age" : 20}

먼저 JSON 데이터를 파싱할 객체를 하나 생성하자.
`hello.servlet.basic.HelloData`
```java
@Getter @Setter  
public class HelloData {  
    private String username;  
    private int age;  
}
```
롬복 써서 getter, setter  만들었다.

이제 Servlet을 만들어 보자. 

`hello.servlet.basic.request.RequestBodyJsonServlet`
```java
@WebServlet(name = "requestBodyJsonServlet", urlPatterns = "/request-body-json")  
public class RequestBodyJsonServlet extends HttpServlet {  
  
    private ObjectMapper objectMapper = new ObjectMapper();  
  
    @Override  
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {  
  
        ServletInputStream inputStream = request.getInputStream();  
        String messageBody = StreamUtils.copyToString(inputStream, StandardCharsets.UTF_8);  
  
        System.out.println("messageBody = " + messageBody);  
        HelloData helloData = objectMapper.readValue(messageBody, HelloData.class);  
  
        System.out.println("helloData.username = " + helloData.getUsername());  
        System.out.println("helloData.age = " + helloData.getAge());  
  
        response.getWriter().write("ok");  
    }  
}
```

자 이거는 일단 

```java
ServletInputStream inputStream = request.getInputStream();  
String messageBody = StreamUtils.copyToString(inputStream, StandardCharsets.UTF_8); 
```
이 두 줄 까진 HTTP message body 에 데이터 직접 담아서 요청한 걸  가져오는 것 똫같다.

```java
HelloData helloData = objectMapper.readValue(messageBody, HelloData.class);
```

여기서 `Jackson` 라이브러리를 이용해서 우리가 만든 객체에 메세지 바디 JSON데이터를 파싱하게 된다.

밑에는 출력 관련 코드이고,  바로 PostMan으로 테스트 해보자.

![](https://i.imgur.com/pXKLyTj.png){: .align-center}

POST, Body, raw(json)인거 확인하고 값을 만들어서 보내면


![](https://i.imgur.com/77GBmB0.png){: .align-center} 

내가 원하는 대로 값을 잘 파싱해서 출력해 주고 있다.


> 참고<br> JSON 결과를 파싱해서 사용할 수 있는 자바 객체로 변환하려면 Jackson, Gson 같은 JSON 변환 라이브러리를 추가해서 사용해야 한다. 스프링 부트로 Spring MVC를 선택하면 기본으로 Jackson 라이브러리(`ObjectMapper`)를 함께 제공한다.

> 참고<br>HTML form 데이터도 메시지 바디를 통해 전송되므로 직접 읽을 수 있다. 하지만 편리한 파라미터 조회 기능 `request.getParameter(...)`을 이미 제공하기 때문에 파라미터 조회 기능을 사용하자.



