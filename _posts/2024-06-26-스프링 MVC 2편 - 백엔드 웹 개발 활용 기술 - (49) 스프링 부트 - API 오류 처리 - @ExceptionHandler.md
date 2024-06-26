---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (49) 스프링 부트 - API 오류 처리 - @ExceptionHandler
aliases: 
tags:
  - spring
  - exception
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-06-26
last_modified_at: 2024-06-26
---

> 인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.

## API 예외 처리 - @ExceptionHandler


### HTML 화면 오류 vs API 오류

- HTML 화면 제공 시 오류 발생 → `BasicErrorController` 
- API → `???`

API는 각 시스템마다 **응답의 모양도 다르고, 스펙도 모두 다르다**.

예외 상황에 단순히 오류 화면을 보여주는 것이 아니라, 예외에 따라서 각각 다른 데이터를 출력해야 할 수도 있다.

그리고 같은 예외라고 해도 어떤 컨트롤러에서 발생했는가 에 따라서 다른 예외 응답을 내려주어야 할 수도 있다.


### API 예외 처리의 어려운 점

- `HandlerExceptionResolver`는 `ModelAndView`를 리턴 해야 했다. 이건 API는 사실 필요 없다.
- API 응답을 위해서 `HttpServletResponse`에 직접 응답 데이터를 만들어 넣어 줘야 했다. 마치 스프링 안 쓰고 서블릿을 쓴 것 처럼
- 특정 컨트롤러에서만 발생하는 예외를 별도로 처리하기 어렵다. 
	- 회원 처리 컨트롤러에서 발생한 `RuntimeException`과 <br>상품 관리 컨트롤러에서 발생한 `RuntimeException` <br>서로 다르게 처리해야 하는데 어떻게 처리할까?


### @ExceptionHandler

스프링은 API 예외 처리 문제를 해결하기 위해 `@ExceptionHandler`라는 애노테이션을 제공한다.

매우 편리한 예외 처리 기능을 제공하는데, 이것이 바로 `ExceptionHandlerExceptionResolver`이다.

`ExceptionHandlerExceptionResolver` 를 기본으로 제공하고, `ExceptionResolver`중에 **우선 순위도 가장 높다.**


### 예제 준비비

먼저 에러 데이터를 바인딩할 `ErrorResult.java`를 생성

