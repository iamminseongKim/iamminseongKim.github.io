---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (48) 스프링 부트 - API 오류 처리 - 스프링이 제공하는 ExceptionResolver
aliases: 
tags:
  - spring
  - exception
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-06-25
last_modified_at: 2024-06-26
---

> 인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.


## API 예외 처리 - 스프링이 제공하는 ExceptionResolver 1

스프링 부트가 기본으로 제공하는 `ExceptionResolver`는 다음과 같다.

`HandlerExceptionResolverComposite`에 다음 순서대로 등록
1. `ExceptionHandlerExceptionResolver`
2. `ResponseStatusExceptionResolver`
3. `DefaultHandlerExceptionResolver` → 우선 순위 가장 낮음

**ExceptionHandlerExceptionResolver**

`@ExceptionHandler` 을 처리한다. API 예외 처리는 대부분 이 기능으로 해결한다. 

매우 중요한 기능이기 때문에 따로 정리


**ResponseStatusExceptionResolver**

HTTP 상태 코드를 지정해준다.

예) `@ResponseStatus(value = HttpStatus.NOT_FOUND)`

**DefaultHandlerExceptionResolver**

스프링 내부 기본 예외를 처리한다.

### ResponseStatusExceptionResolver

`ResponseStatusExceptionResolver`는 예외에 따라서 HTTP 상태 코드를 지정해주는 역할을 한다.

다음 두 가지 경우를 처리한다.
- `@ResponseStatus` 가 달려 있는 예외
- `ResponseStatusException` 예외


`hello.exception.exception.BadRequestException`
```java
@ResponseStatus(code = HttpStatus.BAD_REQUEST, reason = "잘못된 요청 오류")  
public class BadRequestException extends RuntimeException {  
}
```

다음과 같이 Exception 하나를 만들었다.


```java
@GetMapping("/api/response-status-ex1")  
public String responseStatusEx1() {  
    throw new BadRequestException();  
}
```

컨트롤러 하나 만들어서 호출 해보자.

![](https://i.imgur.com/Fwfa18L.png){: .align-center}

400 에러가 잘 나온다.

`BadRequestException`예외가 컨트롤러 밖으로 넘어가면 `ResponseStatusExceptionResolver`예외가 해당 애노테이션을 확인해서 오류 코드를 `HttpStatus.BAD_REQUEST(400)`으로 변경하고, 메시지도 남긴다.


```java
protected ModelAndView applyStatusAndReason(int statusCode, @Nullable String reason, HttpServletResponse response) throws IOException {  
    if (!StringUtils.hasLength(reason)) {  
        response.sendError(statusCode);  
    } else {  
        String resolvedReason = this.messageSource != null ? this.messageSource.getMessage(reason, (Object[])null, reason, LocaleContextHolder.getLocale()) : reason;  
        response.sendError(statusCode, resolvedReason);  
    }  
  
    return new ModelAndView();  
}
```

`ResponseStatusExceptionResolver`에 `applyStatusAndReason`를 보면 

결국 마지막에 `response.sendError(statusCode, resolvedReason)`를 호출하는 걸 확인할 수 있다.



```java
@ResponseStatus(code = HttpStatus.BAD_REQUEST, reason = "잘못된 요청 오류")
```

`reason = "잘못된 요청 오류"`  메시지 기능도 있다.

> application.propeties 에 server.error.include-message=always가 되어 있어야 함.

요청을 http://localhost:8080/api/response-status-ex1?message= 으로 해보면

![](https://i.imgur.com/lcaZRP2.png){: .align-center}

다음과 같이 담아지며 messages.properties를 만들어서 따로 관리할 수도 있다.


```java
@ResponseStatus(code = HttpStatus.BAD_REQUEST, reason = "error.bad")  
public class BadRequestException extends RuntimeException {  
}
```

`messages.properties`
```
error.bad=잘못된 요청입니다. 메시지 사용
```

다음과 같이 세팅해 놓으면 

![](https://i.imgur.com/WGJ1aIG.png){: .align-center}

지정 해 놓은 메시지가 출력 되는 것을 확인 가능.


### ResponseStatusException

`@ResponseStatus`는 개발자가 직접 변경할 수 없는 예외에는 적용할 수 없다. (애노테이션을 직접 넣어야 하는데, 내가 코드를 수정할 수 없는 라이브러리의 예외 코드 같은 곳에는 적용할 수 없다.)

추가로 애노테이션을 사용하기 때문에 조건에 따라 동적으로 변경하는 것도 어렵다. 

이때는 `ResponseStatusException`를 사용하면 된다.


```java
@GetMapping("/api/response-status-ex2")  
public String responseStatusEx2() {  
    throw new ResponseStatusException(HttpStatus.NOT_FOUND, "error.bad", new IllegalArgumentException());  
}
```

다음과 같이 컨트롤러에서 `ResponseStatusException`를 호출 했는데, 여기서 바로 status랑 메시지, 오류를 넣어줄 수 있다.


![](https://i.imgur.com/C2vx96G.png){: .align-center}


status 가 404로 잘 나오는 걸 확인할 수 있다.


## API 예외 처리 - 스프링이 제공하는 ExceptionResolver2

이번에는 `DefaultHandlerExceptionResolver`를 살펴보자.

`DefaultHandlerExceptionResolver`는 스프링 **내부**에서 발생하는 스프링 예외를 해결한다.

대표적으로 파라미터 바인딩 시점에 타입이 맞지 않으면 내부에서 `TypeMismatchException`이 발생하는데, 이 경우 예외가 발생했기 때문에 그냥 두면 서블릿 컨테이너까지 올라가고, 결과적으로 500 오류가 발생한다.

그런데 파라미터 바인딩은 대부분 클라이언트가 잘못 호출해서 발생한 오류이기 때문에 400 오류를 리턴하는게 맞다.



```java
@GetMapping("/api/default-handler-ex")  
public String defaultException(@RequestParam(name = "data") Integer data) {  
    return "ok";  
}
```

다음과 같이 컨트롤러를 만들었고, 숫자를 받도록 만든 다음에 문자를 넘겨 봤다.


![](https://i.imgur.com/iCYv1mf.png)
{: .align-center}


원래라면 서블릿 타고 500 에러가 나와야 하는데, 400 Bad Request가 나온다.

`DefaultHandlerExceptionResolver`가 처리해 준 것 이다.


![](https://i.imgur.com/pHUKrch.png){: .align-center}

로그에도 친절히 나와있다.


![](https://i.imgur.com/pxZ2zPI.png){: .align-center}

DefaultHandlerExceptionResolver 코드를 보면은 엄청 많은 오류를 체크 하고 있고 

`TypeMismatchException`도 있는 걸 확인 할 수 있다.


