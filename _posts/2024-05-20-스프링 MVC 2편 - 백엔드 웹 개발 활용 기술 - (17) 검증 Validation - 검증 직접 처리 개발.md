---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (17) 검증 Validation - 검증 직접 처리 개발
aliases: 
tags:
  - spring
  - validation
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-05-20
last_modified_at: 2024-05-20
---
>  인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.


### 상품 등록 검증

먼저 상품 등록 검증 코드를 작성해 보자.

`ValidationItemControllerV1` - `addItem()` 수정 
```java
@PostMapping("/add")  
public String addItem(@ModelAttribute Item item, RedirectAttributes redirectAttributes, Model model) {  
  
    // 검증 오류 결과 보관  
    Map<String, String> errors = new HashMap<>();  
  
    // 검증 로직  
    if (!StringUtils.hasText(item.getItemName())) {  
        errors.put("itemName", "상품 이름은 필수 입니다.");  
    }  
    if (item.getPrice()==null || item.getPrice() < 1000 || item.getPrice() > 1000000) {  
        errors.put("price", "가격은 1,000 ~ 1,000,000 까지 허용됩니다.");  
    }  
    if (item.getQuantity()==null || item.getQuantity() >= 9999) {  
        errors.put("quantity", "수량은 최대 9,999 까지 허용됩니다.");  
    }  
  
    // 특정 필드가 아닌 복합 룰 검증  
    if (item.getPrice() != null && item.getQuantity() != null) {  
        int resultPrice = item.getPrice() * item.getQuantity();  
        if (resultPrice < 10000) {  
            errors.put("globalError", "가격 * 수량의 합은 10,000원 이상이여야 합니다. 현재 값 = " + resultPrice);  
        }  
    }  
  
    // 검증에 실패하면 다시 입력 폼으로 가야함.  
    if (!errors.isEmpty()) {  
        model.addAttribute("errors", errors);  
        return "validation/v1/addForm";  
    }  
  
    // 성공 로직  
    Item savedItem = itemRepository.save(item);  
    redirectAttributes.addAttribute("itemId", savedItem.getId());  
    redirectAttributes.addAttribute("status", true);  
    return "redirect:/validation/v1/items/{itemId}";  
}
```

돌려서 오류를 발생 시키게 넣고 로그를 찍어 보면..

