---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (7) 타임리프 기본기능 - 템플릿 조각, 템플릿 레이아웃
aliases: 
tags:
  - thymeleaf
  - spring
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-05-07
last_modified_at: 2024-05-07
---
>  인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.

## 템플릿 조각

웹 페이지를 개발할 때는 공통 영역이 많이 있다. 

예를 들어서 상당 영역이나 하단 영역, 좌측 카테고리 등등 여러 페이지에서 함께 사용하는 영역들이 있다.

이런 부분을 코드를 복사해서 사용한다면 변경 시 여러 페이지를 다 수정해야 하므로 상당히 비효율 적이다. 

타임리프는 이런 문제를 해결하기 위해 템플릿 조각과 레이아웃 기능을 지원한다.


`hello.thymeleaf.basic.TemplateController`
```java
@Controller  
@RequestMapping("/template")  
public class TemplateController {  
  
    @GetMapping("/fragment")  
    public String template() {  
        return "template/fragment/fragmentMain";  
    }  
}
```

먼저 공통으로 사용할 `footer.html`을 만들자.

`resources/templates/template/fragment/footer.html`
```html
<!DOCTYPE html>  
<html xmlns:th="http://www.thymeleaf.org">  
  
<body>  
  
<footer th:fragment="copy">  
    푸터 자리 입니다.  
</footer>  
  
<footer th:fragment="copyParam (param1, param2)">  
    <p>파라미터 자리 입니다.</p>  
    <p th:text="${param1}"></p>  
    <p th:text="${param2}"></p>  
</footer>  분 
```html
<footer th:fragment="copyParam (param1, param2)">  
    <p>파라미터 자리 입니다.</p>  
    <p th:text="${param1}"></p>  
    <p th:text="${param2}"></p>  
</footer>
```

그 다음에 이걸 사용할 메인 html을 만들자.

`/resources/templates/template/fragment/fragmentMain.html`
```html
<!DOCTYPE html>  
<html xmlns:th="http://www.thymeleaf.org">  
<head>  
    <meta charset="UTF-8">  
    <title>Title</title>  
</head>  
<body>  
<h1>부분 포함</h1>  
<h2>부분 포함 insert</h2>  
<div th:insert="~{template/fragment/footer :: copy}"></div>  
  
<h2>부분 포함 replace</h2>  
<div th:replace="~{template/fragment/footer :: copy}"></div>  
  
<h2>부분 포함 단순 표현식</h2>  
<div th:replace="template/fragment/footer :: copy"></div>  
  
<h1>파라미터 사용</h1>  
<div th:replace="~{template/fragment/footer :: copyParam ('데이터1', '데이터2')}"></div>  
</body>  
</html>
```

![](https://i.imgur.com/AUdXhA3.png){: .align-center}

결과는 다음과 같이 출력 됐다. 각각 알아보자.

- `template/fragment/footer :: copy`  
	- `template/fragment/footer.html`에 있는 `copy`라는 부분을 템플릿 조각으로 가져와서 사용한다.


### 부분 포함 insert

```html
<h2>부분 포함 insert</h2>  
<div th:insert="~{template/fragment/footer :: copy}"></div>  
```

실제 렌더링 된 소스를 보자.

![](https://i.imgur.com/6uqfSrV.png){: .align-center}


`th:insert`를 사용하면 현재 태그(`div`) 내부에 추가한다.

### 부분 포함 replace

```html
<h2>부분 포함 replace</h2>  
<div th:replace="~{template/fragment/footer :: copy}"></div>  
```

![](https://i.imgur.com/56byLla.png){: .align-center}

`th:replace`를 사용하면 현재 태그(`div`)를 대체한다

### 부분 포함 단순 표현식

```html
<h2>부분 포함 단순 표현식</h2>  
<div th:replace="template/fragment/footer :: copy"></div>  
```

![](https://i.imgur.com/dKXRYiB.png){: .align-center}

`~{...}`를 사용하는 것이 원칙이지만 템플릿 조각을 사용하는 코드가 단순하면 이 부분을 생략할 수 있다.


### 파라미터 사용

```html
<h1>파라미터 사용</h1>  
<div th:replace="~{template/fragment/footer :: copyParam ('데이터1', '데이터2')}"></div>
```

![](https://i.imgur.com/7kQl9gq.png){: .align-center}

다음과 같이 `copyParam` 템플릿 조각을 호출 하면서 파라미터를 전달해 동적으로 조각을 렌더링 시킬 수도 있다. 


## 템플릿 레이아웃 1 

이전에는 일부 코드 조각을 가지고 와서 사용했다면, 이번에는 개념을 더 확장해서 코드 조각을 레이아웃에 넘겨서 사용하는 방법에 대해서 알아보자.

예를 들어서 `<head>`에 공통으로 사용하는 `css`, `javascript`같은 정보들이 있는데, 이러한 공통 정보들을 한 곳에 모아두고, 

공통으로 사용하지만 각 페이지마다 필요한 정보를 더 추가해서 사용하고 싶다면 다음과 같이 사용하자.

컨트롤러에 추가하자.
```java
@GetMapping("/layout")  
public String layout() {  
    return "template/layout/layoutMain";  
}
```

이제 뼈대를 만들자.

`resources/templates/template/layout/base.html`
```html
<html xmlns:th="http://www.thymeleaf.org">  
<head th:fragment="common_header(title,links)">  
  
    <title th:replace="${title}">레이아웃 타이틀</title>  
  
    <!-- 공통 -->  
    <link rel="stylesheet" type="text/css" media="all" th:href="@{/css/awesomeapp.css}">  
    <link rel="shortcut icon" th:href="@{/images/favicon.ico}">  
    <script type="text/javascript" th:src="@{/sh/scripts/codebase.js}"></script>  
  
    <!-- 추가 -->  
    <th:block th:replace="${links}" />  
  
