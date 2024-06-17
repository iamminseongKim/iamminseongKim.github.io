---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (42) 스프링 인터셉터 - ArgumentResolver 활용 로그인 체크
aliases: 
tags:
  - spring
  - login
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-06-18
last_modified_at: 2024-06-18
---

> 인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.


## ArgumentResolver 활용

 [mvc1편 - 요청 매핑 핸들러 어뎁터 구조](https://iamminseongkim.github.io/spring/%EC%8A%A4%ED%94%84%EB%A7%81-MVC-1%ED%8E%B8-%EC%8A%A4%ED%94%84%EB%A7%81-MVC-%EA%B8%B0%EB%B3%B8-%EA%B8%B0%EB%8A%A5-(11)-%EC%9A%94%EC%B2%AD-%EB%A7%A4%ED%95%91-%ED%97%A8%EB%93%A4%EB%9F%AC-%EC%96%B4%EB%8E%81%ED%84%B0-%EA%B5%AC%EC%A1%B0/#argumentresolver)에서 `ArgumentResolver`를 학습했다.

이 기능을 사용해서 로그인 회원을 조금 편리하게 찾아 보자.

### HomeController 추가

```java
@GetMapping("/")  
public String homeLoginV3ArgumentResolver(@Login Member loginMember, Model model) {  
  
    // 세션에 회원 데이터가 없으면 그냥 홈  
    if (loginMember == null) return "home";  
  
    model.addAttribute("member", loginMember);  
    return "loginHome";  
}
```

`@Login`이라는 커스텀 애노테이션이 생겼다. 

이 애노테이션은 직접 만든 `ArgumentResolver`가 동작해서 자동으로 세션에 있는 로그인 회원을 찾아주고, 만약 세션에 없다면 `null`을 반환하도록 개발해 보자.


### @Login 애노테이션 생성

![](https://i.imgur.com/0j0MLUt.png){: .align-center}

![](https://i.imgur.com/c4uu4XS.png){: .align-center}

다음과 같이 패키지 구성했고, Annotation을 만들었다.


`hello.login.web.argumentresolver.Login`
```java
@Target(ElementType.PARAMETER)  
@Retention(RetentionPolicy.RUNTIME)  
public @interface Login {  
}
```

처음 인터페이스를 만들어 봐서 공부 해봤다.

#### 1. @Target(ElementType.PARAMETER)

- 어노테이션이 생성 될 수 있는 위치 지정
- PARAMETER 로 지정하였으므로 메서드에 파라미터로 선언된 객체에서만 사용 가능
- 이 외에도 클래스 선언문에서 쓸 수 있는 TYPE 등이 있음.

#### 2. @interface

- 이 파일을 어노테이션 클래스로 지정
- `@Login` 애노테이션이 생겼다고 보면 됨.

#### 3. @Retention

- 애노테이션의 라이프 사이클
- 즉 애노테이션이 언제까지 살아 있을지 결정하는 것.
- `SOURCE` : 소스 코드 (.java) (ex. lombok - getter / 컴파일 하면 사라지는 이유!)
- `CLASS` : 클래스 파일(.class) = 바이트 코드 (이거 ) (만약에 내가 애노테이션을 jar 배포해서 애노테이션을 잠재적으로 더 활용 가능하게 하기 위해.. )
- RUNTIME : 런타임 까지 (= 즉 안사라짐)  (ex. @Controller, @Service, @Autowired 등 스프링이 컴포넌트 스캔을 하는 방법(Reflection)이 애노테이션을 찾아서 사용.)


이제 이 애노테이션을 활용할 `LoginMemberArgumentResolver`을 만들어 보자.



### LoginMemberArgumentResolver 구현

`hello.login.web.argumentresolver.LoginMemberArgumentResolver`
```java
@Slf4j  
public class LoginMemberArgumentResolver implements HandlerMethodArgumentResolver {  
  
    @Override  
    public boolean supportsParameter(MethodParameter parameter) {  
  
        log.info("supportsParameter 실행");  
  
        boolean hasLoginAnnotation = parameter.hasParameterAnnotation(Login.class);  
        boolean hasMemberType = Member.class.isAssignableFrom(parameter.getParameterType());  
  
        return hasLoginAnnotation && hasMemberType;  
    }  
  
    @Override  
    public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer, NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {  
  
        log.info("resolveArgument 실행");  
  
        HttpServletRequest request = (HttpServletRequest) webRequest.getNativeRequest();  
        HttpSession session = request.getSession(false);  
        if (session == null) {  
            return null;  
        }  
        return session.getAttribute(SessionConst.LOGIN_MEMBER);  
    }  
}
```


- `supportsParameter()` : 
	- `parameter.hasParameterAnnotation(Login.class)` : 파라미터가 Login 애노테이션 인지 확인
	- `Member.class.isAssignableFrom(parameter.getParameterType())` : 이게 Member 객체인지 확인 
	- 두 개가 다 `true`여야 다음 `resolveArgument()`로 넘어감
- `resolveArgument()`
	- 세션이 없으면 비회원 이기 때문에 `null`을 반환
	- 멤버 관련 세션이 있으면 회원이기 때문에 멤버 객체를 반환


### LoginMemberArgumentResolver 등록

등록은 `WebConfig`에서 하겠다.

```java
@Override  
public void addArgumentResolvers(List<HandlerMethodArgumentResolver> resolvers) {  
    resolvers.add(new LoginMemberArgumentResolver());  
}
```

`addArgumentResolvers`을 오버라이드 해서 

`resolvers.add(new LoginMemberArgumentResolver())` 해주면 등록이 된다.


이제 결과를 보자.


![](https://i.imgur.com/JP1jGcb.png){: .align-center}

최초 들어갈 때 `LoginMemberArgumentResolver` 가 판단해서 지금 멤버 정보가 없으므로 `null`을 반환했고,

```java
@GetMapping("/")  
public String homeLoginV3ArgumentResolver(@Login Member loginMember, Model model) {  
  
    // 세션에 회원 데이터가 없으면 그냥 홈  
    if (loginMember == null) return "home";  
  
    model.addAttribute("member", loginMember);  
    return "loginHome";  
}
```

`loginMember`가 null이기 때문에 비회원 홈으로 간다.


![](https://i.imgur.com/RXPRi8P.png){: .align-center}

그 다음 로그인 했을 때는, 이제 `resolveArgument`에서 멤버 객체 세션에 있기 때문에 그걸 반환했다.

그래서 회원 페이지로 이동했다.

참고로 `supportsParameter`는 캐시 기능이 있어서 `@Login` 검증을 스킵 했다.