![](https://i.imgur.com/UmhRd79.png){:.align-center}

다음과 같이 페이지 이동이 없는 것 같은데 (뷰단 에선 처리가 아직 안돼있어서.) 로그엔 남아있다.


#### 검증 오류 보관
`Map errors = new HashMap<>();`

만약 검증 시 오류가 발생하면 어떤 검증에서 오류가 발생했는지 정보를 담아둔다.

#### 검증 로직

```java
// 검증 로직  
if (!StringUtils.hasText(item.getItemName())) {  
	errors.put("itemName", "상품 이름은 필수 입니다.");  
}  
```

검증시 오류가 발생하면 `errors`에 담아둔다. 

이때 어떤 필드에서 오류가 발생했는지 구분하기 위해 오류가 발생한 필드명을 `key`로 사용한다. 

이후 뷰에서 이 데이터를 사용해서 고객에게 친절한 오류 메시지를 출력할 수 있다.

#### 특정 필드의 범위를 넘어서는 검증 로직 

```java
// 특정 필드가 아닌 복합 룰 검증  
if (item.getPrice() != null && item.getQuantity() != null) {  
	int resultPrice = item.getPrice() * item.getQuantity();  
	if (resultPrice < 10000) {  
		errors.put("globalError", "가격 * 수량의 합은 10,000원 이상이여야 합니다. 현재 값 = " + resultPrice);  
	}  
}  
```
특정 필드를 넘어서는 오류를 처리해야 할 수도 있다. 

이때는 필드 이름을 넣을 수 없으므로 `globalError`라는 key 를 사용한다.


#### 검증에 실패하면 다시 입력 폼으로 
```java
// 검증에 실패하면 다시 입력 폼으로 가야함.  
if (!errors.isEmpty()) {  
	model.addAttribute("errors", errors);  
	return "validation/v1/addForm";  
}
```

만약 검증에서 오류 메시지가 하나라도 있으면 오류 메시지를 출력하기 위해 `model`에 `errors`를 담고, 입력 폼이 있는 뷰 템플릿으로 보낸다.


#### 뷰 템플릿 수정

`resources/templates/validation/v1/addForm.html`
```css
.field-error {  
    border-color: #dc3545;  
    color: #dc3545;  
}
```

이 부분은 오류 메시지를 빨간색으로 강조하기 위해 추가했다.


```html
<form action="item.html" th:action th:object="${item}" method="post">  


    <div th:if="${errors?.containsKey('globalError')}">  
        <p class="field-error" th:text="${errors['globalError']}">전체 오류 메시지</p>  
    </div>
   .
   .
   .
</form>   
```

오류 메시지는 `errors`에 내용이 있을 때만 출력하면 된다. 타임리프의 `th:if`를 사용하면 조건에 만족할 때만 해당 HTML 태그를 출력할 수 있다.

![](https://i.imgur.com/IQZEkfT.png){: .align-center}

```html
<div th:if="${errors?.containsKey('globalError')}">
```

> **참고 Safe Navigation Operator**<br>만약 여기에서 `errors`가 `null`이라면 어떻게 될까?<br>생각해보면 등록 폼에 진입한 시점에는 `errors`객체가 없다.<br>따라서 `errors.containsKey()`를 호출하는 순간 `NullPointerException`이 발생한다.<br><br>`errors?.`은 `errors`가 `null`일 때 `NullPointerException`이 발생하는 대신, `null`을 반환하는 문법이다. <br>`th:if`에서 `null`은 실패로 처리되므로 오류 메시지가 출력 되지 않는다.<br>이것은 스프링의 SpringEL이 제공하는 문법이다. [자세한 내용은 다음을 참고하자.](https://docs.spring.io/spring-framework/reference/core/expressions/language-ref/operator-safe-navigation.html)


#### 필드 오류 처리

자 이제 필드 오류를 받았을 때 뷰 템플릿을 수정해 보자.


```html
<div>  
    <label for="itemName" th:text="#{label.item.itemName}">상품명</label>  
    <input type="text" id="itemName" th:field="*{itemName}"  
           th:class="${errors?.containsKey('itemName')} ? 'form-control field-error' : 'form-control'"           class="form-control" placeholder="이름을 입력하세요">  
    <div class="field-error" th:if="${errors?.containsKey('itemName')}" th:text="${errors[itemName]}">  
        상품명 오류  
    </div>  
</div>
```

상품명 오류를 추가했다.

여기서 잘 보면, 

```html
<div class="field-error" th:if="${errors?.containsKey('itemName')}" th:text="${errors[itemName]}">  
	상품명 오류  
</div>
```

`th:if="${errors?.containsKey('itemName')}"`으로 `errors` 객체에  `itemName`이 있는지 판단하고

있으면 `th:text="${errors[itemName]}"`로 텍스트를 출력한다.


```html
<input type="text" id="itemName" th:field="*{itemName}"  
           th:class="${errors?.containsKey('itemName')} ? 'form-control field-error' : 'form-control'"           
           class="form-control" placeholder="이름을 입력하세요">
```

그리고 

![](https://i.imgur.com/w3Aea5W.png){: .align-center}

이렇게 인풋 필드 테두리도 강조 하고 싶어서, 

`th:class="${errors?.containsKey('itemName')} ? 'form-control field-error' : 'form-control'"` 

이런 식으로 에러가 있나? -> 있으면 에러 클래스 -> 없으면 그냥 일반 클래스 이런 문법을 사용해서 동적으로 클래스를 바꾸도록 했다.

여기서 사용한 `th:class`는 다음과 같이

`th:classappend="${errors?.containsKey('itemName')} ? 'fielderror' : _"`

이걸로 대체할 수 있다. 이건 조건이 맞으면 추가하는 것.

이런 식으로 이름, 가격, 수량을 전부 추가해 주면 된다.

전체 코드는 다음과 같다.

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
        .field-error {  
            border-color: #dc3545;  
            color: #dc3545;  
        }  
    </style>  
</head>  
<body>  
  
<div class="container">  
  
    <div class="py-5 text-center">  
        <h2 th:text="#{page.addItem}">상품 등록</h2>  
    </div>  
  
    <form action="item.html" th:action th:object="${item}" method="post">  
  
        <div th:if="${errors?.containsKey('globalError')}">  
            <p class="field-error" th:text="${errors['globalError']}">전체 오류 메시지</p>  
        </div>  
  
        <div>  
            <label for="itemName" th:text="#{label.item.itemName}">상품명</label>  
            <input type="text" id="itemName" th:field="*{itemName}"  
                   th:classappend="${errors?.containsKey('itemName')} ? 'fielderror' : _"                   
                   class="form-control" placeholder="이름을 입력하세요">  
  
            <div class="field-error" th:if="${errors?.containsKey('itemName')}" th:text="${errors[itemName]}">  
                상품명 오류  
            </div>  
        </div>  
        <div>  
            <label for="price" th:text="#{label.item.price}">가격</label>  
            <input type="text" id="price" th:field="*{price}"  
                   th:class="${errors?.containsKey('price')} ? 'form-control field-error' : 'form-control'"                   
                   class="form-control" placeholder="가격을 입력하세요">  
            <div class="field-error" th:if="${errors?.containsKey('price')}" th:text="${errors[price]}">  
                가격 오류  
            </div>  
        </div>  
        <div>  
            <label for="quantity" th:text="#{label.item.quantity}">수량</label>  
            <input type="text" id="quantity" th:field="*{quantity}"  
                   th:class="${errors?.containsKey('quantity')} ? 'form-control field-error' : 'form-control'"                   
                   class="form-control" placeholder="수량을 입력하세요">  
            <div class="field-error" th:if="${errors?.containsKey('quantity')}" th:text="${errors[quantity]}">  
                수량 오류  
            </div>  
        </div>  
  
        <hr class="my-4">  
  
        <div class="row">  
            <div class="col">  
                <button class="w-100 btn btn-primary btn-lg" type="submit" th:text="#{button.save}">상품 등록</button>  
            </div>  
            <div class="col">  
                <button class="w-100 btn btn-secondary btn-lg"  
                        onclick="location.href='items.html'"  
                        th:onclick="|location.href='@{/validation/v1/items}'|"  
                        type="button" th:text="#{button.cancel}">취소</button>  
            </div>  
        </div>  
  
    </form>  
  
</div> <!-- /container -->  
</body>  
</html>
```

![](https://i.imgur.com/36Pts1w.png){: .align-center}


이런 식으로 모든 오류를 띄워 봤다.

#### 남은 문제점

- 뷰 템플릿에서 중복 처리가 많다. 반복이다. 이름, 가격, 수량 ...

- 타입 오류 처리가 안된다. `Item`의 `price`, `quantity`같은 숫자 필드는 `Integer`이므로 문자 타입으로설정 하는 것이 불가능 하다. 숫자 타입에 문자가 들어오면 오류가 발생한다. 그런데 이러한 오류는 스링 MVC에서 컨트롤러에 진입하기도 전에 예외가 발생하기 때문에, 컨트롤러가 호출 되지도 않고, 400 예외가 발생하면서 오류 페이지를 띄워 버린다.

- `Item`의 `price`에 문자를 입력하는 것 처럼 타입 오류가 발생해도 고객이 입력한 문자를 화면에 남겨야 한다. 만약 컨트롤러가 호출된다고 가정해도 `Item`의 `price`는 `Integer`이므로 문자를 보관할 수 없다. 결국 문자는 바인딩이 불가능하므로 고객이 입력한 문자가 사라지게 되고, 고객은 본인이 어떤 내용을 입력해서 오류가 발생했는지 이해하기 어렵다.

- 결국 고객이 입력한 값도 어딘가에 별도로 관리가 되어야 한다.