</head>
```

마지막으로 실행 될 페이지를 만들자.

`resources/templates/template/layout/layoutMain.html`
```html
<!DOCTYPE html>  
<html xmlns:th="http://www.thymeleaf.org">  
<head th:replace="template/layout/base :: common_header(~{::title},~{::link})">  
    <title>메인 타이틀</title>  
    <link rel="stylesheet" th:href="@{/css/bootstrap.min.css}">  
    <link rel="stylesheet" th:href="@{/themes/smoothness/jquery-ui.css}">  
</head>  
<body>  
메인 컨텐츠  
</body>  
</html>
```

![](https://i.imgur.com/Kz3vxZD.png){: .align-center}

결과는 다음과 같다.


- `common_header(~{::title},~{::link})` : 이 부분이 핵심이다.
	- `::title`은 현재 페이지의 title 태그를 전달한다.
		- `<title>메인 타이틀</title>`
	- `::link`는 현재 페이지의 link 태그를 전달한다.
		- `<link rel="stylesheet" th:href="@{/css/bootstrap.min.css}">`
		- `<link rel="stylesheet" th:href="@{/themes/smoothness/jquery-ui.css}">`

결과 소스를 보자.

```html
<!DOCTYPE html>
<html>
<head>
    <title>메인 타이틀</title>
    <!-- 공통 -->
    <link rel="stylesheet" type="text/css" media="all" href="/css/awesomeapp.css">
    <link rel="shortcut icon" href="/images/favicon.ico">
    <script type="text/javascript" src="/sh/scripts/codebase.js"></script>
    <!-- 추가 -->
    <link rel="stylesheet" href="/css/bootstrap.min.css"><link rel="stylesheet" href="/themes/smoothness/jquery-ui.css">

</head>
<body>
메인 컨텐츠
</body>
</html>
```

- 메인 타이틀이 전달한 부분으로 교체 됨.
- 공통 부분은 그대로 유지하고, 추가 부분에 전달한 `<link>`들이 포함된 것을 확인.

이 방식은 사실 앞서 배운 코드 조각을 조금 더 적극적으로 사용하는 방식. 

쉽게 이야기해서 레이아웃 개념을 두고, 그 레이아웃에 필요한 코드 조각들을 전달해서 완성하는 것으로 이해하면 된다.

## 템플릿 레이아웃 2

### 템플릿 레이아웃 확장

앞서 이야기한 개념을 `<head>`에만 사용하는 것이 아니라, `<html>`전체에 적용해 보자.

먼저 뼈대를 만들자.

`/resources/templates/template/layoutExtend/layoutFile.html`
```html
<!DOCTYPE html>  
<html th:fragment="layout (title, content)" xmlns:th="http://www.thymeleaf.org">  
<head>  
    <title th:replace="${title}">레이아웃 타이틀</title>  
</head>  
<body>  
<h1>레이아웃 H1</h1>  
<div th:replace="${content}">  
    <p>레이아웃 컨텐츠</p>  
</div>  
<footer>  
    레이아웃 푸터  
</footer>  
</body>  
</html>
```


```html
<head>  
    <title th:replace="${title}">레이아웃 타이틀</title>  
</head>  
```

여긴 타이틀이 바뀔 영역이다.

```html
<div th:replace="${content}">  
    <p>레이아웃 컨텐츠</p>  
</div> 
```

이 영역이 컨텐츠가 들어가서 바뀔 영역이다.

이제 내용을 한번 만들어 보자.

`/resources/templates/template/layoutExtend/layoutExtendMain.html`
```html
<!DOCTYPE html>  
<html th:replace="~{template/layoutExtend/layoutFile :: layout(~{::title}, ~{::section})}"  
      xmlns:th="http://www.thymeleaf.org">  
<head>  
    <title>메인 페이지 타이틀</title>  
</head>  
<body>  
<section>  
    <p>메인 페이지 컨텐츠</p>  
    <div>메인 페이지 포함 내용</div>  
</section>  
</body>  
</html>
```

![](https://i.imgur.com/vDrYNG9.png){: .align-center}

```html
<!DOCTYPE html>
<html>
<head>
    <title>메인 페이지 타이틀</title>
</head>
<body>
<h1>레이아웃 H1</h1>
<section>
    <p>메인 페이지 컨텐츠</p>
    <div>메인 페이지 포함 내용</div>
</section>
<footer>
    레이아웃 푸터
</footer>
</body>
</html>
```

보면 

```html
<html th:replace="~{template/layoutExtend/layoutFile :: layout(~{::title}, ~{::section})}"  
      xmlns:th="http://www.thymeleaf.org">  
```

여기서 title 태그와 section 태그를 넘겨서 `layoutFile.html`을 호출 했고

아까 미리 바뀔 영역을 만들어 놨기 때문에 내가 원하는 영역에 replace된 것을 볼 수 있다.

결국 `<html>`자체를 `th:replace`를 사용해서 변경하는 것을 확인할 수 있었다.

`layoutFile.html`에 필요한 내용을 전달하면서 `<html>`자체를 `layoutFile.html`로 변경한다.


