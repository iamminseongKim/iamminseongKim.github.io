---
title: 스프링 MVC - 1편 - MVC 프레임워크 만들기 - View 분리 -V2
aliases: 
tags:
  - spring
  - mvc패턴
  - java
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-03-22
last_modified_at: 2024-03-22
---
>  인프런 스프링 MVC 1편 - 백엔드 웹 개발 핵심 기술편을 학습하고 정리한 내용 입니다.

자 이제 

```java
String viewPath = "/WEB-INF/views/save-result.jsp";  
RequestDispatcher dispatcher = request.getRequestDispatcher(viewPath);  
dispatcher.forward(request, response);
```

각 컨트롤러 마다 반복 되던 이 행위를 분리 시켜서 깔끔하게 만들어 보자.

**MyView**
뷰 객체는 이후 다른 버전에서도 함께 사용하므로 패키지 위치를 `frontcontroller`에 두었다.

`hello.servlet.web.frontcontroller.MyView`
```java
public class MyView {  
  
    private String viewPath;  
  
    public MyView(String viewPath) {  
        this.viewPath = viewPath;  
    }  
  
    public void render(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {  
        RequestDispatcher dispatcher = request.getRequestDispatcher(viewPath);  
        dispatcher.forward(request, response);  
    }  
}
```
생성자를 통해 view의 위치를 받게 되고, 

`render()` 이 메서드를 통해 기존 해오던 forward 행위를 공통으로 수행하게 된다.

이제 [MVC 프레임워크 만들기 - V1](https://iamminseongkim.github.io/spring/%EC%8A%A4%ED%94%84%EB%A7%81-MVC-1%ED%8E%B8-MVC-%ED%94%84%EB%A0%88%EC%9E%84%EC%9B%8C%ED%81%AC-%EB%A7%8C%EB%93%A4%EA%B8%B0-%ED%94%84%EB%A1%A0%ED%8A%B8-%EC%BB%A8%ED%8A%B8%EB%A1%A4%EB%9F%AC-%ED%8C%A8%ED%84%B4/) 에서 했던 것 처럼 만들어 보자.

먼저 `ControllerV2` 인터페이스 이다.

`hello.servlet.web.frontcontroller.v2.ControllerV2`
```java
public interface ControllerV2 {  
    MyView process(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException;  
}
```

크게 달라진 건 없지만 `MyView` 객체를 리턴 하도록 만들었다.

이제 컨트롤러 3개 폼, 저장, 리스트를 만들러 가보자.

`hello.servlet.web.frontcontroller.v2.controller.MemberFormControllerV2`
```java
public class MemberFormControllerV2 implements ControllerV2 {  
    @Override  
    public MyView process(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {  
        return new MyView("/WEB-INF/views/new-form.jsp");  
    }  
}
```

코드가 매우 간결해 졌다.
MyView객체를 만들어서 view 위치를 넘겨버리면 끝이다.

`hello.servlet.web.frontcontroller.v2.controller.MemberSaveControllerV2`
```java
public class MemberSaveControllerV2 implements ControllerV2 {  
    private MemberRepository memberRepository = MemberRepository.getInstance();  
    @Override  
    public MyView process(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {  
        String username = request.getParameter("username");  
        int age = Integer.parseInt(request.getParameter("age"));  
  
        Member member = new Member(username, age);  
        memberRepository.save(member);  
  
        // Model에 데이터를 보관한다.  
        request.setAttribute("member", member);  
  
        return new MyView("/WEB-INF/views/save-result.jsp");  
    }  
}
```

저장 로직이 있지만, 매우 간결해 졌다. 

`hello.servlet.web.frontcontroller.v2.controller.MemberListControllerV2`
```java
public class MemberListControllerV2 implements ControllerV2 {  
  
    private MemberRepository memberRepository = MemberRepository.getInstance();  
    @Override  
    public MyView process(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {  
  
        List<Member> members = memberRepository.findAll();  
        request.setAttribute("members", members);  
  
        return new MyView("/WEB-INF/views/members.jsp");  
  
    }  
}
```

마지막으로 리스트 컨트롤러 이다.

이제 프론트 컨트롤러를 만들어 보자.

`hello.servlet.web.frontcontroller.v2.FrontControllerServletV2`
```java
@WebServlet(name = "frontControllerServletV2", urlPatterns = "/front-controller/v2/*")  
public class FrontControllerServletV2 extends HttpServlet {  
  
    private Map<String, ControllerV2> controllerMap = new HashMap<>();  
  
    public FrontControllerServletV2() {  
        controllerMap.put("/front-controller/v2/members/new-form", new MemberFormControllerV2());  
        controllerMap.put("/front-controller/v2/members/save", new MemberSaveControllerV2());  
        controllerMap.put("/front-controller/v2/members", new MemberListControllerV2());  
    }  
  
    @Override  
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {  
        String requestURI = request.getRequestURI();  
  
        ControllerV2 controller = controllerMap.get(requestURI);  
        if (controller == null) {  
            response.setStatus(HttpServletResponse.SC_NOT_FOUND);  
            return;  
        }  
  
        MyView view = controller.process(request, response);  
        view.render(request, response);  
    }  
}
```

자 V1 때와 마찬가지로 Map 자료구조 안에 미리 다 담아 놓고,

`String requestURI = request.getRequestURI();` 로 들어온 값 확인 하고

```java
MyView view = controller.process(request, response);
view.render(request, response);
```


ControllerV2의 반환 타입이 `MyView`이므로 프론트 컨트롤러는 컨트롤러의 호출 결과로 `MyView`를 반환 받는다. 

그리고 `view.render()`를 호출하면 `forward`로직을 수행해서 JSP가 실행된다.


프론트 컨트롤러의 도입으로 `MyView`객체의 `render()`를 호출하는 부분을 모두 일관되게 처리할 수 있게 되었다. 각각의 컨트롤러는 `MyView`객체를 생성만 해서 반환하면 된다.

![](https://i.imgur.com/RzwjEGB.png){: .align-center}

이 구조를 만든 것이다.






