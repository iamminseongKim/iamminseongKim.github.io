---
title: 스프링 MVC - 1편 - 웹 페이지 만들기 - (5) 상품 수정, PRG PostRedirectGet, RedirectAttributes
aliases: 
tags:
  - spring
  - thymeleaf
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-04-24
last_modified_at: 2024-04-24
---
>  인프런 스프링 MVC 1편 - 백엔드 웹 개발 핵심 기술편을 학습하고 정리한 내용 입니다.

  
  

## 상품 수정

### 상품 수정 폼 컨트롤러

  

`hello.itemservice.web.basic.BasicItemController`

```java

@GetMapping("/{itemId}/edit")  

public String editForm(@PathVariable("itemId") Long itemId, Model model) {  

    Item item = itemRepository.findById(itemId);  

    model.addAttribute("item", item);  

    return "basic/editForm";  

}

```

  

`/resources/static/editForm.html` → 복사 → `/resources/templates/basic/editForm.html`

  

`/resources/templates/basic/editForm.html`

```html

<!DOCTYPE HTML>  

<html xmlns:th="http://www.thymeleaf.org">  

<head>  

    <meta charset="utf-8">  

    <link th:href="@{/css/bootstrap.min.css}"  

          href="../css/bootstrap.min.css" rel="stylesheet">  

    <style>  

        .container {  

            max-width: 560px;  

        }  

    </style>  

</head>  

<body>  

<div class="container">  

    <div class="py-5 text-center">  

        <h2>상품 수정 폼</h2>  

    </div>  

    <form action="item.html" th:action method="post">  

        <div>  

            <label for="id">상품 ID</label>  

            <input type="text" id="id" name="id" class="form-control" value="1" th:value="${item.id}" readonly>  

        </div>  

        <div>  

            <label for="itemName">상품명</label>  

            <input type="text" id="itemName" name="itemName" class="form-control" th:value="${item.itemName}" value="상품A">  

        </div>  

        <div>  

            <label for="price">가격</label>  

            <input type="text" id="price" name="price" class="form-control" th:value="${item.price}" value="10000">  

        </div>  

        <div>  

            <label for="quantity">수량</label>  

            <input type="text" id="quantity" name="quantity" class="form-control" th:value="${item.quantity}" value="10">  

        </div>  

        <hr class="my-4">  

        <div class="row">  

            <div class="col">  

                <button class="w-100 btn btn-primary btn-lg" type="submit">저장</button>  

            </div>  

            <div class="col">  

                <button class="w-100 btn btn-secondary btn-lg"  

                        th:onclick="|location.href='@{/basic/items/{itemId}(itemId=${item.id})}'|"  

                        onclick="location.href='item.html'" type="button">취소  

                </button>  

            </div>  

        </div>  

    </form>  

</div> <!-- /container -->  

</body>  

</html>

```

  

상품 수정에 `th:value`를 사용해서 값들을 세팅했다.

  

이제 상품 수정을 동작하게 하는 로직을 개발하자.

  

### 상품 수정 개발

  

```java

@PostMapping("/{itemId}/edit")  

public String edit(@PathVariable("itemId") Long itemId, @ModelAttribute Item item) {  

    itemRepository.update(itemId, item);  

    return "redirect:/basic/items/{itemId}";  

}

```

  

```java

public void update(Long itemId, Item updateParam) {  

    Item findItem = findById(itemId);  

    findItem.setItemName(updateParam.getItemName());  

    findItem.setPrice(updateParam.getPrice());  

    findItem.setQuantity(updateParam.getQuantity());  

}

```

  

뭐 이런 식이다.

  

- GET `/items/{itemId}/edit` : 상품 수정 폼

- POST `/items/{itemId}/edit` : 상품 수정 처리

  