![](https://i.imgur.com/bEBHhoq.png){: .align-center}


```java
@Data  
@AllArgsConstructor  
public class ErrorResult {  
    private String code;  
    private String message;  
}
```

이제 컨트롤러를 하나 만들어 보자.

`hello.exception.api.ApiExceptionV2Controller`

```java
@Slf4j  
@RestController  
public class ApiExceptionV2Controller {  

    @GetMapping("/api2/members/{id}")  
    public MemberDto getMember(@PathVariable("id") String id) {  
        if (id.equals("ex")) {  
            throw new RuntimeException("잘못된 사용자");  
        }  
        if (id.equals("bad")) {  
            throw new IllegalArgumentException("잘못된 입력 값");  
        }  
        if (id.equals("user-ex")) {  
            throw new UserException("사용자 오류");  
        }  
        return new MemberDto(id, "hello " +id);  
    }  
  
    @Data  
    @AllArgsConstructor    
    static class MemberDto {  
        private String memberId;  
        private String name;  
    }  
  
}
```

간단하게 예외를 발생 시키는 컨트롤러를 만들었다.

이제 `@ExceptionHandler`처리 방법을 알아보자.


### @ExceptionHandler 예외 처리 사용

`@ExceptionHandler`애노테이션을 선언하고, 해당 컨트롤러에서 처리하고 싶은 예외를 지정해주면 된다.

해당 컨트롤러에서 예외가 발생하면 이 메서드가 호출된다. 

참고로 지정한 예외 또는 그 예외의 자식 클래스는 모두 잡을 수 있다.


컨트롤러 위에 다음과 같이 추가하자.

```java
@ExceptionHandler(IllegalArgumentException.class)  
public ErrorResult illegalExHandler(IllegalArgumentException e) {  
    log.error("[exceptionHandle] ex", e);  
    return new ErrorResult("BAD", e.getMessage());  
}
```

이 메서드는 `IllegalArgumentException`가 컨트롤러 클래스 내에서 발생하면 실행 된다.

`ErrorResult`를 반환한다.

![](https://i.imgur.com/qJ0d7ha.png){: .align-center}

내가 원하던 code, message가 왔다. 하지만 `200 OK`가 응답 됐다.

스프링 입장에서 정상 처리한 건 맞으니깐..

그럼 다음과 같이 `@ResponseStatus`를 추가하면 된다.

```java
@ResponseStatus(HttpStatus.BAD_REQUEST)  
@ExceptionHandler(IllegalArgumentException.class)  
public ErrorResult illegalExHandler(IllegalArgumentException e) {  
    log.error("[exceptionHandle] ex", e);  
    return new ErrorResult("BAD", e.getMessage());  
}
```

![](https://i.imgur.com/D9cXMyf.png){: .align-center}

400 Bad Request가 잘 나온다.


```java
@ExceptionHandler  
public ResponseEntity<ErrorResult> userExHandler(UserException e) {  
    log.error("[exceptionHandle] ex", e);  
    ErrorResult errorResult = new ErrorResult("USER-EX", e.getMessage());  
    return new ResponseEntity<>(errorResult, HttpStatus.BAD_REQUEST);  
}  
  
@ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR) // 500 에러  
@ExceptionHandler  
public ErrorResult exHandler(Exception e) {  
    log.error("[exceptionHandle] ex", e);  
    return new ErrorResult("exception", "내부 오류");  
}
```

`@ExceptionHandler`는 다양한 리턴과 파라미터를 사용할 수 있다.

> 참고<br>[스프링 공식 문서](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-exceptionhandler.html#mvc-ann-exceptionhandler-args)에서 다양한 리턴, 파라미터를 확인해 보자.

다음과 같이 `ResponseEntity<>`를 리턴 하면 메시지와 코드를 한번에 보낼 수도 있다.

#### 실행 흐름

- 컨트롤러를 호출한 결과 `IllegalArgumentException`(예시) 예외가 컨트롤러 밖으로 던져진다.
- 예외가 발생했으므로 `ExceptionResolver`가 작동한다. 
	1. `ExceptionHandlerExceptionResolver` ← 우선순위 1등
	2. `ResponseStatusExceptionResolver`
	3. `DefaultHandlerExceptionResolver` 
- `ExceptionHandlerExceptionResolver`는 해당 컨트롤러에 `IllegalArgumentException`을 처리할 수있는 `@ExceptionHandler`가 있는지 확인한다.
- `illegalExHandler()`를 실행한다. `@RestController`이므로 `illegalExHandler()`도 `@ResponseBody`가 적용된다.
	- HTTP 컨버터가 사용되고, 응답이 JSON으로 반환된다.
- `@ResponseStatus(HttpStatus.BAD_REQUEST)`로 지정했기 때문에 400으로 응답한다.



### @ExceptionHandler - 우선 순위

스프링의 우선 순위는 항상 **자세한 것**이 우선 순위를 가진다.

부모, 자식 클래스가 있다 치면 다음과 같이 처리된다.

```java
@ExceptionHandler(부모예외.class) 
public String 부모예외처리()(부모예외 e) {} 

@ExceptionHandler(자식예외.class) 
public String 자식예외처리()(자식예외 e) {}
```

이렇게 있다면 

`부모예외처리()`는 `자식예외.class`를 처리할 수 있다. 하지만 자세한 `자식예외처리()`가 있기 때문에 `자식예외처리()`에서 처리된다.

`부모예외.class`는 `자식예외처리()`에서 처리할 순 없고, `부모예외처리()`에서 처리할 수 있다.


### 다양한 예외

```java
@ExceptionHandler({AException.class, BException.class}) 
public String ex(Exception e) { 
	log.info("exception e", e); 
}
```

다음과 같이 다양한 예외를 한 번에 처리할 수 있다.


### 예외 생략

`@ExceptionHandler`를 생략할 수 있다. 생략하면 메서드 **파라미터의 예외**가 지정된다.

```java
@ExceptionHandler  
public ResponseEntity<ErrorResult> userExHandler(UserException e) {}
```

이러면 `UserException`이 지정된 것이다.


### 기타

```java
@ExceptionHandler  
public String ex(RuntimeException e) {  
    return "error/500";  
}
```

이런 식으로 뷰 페이지를 띄워 줄 수도 있다.

단지 여기 서는 `@RestController`가 있기 때문에  그냥 문자열 그대로 나가니깐

`@ResponseBody`에 주의하자.


