---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (47) 스프링 부트 - API 오류 처리 - HandlerExceptionResolver
aliases: 
tags:
  - spring
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


## API 예외 처리 - HandlerExceptionResolver 시작

>**목표** : 예외가 발생해서 서블릿을 넘어 WAS까지 예외가 전달되면 HTTP 상태 코드가 500으로 처리된다. <br>발생하는 예외에 따라서 400, 404 등등 다른 상태 코드로 처리하고 싶다.<br><br>오류 메시지, 형식 등을 API 마다 다르게 처리하고 싶다.

### 상태 코드 변환

예를 들어서 `IllegalArgumentException`을 처리하지 못해서 컨트롤러 밖으로 넘어가는 일이 발생하면 HTTP 상태 코드를 400으로 처리하고 싶다. 어떻게 해야 할까?

```java
@GetMapping("/api/members/{id}")  
public MemberDto getMember(@PathVariable("id") String id) {  
    if (id.equals("ex")) {  
        throw new RuntimeException("잘못된 사용자");  
    }  
  
    if (id.equals("bad")) {  
        throw new IllegalArgumentException("잘못된 입력 값");  
    }  
  
    return new MemberDto(id, "hello " +id);  
}
```

다음과 같이 `IllegalArgumentException`이 발생하도록 조건을 추가했다.

![](https://i.imgur.com/OcHIONq.png){: .align-center}

하지만 컨트롤러 밖으로 그냥 예외가 던져 졌기 때문에 **WAS는 그냥 `500` 처리해버린다.**

### HandlerExceptionResolver

스프링 MVC는 컨트롤러(핸들러) 밖으로 예외가 던져진 경우 예외를 해결하고, 동작을 새로 정의할 수 있는 방법을 제공한다. 

컨트롤러 밖으로 던져진 예외를 해결하고, 동작 방식을 변경하고 싶으면 `HandlerExceptionResolver`를 사용하면 된다. 

줄여서 `ExceptionResolver`라 한다.

![](https://i.imgur.com/oolmENh.png){: .align-center}

#### HandlerExceptionResolver - 인터페이스 
```java
public interface HandlerExceptionResolver {  
    @Nullable  
    ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex);  
}
```

- `handler` : 핸들러(컨트롤러) 정보
- `Exception ex` : 핸들러(컨트롤러)에서 발생한 예외


### MyHandlerExceptionResolver 구현

이제 만들어 보자.

```java
@Slf4j  
public class MyHandlerExceptionResolver implements HandlerExceptionResolver {  
    @Override  
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {  
        try {  
            if (ex instanceof IllegalArgumentException) {  
                log.info("IllegalArgumentException resolver to 400");  
                response.sendError(HttpServletResponse.SC_BAD_REQUEST, ex.getMessage());  
                return new ModelAndView();  
            }  
        } catch (IOException e) {  
            log.error("resolver ex", e);  
        }  
  
        return null;  
    }  
}
```

- `ExceptionHandler`가 `ModelAndView`를 반환하는 이유는 try, catch 하듯이, `Exception`을 처리해서 정상 흐름 처럼 변경하는 것이 목적이다.

여기 서는 `IllegalArgumentException`이 발생하면 `response.sendError(400)`을 호출해서 HTTP 상태 코드를 400으로 바꾸고 빈 `ModelAndView`를 반환 한다.

#### 반환 값에 따른 동작 방식

`HandlerExceptionResolver`의 반환 값에 따른 `DispatcherServlet`의 동작 방식은 다음과 같다.

- **빈 ModelAndView** : `new ModelAndView()` 빈 모델앤뷰를 반환하면 뷰를 렌더링 하지 않고, 정상 흐로 서블릿이 리턴된다.
- **ModelAndView 지정** : `ModelAndView`에 View, Model등의 정보를 지정해서 반환하면 뷰를 렌더링 한다.
- **null** : `null`을 반환하면, 다음 `ExceptionResolver`를 찾아서 실행한다. 만약 처리할 수 있는 `ExceptionResolver`가 없으면 예외 처리가 안되고, 기존에 발생한 예외를 서블릿 밖으로 던진다.


#### ExceptionResolver 활용

- 예외 상태 코드 변환
	- 예외를 `response.sendError(xxx)`호출로 변경해서 서블릿에서 상태 코드에 따른 오류를 처리하도록 위임
	- 이후 WAS는 서블릿 오류 페이지를 찾아서 내부 호출, 예를 들어서 스프링 부트가 기본으로 설정한 `/error`가 호출 됨.
- 뷰 템플릿 처리
	- `ModelAndView`에 값을 채워서 예외에 따른 새로운 오류 화면 뷰 렌더링 해서 고객에게 제공
- API 응답 처리
	- `response.getWriter().println("hello");`처럼 HTTP 응답 바디에 직접 데이터를 넣어주는 것도 가능하다. 여기에 JSON으로 응답하면 API 응답 처리를 할 수 있다.



이제 만든 `MyHandlerExceptionResolver`를 등록해 보자.

### MyHandlerExceptionResolver 등록


`hello.exception.WebConfig`
```java
@Override  
public void extendHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) { 
    resolvers.add(new MyHandlerExceptionResolver());  
}
```

`WebConfig`에 `extendHandlerExceptionResolvers`를 override 해서 

`resolvers.add(new MyHandlerExceptionResolver())`로 등록하였다.

`WebMvcConfigurer`에서 `configureHandlerExceptionResolvers`랑 `extendHandlerExceptionResolvers`
가 있는데 `extendHandlerExceptionResolvers`를 구현해야 한다.



![](https://i.imgur.com/gN3H4ET.png){: .align-center}

이제 500이 아니라 의도한 대로 400이 나오게 된다.