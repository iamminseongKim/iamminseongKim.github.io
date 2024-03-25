---
title: 스프링 MVC - 1편 - MVC 프레임워크 만들기 - Model 추가 (3)
aliases: 
tags:
  - mvc패턴
  - spring
  - java
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-03-25
last_modified_at: 2024-03-25
---
>  인프런 스프링 MVC 1편 - 백엔드 웹 개발 핵심 기술편을 학습하고 정리한 내용 입니다.

**서블릿 종속성 제거**
컨트롤러 입장에서 HttpServletRequest, HttpServletResponse 가 꼭 필요할까?
요청 파라미터 정보는 자바의 Map으로 대신 넘기도록 하면 지금 구조에서는 컨트롤러가 서블릿 기술을 몰라도 동작할 수 있다. 그리고 request 객체를 Model로 사용하는 대신에 별도의 Model 객체를 만들어서 반환하면 된다.

우리가 구현하는 컨트롤러가 서블릿 기술을 전혀 사용하지 않도록 변경해 보자.
이렇게 하면 구현 코드도 간단하고, 테스트 코드도 작성이 쉽다.

**뷰 이름 중복 제거**
컨트롤러에서 지정하는 뷰 이름에 중복이 있는 것을 확인 할 수 있다. 

컨트롤러는 **뷰의 논리 이름**을 반환하고, 실제 물리 위치의 이름은 프론트 컨트롤러에서 처리하도록 단순화 하자.

이렇게 해두면 향후 뷰의 폴더 위치가 함께 이동해도 프론트 컨트롤러만 고치면 된다.

ex) 
- `/WEB-INF/views/new-form.jsp`  →  **new-form**
- `/WEB-INF/views/save-result.jsp`  →  **save-result**
- `/WEB-INF/views/mebmers.jsp`  →  **members**

