---
title: 스프링 MVC - 1편 - HttpServletResponse - 기본 사용법
aliases: 
tags:
  - spring
  - servlet
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-03-18
last_modified_at: 2024-03-18
---

>  인프런 스프링 MVC 1편 - 백엔드 웹 개발 핵심 기술편을 학습하고 정리한 내용 입니다.

### HttpServletResponse 역할

**HTTP 응답 메시지 생성**
- HTTP 응답코드 지정
- 헤더 생성
- 바디 생성

**편의 기능 제공**
- Content-Type, 쿠키, Redirect


### HttpServletResponse 기본 사용법

`hello.servlet.basic.response.ResponseHeaderServle`
```java
@WebServlet(name = "responseHeaderServlet", urlPatterns = "/response-header")  
public class ResponseHeaderServlet extends HttpServlet {  
  
    @Override  
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {  
  
        //[status-line]  
        response.setStatus(HttpServletResponse.SC_OK); // 200  
  
        //[response-headers]        
        response.setHeader("Cache-Control", "no-cache, no-store, must-revalidate");  
        response.setHeader("Pragma", "no-cache");  
        response.setHeader("my-header", "hello");  
  
        // header 편의 메서드  
        content(response);  
  
        // cookie 편의 메서드  
        cookie(response);  
  
        // redirect 편의 메서드  
        redirect(response);  
  
        //[message body]  
        PrintWriter writer = response.getWriter();  
        writer.println("ok");  
    }
}
```

기본적으로 Response 는 Http 응답 규칙에 맞게 해주면 된다.

먼저 첫 번째로 `status-line`은 말 그대로 응답 코드이며 `response.setStatus()` 에 값을 넣어주면 된다.

`200` 이렇게 넣어도 되지만, 그것 보다는 `HttpServletResponse.SC_OK` 인터페이스를 활용해 보자.

![](https://i.imgur.com/UFT85sA.png){: .align-center}

모든 status 코드가 다 있으니깐, 잘 활용하면 될 거 같다.


그 다음은 헤더 정보를 세팅해 보자.
`response.setHeader("Cache-Control", "no-cache, no-store, must-revalidate");`

이건 캐시를 저장 안 하겠다는 코드이다. [웹 서비스 캐시 똑똑하게 다루기](https://toss.tech/article/smart-web-service-cache) 참고해 보자.

어쨌든 헤더에 뭔가를 보내려면 `response.setHeader(이름, 값);` 이런 형식으로 보내면 된다.

그래서 `response.setHeader("my-header", "hello");` 이런 것도 가능하다.

또 header 편의 메서드를 만들 보면

```java
private void content(HttpServletResponse response) {  
    //Content-Type: text/plain;charset=utf-8  
    //Content-Length: 2    //response.setHeader("Content-Type", "text/plain;charset=utf-8");  
    response.setContentType("text/plain");  
    response.setCharacterEncoding("utf-8");  
    //response.setContentLength(2); //(생략시 자동 생성)  
}
```
다음과 같이 `setContentType`, `setCharacterEncoding` 메서드로 값만 넣어주면 헤더 세팅이 완료되게 할 수 있다.

다음으로 쿠키 편의 메서드를 보자

```java
private void cookie(HttpServletResponse response) {  
    //Set-Cookie: myCookie=good; Max-Age=600;  
    //response.setHeader("Set-Cookie", "myCookie=good; Max-Age=600");    
    Cookie cookie = new Cookie("myCookie", "good");  
    cookie.setMaxAge(600); //600초  
    response.addCookie(cookie);  
}
```
다음과 같이 쿠키를 만들어서 response에 추가 시킬 수 있다.

마지막으로 redirect 편의 메서드를 보자.

```java
private void redirect(HttpServletResponse response) throws IOException {  
    //Status Code 302  
    //Location: /basic/hello-form.html    
    //response.setStatus(HttpServletResponse.SC_FOUND); //302    
    //response.setHeader("Location", "/basic/hello-form.html");
	response.sendRedirect("/basic/hello-form.html");  
}
```

자 이 한 줄로 리다이렉트를 시킬 수 있다. 참고로 주석들은 저 `sendRedirect` 메서드를 사용 안하고 

원래 기본적으로 세팅하면 저렇게 사용해야 한다는 것이다.

참고로 리다이렉트 상태 코드는 302 이다.


이제 결과를 확인해 보자.

![](https://i.imgur.com/HVN37UZ.png){: .align-center}

우리가 지정해 놓은 값들로 잘 넘어온 걸 볼 수 있다.

