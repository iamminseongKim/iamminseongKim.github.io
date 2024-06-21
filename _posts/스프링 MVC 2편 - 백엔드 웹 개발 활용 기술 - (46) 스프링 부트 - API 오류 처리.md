---
title: 2024-06-24-스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (46) 스프링 부트 - API 오류 처리
aliases: 
tags:
  - exception
  - api
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-06-24
last_modified_at: 2024-06-24
---

> 인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.

## API 예외 처리 - 시작

**목표**
API 예외 처리는 어떻게 해야 할까?

HTML 페이지의 경우 4xx, 5xx 같은 오류 페이지를 띄워주면 대부분 ok

API의 경우는 생각할 내용이 더 많다.

API는 고객이 사용하지 않고 기업-기업, 서버-모바일 등 개발자끼리 통신하기 위한 약속

API는 각 오류 상황에 맞는 오류 응답 스펙을 정하고, JSON으로 데이터를 내려주어야 한다.

먼저 서블릿 오류 페이지 방식으로 알아보자. 


### 구현

일단 프로젝트는 [8. 예외 처리와 오류 페이지](https://iamminseongkim.github.io/spring/%EC%8A%A4%ED%94%84%EB%A7%81-MVC-2%ED%8E%B8-%EB%B0%B1%EC%97%94%EB%93%9C-%EC%9B%B9-%EA%B0%9C%EB%B0%9C-%ED%99%9C%EC%9A%A9-%EA%B8%B0%EC%88%A0-(43)-%EC%98%88%EC%99%B8-%EC%B2%98%EB%A6%AC%EC%99%80-%EC%98%A4%EB%A5%98-%ED%8E%98%EC%9D%B4%EC%A7%80/)에서 사용하던 프로젝트 그대로 사용함.

먼저 서블릿 예외를 처리하기 위해 기존에 사용하던 `WebServerCustomizer`에 `@Component` 를 주석 해제

그리고 Api 컨트롤러를 하나 만들었다.

![](https://i.imgur.com/RW5BQzN.png){: .align-center}


```java
@RestController  
public class ApiExceptionController {  
  
    @GetMapping("/api/members/{id}")  
    public MemberDto getMember(@PathVariable("id") String id) {  
        if (id.equals("ex")) {  
            throw new RuntimeException("잘못된 사용자");  
        }  
  
        return new MemberDto(id, "hello " +id);  
    }  
  
    @Data  
    @AllArgsConstructor    static class MemberDto {  
        private String memberId;  
        private String name;  
    }  
}
```

자 간단하게 `ex`라는 id 가 들어오면 예외를 터트리는 메서드이다.


postman으로 테스트 해보자.

![](https://i.imgur.com/iJENvFl.png){: .align-center}

정상은 JSON으로 잘 나온다.


![](https://i.imgur.com/kTcJLRO.png){: .align-center}

저번 오류 페이지 만들 때 등록한 **화면이 출력** 돼버렸다. 

이것은 기대한 바가 아니다. 클라이언트는 정상 요청이던, 오류 요청이던 JSON이 반환 되기를 기대한다.

웹 브라우저가 아닌 이상 HTML을 직접 받아서 할 수 있는 것은 별로 없다.

문제를 해결하려면 오류 페이지 컨트롤러도 JSON 응답을 할 수 있도록 수정해야 한다.


### ErrorPageController - API 응답 추가 

```java

@RequestMapping("/error-page/500")  
public String errorPage500(HttpServletRequest request, HttpServletResponse response) {  
    log.info("error-page 500");  
    printErrorInfo(request);  
    return "error-page/500";  
}  

// api용 추가  
@RequestMapping(value = "/error-page/500", produces = MediaType.APPLICATION_JSON_VALUE) 
public ResponseEntity<Map<String, Object>> errorPage500Api(HttpServletRequest request, HttpServletResponse response) {  
  
    log.info("API errorPage 500");  
  
    Map<String, Object> result = new HashMap<>();  
    Exception ex = (Exception) request.getAttribute(ERROR_EXCEPTION);  
    result.put("status", request.getAttribute(ERROR_STATUS_CODE));  
    result.put("message", ex.getMessage());  
  
    Integer statusCode = (Integer) request.getAttribute(RequestDispatcher.ERROR_STATUS_CODE);  
    return new ResponseEntity<>(result, HttpStatusCode.valueOf(statusCode));  
}

```


```
produces = MediaType.APPLICATION_JSON_VALUE
```

이 뜻은 클라이언트가 요청하는 HTTP Header `Accept`의 값이 `application/json`일 때 해당 메서드가 호출된다는 의미이다.

그래서 일반 요청에 의해 500이 터지면 위에 컨트롤러가 호출 되고, API요청에 의해 500 에러가 발생하면 아래 새로 만든 컨트롤러가 호출 될 것이다.


`ResponseEntity`를 사용해서 응답하기 때문에 [메시지 컨버터](https://iamminseongkim.github.io/spring/%EC%8A%A4%ED%94%84%EB%A7%81-MVC-1%ED%8E%B8-%EC%8A%A4%ED%94%84%EB%A7%81-MVC-%EA%B8%B0%EB%B3%B8-%EA%B8%B0%EB%8A%A5-(10)-HTTP-%EB%A9%94%EC%8B%9C%EC%A7%80-%EC%BB%A8%EB%B2%84%ED%84%B0/)가 동작하면서 클라이언트에 JSON이 반환 된다. (여기서는 `MappingJackson2HttpMessageConverter`가 실행될 것이다.)


![](https://i.imgur.com/Pp46zpM.png){: .align-center}

이제 제대로 json 데이터를 응답 받았다.



## API 예외 처리 - 스프링 부트 기본 오류 처리

이번엔 서블릿이 아니라 스프링 부트가 기본 제공하는 방식을 사용해 보자.

먼저`WebServerCustomizer`에 `@Component` 를 주석 처리 하자.

스프링은 `BasicErrorController`를 지원해 준다. 기본 경로는 `/error`

그래서 바로 포스트맨으로 요청을 보내보자.

![](https://i.imgur.com/Najn7O1.png){: .align-center}

json 양식으로 리턴이 된다.

`BasicErrorController`
```java
@RequestMapping(  
    produces = {"text/html"}  
)  
public ModelAndView errorHtml(HttpServletRequest request, HttpServletResponse response) {  
    HttpStatus status = this.getStatus(request);  
    Map<String, Object> model = Collections.unmodifiableMap(this.getErrorAttributes(request, this.getErrorAttributeOptions(request, MediaType.TEXT_HTML)));  
    response.setStatus(status.value());  
    ModelAndView modelAndView = this.resolveErrorView(request, response, status, model);  
    return modelAndView != null ? modelAndView : new ModelAndView("error", model);  
}  
  
@RequestMapping  
public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {  
    HttpStatus status = this.getStatus(request);  
    if (status == HttpStatus.NO_CONTENT) {  
        return new ResponseEntity(status);  
    } else {  
        Map<String, Object> body = this.getErrorAttributes(request, this.getErrorAttributeOptions(request, MediaType.ALL));  
        return new ResponseEntity(body, status);  
    }  
}
```

보면은 `errorHtml`은 일반 오류 페이지를 띄워주고 그 밑에 `error`메서드는 `ResponseEntity<Map<String, Object>>` 즉 json 응답을 보내주는 것 같다.


### Html 페이지 vs API 오류

`BasicErrorController`를 확장하면 JSON 메시지도 변경할 수 있다. 하지만 API오류는 `@ExceptionHandler`를 제공하는 기능을 사용하는 것이 더 좋다. 

지금은 `BasicErrorController`를 확장해서 JSON 오류 메시지를 변경할 수 있다 정도로만 이해해 두자.

스프링 부트가 제공하는 `BasicErrorController`는 HTML 페이지를 제공하는 경우에는 매우 편리하다. 

그런데 API 오류 처리는 다른 차원의 이야기이다. API 마다, 각각의 컨트롤러나 예외 마다 서로 다른 응답 결과를 출력해야 할 수도 있다.

따라서 `BasicErrorController`는 화면을 처리할 때 사용하고, API 오류 처리는 `@ExceptionHandler`를 사용하자.