![](https://i.imgur.com/26xLkHT.png){: .align-center}

**ModelView**
지금까지 컨트롤러에서 서블릿에 종속적인 HttpServletRequest를 사용했다. 
그리고 Model도 `request.setAttribute()`를 통해 데이터를 저장하고 뷰에 전달했다.
서블릿의 종속성을 제거하기 위해 Model을 직접 만들고, 추가로 View 이름까지 전달하는 객체를 만들어 보자.
(이번 버전에서는 컨트롤러에서 HttpServletRequest를 사용할 수 없다. 따라서 직접 `request.setAttribute()` 를 호출할 수 도 없다. 따라서 Model이 별도로 필요하다.)

참고로 `ModelView`객체는 다른 버전에서도 사용하므로 패키지를 frontcontroller에 둔다.

`hello.servlet.web.frontcontroller.ModelView`
```java
@Getter @Setter  
public class ModelView {  
    private String viewName;  
    private Map<String, Object> model = new HashMap<>();  
  
    public ModelView(String viewName) {  
        this.viewName = viewName;  
    }  
  
}
```

이 객체에선 view이름을 담아줄 `viewName` 그 다음에 request 에 객체를 넘겨 줄 map 자료구조를 `model`이라는 이름으로 만들었다.

뷰의 이름과 뷰를 렌더링할 때 필요한 model 객체를 가지고 있다. model은 단순히 map으로 되어 있으므로 컨트롤러 에서 뷰에 필요한 데이터를 key, value로 넣어주면 된다.

이제 컨트롤러 인터페이스를 만들어 보자.
`hello.servlet.web.frontcontroller.v3.ControllerV3`
```java
public interface ControllerV3 {  
    ModelView process(Map<String, String> paramMap);  
}
```
request, response가 파라미터에서 사라졌고, 리턴 타입이 ModelView 이다.

이제 3가지 컨트롤러 폼, 등록, 리스트 컨트롤러를 만들어 보자.

1. `hello.servlet.web.frontcontroller.v3.controller.MemberFormControllerV3` - 회원 등록 폼
```java
public class MemberFormControllerV3 implements ControllerV3 {  
  
    @Override  
    public ModelView process(Map<String, String> paramMap) {  
        return new ModelView("new-form");  
    }  
}
```

`ModelView`를 생성할 때 `new-form`이라는 view의 **논리적** 이름을 지정한다. 

실제 물리적인 이름은 프론트 컨트롤러에서 공통으로 처리한다.

2. `hello.servlet.web.frontcontroller.v3.controller.MemberSaveControllerV3` - 회원 저장
```java
public class MemberSaveControllerV3 implements ControllerV3 {  
    private MemberRepository memberRepository = MemberRepository.getInstance();  
    @Override  
    public ModelView process(Map<String, String> paramMap) {  
        String username = paramMap.get("username");  
        int age = Integer.parseInt(paramMap.get("age"));  
  
        Member member = new Member(username, age);  
        memberRepository.save(member);  
  
        ModelView mv = new ModelView("save-result");  
        mv.getModel().put("member", member);  
        return mv;  
    }  
}
```


```java
String username = paramMap.get("username");
```
이제 request는 없다. 그저 map에서 데이터를 가져오면 된다.

```java
ModelView mv = new ModelView("save-result");  
mv.getModel().put("member", member);
```

그리고 `request.setAttribute()`도 없기 때문에 객체를 넘겨주기 위해 만든 `ModelView`의 맵을 가져와서

`mv.getModel().put("member", member);`  데이터를 넣어준다.

3. `hello.servlet.web.frontcontroller.v3.controller.MemberListControllerV3` - 회원 목록
```java
public class MemberListControllerV3 implements ControllerV3 {  
  
    private MemberRepository memberRepository = MemberRepository.getInstance();  
  
    @Override  
    public ModelView process(Map<String, String> paramMap) {  
        List<Member> members = memberRepository.findAll();  
        ModelView mv = new ModelView("members");  
        mv.getModel().put("members", members);  
        return mv;  
    }  
}
```

```java
List<Member> members = memberRepository.findAll();  
ModelView mv = new ModelView("members");  
mv.getModel().put("members", members);  
```
여기서도 똑같이 멤버 리스트를 받아서 `ModelView`의 맵을 가져와서 넣어주고 리턴해버린다.

이제 프론트 컨트롤러를 만들어 보자.

`hello.servlet.web.frontcontroller.v3.FrontControllerServletV3`
```java
@WebServlet(name = "frontControllerServletV3", urlPatterns = "/front-controller/v3/*")  
public class FrontControllerServletV3 extends HttpServlet {  
  
    private Map<String, ControllerV3> controllerMap = new HashMap<>();  
  
    public FrontControllerServletV3() {  
        controllerMap.put("/front-controller/v3/members/new-form", new MemberFormControllerV3());  
        controllerMap.put("/front-controller/v3/members/save", new MemberSaveControllerV3());  
        controllerMap.put("/front-controller/v3/members", new MemberListControllerV3());  
    }  
  
    @Override  
    protected void service(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {  
        String requestURI = request.getRequestURI();  
  
        ControllerV3 controller = controllerMap.get(requestURI);  
        if (controller == null) {  
            response.setStatus(HttpServletResponse.SC_NOT_FOUND);  
            return;  
        }  
  
        // paramMap  
        Map<String, String> paramMap = createParamMap(request);  
  
        ModelView mv = controller.process(paramMap);  
  
        String viewName = mv.getViewName();// 논리이름 new-form        
        MyView view = viewResolver(viewName);  
  
        view.render(mv.getModel(), request, response);  
    }  
  
    private MyView viewResolver(String viewName) {  
        return new MyView("/WEB-INF/views/" + viewName + ".jsp");  
    }  
  
    private Map<String, String> createParamMap(HttpServletRequest request) {  
        Map<String, String> paramMap = new HashMap<>();  
  
        request.getParameterNames().asIterator()  
                .forEachRemaining(paramName->paramMap.put(paramName, request.getParameter(paramName)));  
        return paramMap;  
    }  
  
  
}
```

코드가 좀 길고 많아졌다. 천천히 보자.

```java
@WebServlet(name = "frontControllerServletV3", urlPatterns = "/front-controller/v3/*")
```
`/front-controller/v3/` 뒤에 아무거나 붙으면 이 컨트롤러를 타겠다는 의미다.

```java
private Map<String, ControllerV3> controllerMap = new HashMap<>();  
  
public FrontControllerServletV3() {  
	controllerMap.put("/front-controller/v3/members/new-form", new MemberFormControllerV3());  
	controllerMap.put("/front-controller/v3/members/save", new MemberSaveControllerV3());  
	controllerMap.put("/front-controller/v3/members", new MemberListControllerV3());  
}
```

미리 url과 컨트롤러를 매핑 해놓고 자료구조에 담아놓는다.


```java
String requestURI = request.getRequestURI();  
  
ControllerV3 controller = controllerMap.get(requestURI);  
if (controller == null) {  
	response.setStatus(HttpServletResponse.SC_NOT_FOUND);  
	return;  
}  
```

클라이언트가 들어온 URI를 가져와서 미리 담아 놓은 자료구조에서 꺼내와서 `ControllerV3` 객체에 담는다. *(부모는 자식을 담을 수 있으니깐.)*

그리고 없는 url이 들어왔으면 404 처리

```java
// paramMap  
Map<String, String> paramMap = createParamMap(request);  

ModelView mv = controller.process(paramMap);  

String viewName = mv.getViewName();// 논리이름 new-form        
MyView view = viewResolver(viewName);  

view.render(mv.getModel(), request, response);
```

그리고 이 부분이다.

먼저 맨 윗 줄에선 url을 타고 넘어온 request 값 들을 Map 자료구조로 넘겨주는 역할을 할 것이다.
```java
private Map<String, String> createParamMap(HttpServletRequest request) {  
    Map<String, String> paramMap = new HashMap<>();  
  
    request.getParameterNames().asIterator()  
            .forEachRemaining(paramName->paramMap.put(paramName, request.getParameter(paramName)));  
    return paramMap;  
}
```

다음과 같이 `request.getParameterNames()`으로 모든 request 파라미터를 가져와서 

`.asIterator()` 로 for문을 돌리고  
`.forEachRemaining()` 각각 `paramName` (request의 키 값)을 꺼내서 로직을 수행하는데

```java
paramMap.put(paramName, request.getParameter(paramName))
```
request의 key, value를 map 자료구조에 넣어서 리턴시킨다.

그 다음 로직은 
```java
String viewName = mv.getViewName();// 논리이름 new-form
MyView view = viewResolver(viewName);
```

이건데 우리는 컨트롤러에서 뷰의 이름을 **논리 이름**만 가져왔기 때문에 이를 물리 이름으로 바꿔준다.

```java
private MyView viewResolver(String viewName) {  
    return new MyView("/WEB-INF/views/" + viewName + ".jsp");  
}
```

절대 경로를 추가해서 다시 리턴 해준다.

이제 마지막으로

```java
view.render(mv.getModel(), request, response);
```
view를 그리러 가주면 된다.

그런데 이제 컨트롤러에서 모든 로직을 수행하고 request에 값을 넣는 과정이 없기 때문에

`mv.getModel()`을 render메서드에 추가 해줘야 한다.

MyView 클래스에 메서드를 새로 만들어 줘야 한다.

`hello.servlet.web.frontcontroller.MyView`
```java
public void render(Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException {  
  
    modelToRequestAttribute(model, request);  
    RequestDispatcher dispatcher = request.getRequestDispatcher(viewPath);  
    dispatcher.forward(request, response);  
}
```

자 이제 마지막으로 map -> request.setAttribute 해주는 메서드를 만들어서 넘어온 map을 처리해 주자

```java
private void modelToRequestAttribute(Map<String, Object> model, HttpServletRequest request) {  
    // model에 있는 값을 다 request에 setAttribute 해준다.  
    model.forEach(request::setAttribute);  
}
```

자 이게 끝이다. 좀 로직이 많이 복잡해 졌는데, 

![](https://i.imgur.com/26xLkHT.png){: .align-center}

이 구조를 잘 생각하고, 객체를 생각해 보면 이해할 수 있을 것이다.

이제 잘 되는지 보자.

![](https://i.imgur.com/bXgtNtN.png){: .align-center}


![](https://i.imgur.com/HVlYKLQ.png){: .align-center}

![](https://i.imgur.com/kZQP5W1.png){: .align-center}

자 이렇게 잘 되는 걸 확인할 수 있다.

이로써 우리는 view이름도 절대 경로 -> 상대 경로로 바꿀 수 있었고
request, response를 반복적 으로 사용 하지 않아도 되도록 수정할 수 있었다.



