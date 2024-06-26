---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (50) 스프링 부트 - API 오류 처리 - @ControllerAdvice
aliases: 
tags:
  - spring
  - error
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-06-27
last_modified_at: 2024-06-27
---

> 인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.

## API 예외 처리 - @ControllerAdvice

![](https://i.imgur.com/3Qcjolq.png){: .align-center}


`@ExceptionHander`를 사용해서 예외를 처리했지만, 정상 코드와 예외 처리 코드가 컨트롤러 하나에 섞여있다.

`@ControllerAdvice`, `@RestControllerAdvice`를 사용하면 분리할 수 있다.


![](https://i.imgur.com/6BWlXBL.png){: .align-center}

다음과 같은 패키지에 `ExControllerAdvice` 생성

`ApiExceptionV2Controller`에 있는 `@ExceptionHandler` 모두 잘라내기 

![](https://i.imgur.com/Jtq3rj2.png){: .align-center}

`hello.exception.exhandler.advice.ExControllerAdvice`
```java
@Slf4j  
@RestControllerAdvice  
public class ExControllerAdvice {  
  
    @ResponseStatus(HttpStatus.BAD_REQUEST)  
    @ExceptionHandler(IllegalArgumentException.class)  
    public ErrorResult illegalExHandler(IllegalArgumentException e) {  
        log.error("[exceptionHandle] ex", e);  
        return new ErrorResult("BAD", e.getMessage());  
    }  
  
  
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
}
```

`@RestControllerAdvice` 이걸 붙여 준 것 만으로 오류 처리가 된다.

![](https://i.imgur.com/azbPQtc.png){: .align-center}

오류 처리가 잘 된다.

### @ControllerAdvice

- `@ControllerAdvice`는 대상으로 지정한 여러 컨트롤러에 `@ExceptionHandler`, `@InitBinder`기능을 부여해주는 역할을 한다.
- `@ControllerAdvice`에 대상을 지정하지 않으면 모든 컨트롤러에 적용된다. (**글로벌 적용**)
- `@RestControllerAdvice`는 `@ControllerAdvice`와 같고, `@ResponseBody`가 추가되어 있다.
	- `@Controller`, `@RestController`의 차이와 같다.


### @ControllerAdvice 대상 컨트롤러 지정 방법

> `@ControllerAdvice`에 대상을 지정하지 않으면 모든 컨트롤러에 적용된다. (**글로벌 적용**)

[@ControllerAdvice](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-advice.html)에 관한 공식 문서

```java
// Target all Controllers annotated with @RestController
@ControllerAdvice(annotations = RestController.class)
public class ExampleAdvice1 {}

// Target all Controllers within specific packages
@ControllerAdvice("org.example.controllers")
public class ExampleAdvice2 {}

// Target all Controllers assignable to specific classes
@ControllerAdvice(assignableTypes = {ControllerInterface.class, AbstractController.class})
public class ExampleAdvice3 {}
```

공식 문서에 나와 있는 컨트롤러 지정 방법이다.

- `(annotations = RestController.class)` : 특정 컨트롤러를 지정함.
- `("org.example.controllers")` : 특정 패키지를 지정함.
- `(assignableTypes = {ControllerInterface.class, AbstractController.class})` : 특정 클래스를 지정함.
- 지정 안 하면 모든 컨트롤러가 적용 대상!

