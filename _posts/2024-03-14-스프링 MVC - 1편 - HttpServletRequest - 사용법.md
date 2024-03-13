---
title: 스프링 MVC - 1편 - HttpServletRequest - 사용법
aliases: 
tags:
  - java
  - spring
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-03-14
last_modified_at: 2024-03-14
---
>  인프런 스프링 MVC 1편 - 백엔드 웹 개발 핵심 기술편을 학습하고 정리한 내용 입니다.

먼저 다음 위치에 서블릿 클래스를 하나 만들어 주자.

![](https://i.imgur.com/JtSzYvP.png){: .align-center}


```java
@WebServlet(name = "requestHeaderServlet", urlPatterns = "/request-header")  
public class RequestHeaderServlet extends HttpServlet {
	@Override  
	protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {  
	    printStartLine(request);  
	    printHeaders(request);  
	    printHeaderUtils(request);  
	    printEtc(request);  
	}
}
```

다음과 같이 위 4개의 메서드를 만들면서 rquest 에 담긴 헤더 정보를 읽는 방법을 배워보자.

### 1. 헤더 시작 정보

`printStartLine(HttpServletRequest request)`
```java
    private void printStartLine(HttpServletRequest request) {
        System.out.println("--- REQUEST-LINE - start ---");

        System.out.println("request.getMethod() = " + request.getMethod()); //GET
        System.out.println("request.getProtocal() = " + request.getProtocol()); //HTTP/1.1
        System.out.println("request.getScheme() = " + request.getScheme()); //http
        // http://localhost:8080/request-header
        System.out.println("request.getRequestURL() = " + request.getRequestURL());
        // /request-test
        System.out.println("request.getRequestURI() = " + request.getRequestURI());
        //username=hello
        System.out.println("request.getQueryString() = " + request.getQueryString());
        System.out.println("request.isSecure() = " + request.isSecure()); //https 사용 유무
        System.out.println("--- REQUEST-LINE - end ---");
        System.out.println();
    }
```

`request` 객체의 다양한 메서드를 이용해서 다음과 같은 정보를 확인해 볼 수 있다.

실제 사용해 보면

![](https://i.imgur.com/9ah3mXp.png){: .align-center}

다음과 같은 정보들이 나오게 된다.

### 2. Header의 모든 정보

이번엔 헤더의 모든 정보를 출력해 보자. 

`printHeaders(HttpServletRequest request)`
```java
private void printHeaders(HttpServletRequest request) {  
    System.out.println("--- Headers - start ---");  
  
    /*Enumeration<String> headerNames = request.getHeaderNames();  
    while (headerNames.hasMoreElements()) {        String headerName = headerNames.nextElement();        System.out.println( headerName + ":" + headerName);    }*/  
    request.getHeaderNames().asIterator()  
                    .forEachRemaining(headerName -> System.out.println(headerName + ":" + headerName));  
  
    System.out.println("--- Headers - end ---");  
    System.out.println();  
}
```

뭐 방식은 두 가지 정도가 있는데 직접 **while** 문을 돌리던가 아니면 `.asIterator()` 컬랙션 프레임워크를 사용하던가 아무튼 중요한 건 

`request.getHeaderNames()` 

이 안에 모든 헤더 정보들이 담겨 있다는 것이다.

결과를 보자.


![](https://i.imgur.com/RzpwzdE.png){: .align-center}

엄청 많은데, 이건 **브라우저**에서 만든 헤더 정보들이 들어가기 때문이다.

![](https://i.imgur.com/rGIKmYh.png){: .align-center}

이렇게 말이다.


### 3. Header 편의 조회

이번엔 실제 사용할 수 있을 지도 모르는 헤더 정보들을 가져오기 위한 방법을 알아보자.

`printHeaderUtils(HttpServletRequest request)`
```java
private void printHeaderUtils(HttpServletRequest request) {
        System.out.println("--- Header 편의 조회 start ---");
        System.out.println("[Host 편의 조회]");
        System.out.println("request.getServerName() = " + request.getServerName()); //Host 헤더
        System.out.println("request.getServerPort() = " + request.getServerPort()); //Host 헤더
        System.out.println();

        System.out.println("[Accept-Language 편의 조회]");
        request.getLocales().asIterator()
                .forEachRemaining(locale -> System.out.println("locale = " + locale));
        System.out.println("request.getLocale() = " + request.getLocale());
        System.out.println();

        System.out.println("[cookie 편의 조회]");
        if (request.getCookies() != null) {
            for (Cookie cookie : request.getCookies()) {
                System.out.println(cookie.getName() + ": " + cookie.getValue());
            }
        }
        System.out.println();

        System.out.println("[Content 편의 조회]");
        System.out.println("request.getContentType() = " + request.getContentType());
        System.out.println("request.getContentLength() = " + request.getContentLength());
        System.out.println("request.getCharacterEncoding() = " + request.getCharacterEncoding());

        System.out.println("--- Header 편의 조회 end ---");
        System.out.println();
    }
```

![](https://i.imgur.com/1XIr9kH.png){: .align-center}


결과로 바로 보자 먼저 Host 편의 조회로 서버 이름과, 포트를 알아낼 수 있고

그다음 Accept-Language 편의 조회는 서버의 지역 정보를 출력할 수 있다.

쿠키도 다음과 같이 `request.getCookies()` 로 쿠키들을 가져와서 for문으로 돌려서 찾아 내는 방법이 있다.

마지막으로 Content 편의 조회는 Body에 실린 데이터의 정보를 알 수 있다.
이를 위해선 Get 방식이 아니라 Post 방식으로 데이터를 보내야 테스트를 해볼 수 있다.


### 4. 기타 정보

마지막으로 기타적인 IP나 그런걸 가져오는 방법을 알아보자.

`printEtc(HttpServletRequest request)`
```java
private void printEtc(HttpServletRequest request) {  
    System.out.println("--- 기타 조회 start ---");  
  
    System.out.println("[Remote 정보]");  
    System.out.println("request.getRemoteHost() = " + request.getRemoteHost()); //  
    System.out.println("request.getRemoteAddr() = " + request.getRemoteAddr()); //  
    System.out.println("request.getRemotePort() = " + request.getRemotePort()); //  
    System.out.println();  
  
    System.out.println("[Local 정보]");  
    System.out.println("request.getLocalName() = " + request.getLocalName()); //  
    System.out.println("request.getLocalAddr() = " + request.getLocalAddr()); //  
    System.out.println("request.getLocalPort() = " + request.getLocalPort()); //  
  
    System.out.println("--- 기타 조회 end ---");  
    System.out.println();  
}
```

클라이언트와 서버의 IP를 가져오는 메서드들 이다.

![](https://i.imgur.com/Kuvvubf.png){: .align-center}

remote는 클라이언트, Local은 서버 정보로 보자. 정확하게 가져오려면 여러 가지를 추가해야 하겠지만..


> **참고** <br>로컬에서 테스트 하면 IPv6 정보가 나오는데, IPv4 정보가 보고 싶으면 다음 옵션을 VM options 에 넣어주면 된다. <br>`-Djava.net.preferIpv4Stack=true`