![](https://i.imgur.com/GSm2WCR.png){: .align-center}

  

![](https://i.imgur.com/kJuYFz7.png){: .align-center}

  

다음과 같이 `/basic/items/{itemId}/edit` 이고 저장 하면..

  

![](https://i.imgur.com/jsnHUJl.png){: .align-center}

  

`/basic/items/{itemId}`로 리다이렉트 되었다. 물론 상품 수정도 잘 됐다.

  
  

### 리다이렉트

  

상품 수정은 마지막에 뷰 템플릿을 호출하는 대신에 상품 상세 화면으로 이동하도록 리다이렉트를 호출한다.

- 스프링은 `redirect:/..`으로 편리하게 리다이렉트를 지원한다.

- `redirect:/basic/items/{itemId}`

    - 컨트롤러의 매핑된 `@PathVariable`의 값은 `redirect`에도 사용할 수 있다.

    - `redirect:/basic/items/{itemId}` → `{itemId}`는 `@PathVariable("itemId") Long itemId` 값을 그대로 사용한다.

  
  

> **참고**<br>HTML Form 전송은 PUT, PATCH를 지원하지 않는다. GET, POST만 사용할 수 있다.<br>PUT, PATCH는 `HTTP API` 전송 시에 사용. <br>스프링에서 HTTP POST로 Form 요청할 때 히든 필드를 통해 PUT, PATCH 매핑을 사용하는 방법이 있지만, HTTP 요청 상 POST 요청이다.

  

<br>

  

![](https://i.imgur.com/pGm3Q2D.png)

  

내가 이전에 [상품 등록](https://iamminseongkim.github.io/spring/%EC%8A%A4%ED%94%84%EB%A7%81-MVC-1%ED%8E%B8-%EC%9B%B9-%ED%8E%98%EC%9D%B4%EC%A7%80-%EB%A7%8C%EB%93%A4%EA%B8%B0-(4)-%EC%83%81%ED%92%88-%EC%83%81%EC%84%B8,-%EB%93%B1%EB%A1%9D/#%EC%83%81%ED%92%88-%EB%93%B1%EB%A1%9D-%EC%B2%98%EB%A6%AC-v2---modelattribute) 할 때 리다이렉트를 왜 안하지.. ?

  

했는데 이유가 있다. 중복 방지를 알려주기 위해서...

  
  

## PRG Post/Redirect/Get

  

사실 지금까지 진행한 상품 등록 처리 컨트롤러는 심각한 문제가 있다. (addItemV1 ~ addItemV4)

  

상품 등록을 완료한 후에 새로고침을 해보면...

  

![](https://i.imgur.com/Xm5eb4y.png)

  
  

![](https://i.imgur.com/l5oCVOs.png)

  

계속 생겨버린다.

  

![](https://i.imgur.com/NFVQ39j.png){: .align-center}

  

**POST 등록 후 새로 고침**

![](https://i.imgur.com/T7geMCq.png){: .align-center}

  
  

```java

@PostMapping("/add")  

public String addItemV4(Item item) {  

    itemRepository.save(item);  

    return "basic/item";  

}

```

  

자 상품 등록한 코드이다.

  

POST 요청으로 등록 → 그냥 뷰 템플릿 띄운다.

  

이제 브라우저에 새로 고침을 하게 되면, 내가 마지막으로 한 요청이 POST이므로

  

POST `/add`가 한번 더 나가버리게 된다.

  

이때 해결하는 방법이 `redirect`인 것이다.

  

```java

@PostMapping("/add")  

public String addItemV5(Item item) {  

    itemRepository.save(item);  

    return "redirect:/basic/items" + item.getId();  

}

```

  

이런 식으로 고쳐야 한다.

  

상품 등록 처리 이후에 뷰 템플릿이 아니라 상품 상세 화면으로 리다이렉트 하도록 코드를 작성하자.

  

이런 문제 해결 방식을 `PRG Post/Redirect/Get`이라 한다.

  

#### 주의

  

`"redirect:/basic/items/" + item.getId()` `redirect`에서 `+item.getId()` 처럼 URL에 변수를 더해서 사용하는 것은 `URL 인코딩`이 안되기 때문에 위험하다. (id는 숫자여서 괜찮은데, 한글, 영어 같은 걸 넣으면..)

  

다음에 설명하는 `RedirectAttributes`를 사용하자.

  

## RedirectAttributes

  

상품을 저장하고 상품 상세 화면으로 리다이렉트 한 것 까지는 좋았다.

  

그런데 고객 입장에서 저장이 잘 된 것인지 아닌지 확신이 들지 않는다.

  

그래서 저장이 잘 되었으면 상품 상세 화면에서 `저장되었습니다.` 라는 메시지를 보여달라는 요구사항이 왔다.

  

간단히 해결해 보자.

  

컨트롤러에 추가할 거다.

  

```java

@PostMapping("/add")  

public String addItemV6(Item item, RedirectAttributes redirectAttributes) {  

    Item savedItem = itemRepository.save(item);  

    redirectAttributes.addAttribute("itemId", savedItem.getId());  

    redirectAttributes.addAttribute("status", true);  

    return "redirect:/basic/items/{itemId}";  

}

```

  

리다이렉트 할 때 간단히 `status=true` 를 추가해보자.

  

그리고 뷰 템플릿에서 이 값이 있으면, `저장되었습니다.` 라는 메시지를 출력 해보자.

  
  

![](https://i.imgur.com/IGMMAF6.png){: .align-center}

  

뷰단에 이렇게 추가했다.

  
  

![](https://i.imgur.com/d2K7Omm.png){: .align-center}

  

`http://localhost:8080/basic/items/3?status=true` url이 다음과 같이 나왔다.

  
  

### RedirectAttributes

  

`RedirectAttributes`를 사용하면 URL 인코딩도 해주고, `pathVariable`, 쿼리 파라미터까지 처리해준다.

  

- `redirect:/basic/items/{itemId}`

    - `pathVariable` 바인딩: `{itemId}`

    - 나머지는 쿼리 파라미터로 처리: `?status=true`

  
  

이제 뷰 쪽을 보면

  

- `th:if` : 해당 조건이 참이면 실행

- `${param.status}` : 타임리프에서 쿼리 파라미터를 편리하게 조회하는 기능

    - 원래는 컨트롤러에서 모델에 직접 담고 값을 꺼내야 한다. 그런데 쿼리 파라미터는 자주 사용해서 타임리프 에서 직접 지원한다.

  
  
  
  

---

<br>

드디어 스프링 MVC 1편이 모두 끝났다..

  

스프링의 내부 구조를 배우게 된 좋은 강의였다고 생각한다.

  

 이제 2편 봐야겠다..