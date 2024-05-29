---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (26) Bean Validation - 에러 코드, 오브젝트 오류
aliases: 
tags:
  - spring
  - validation
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-05-30
last_modified_at: 2024-05-30
---
>  인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.

## Bean Validation - 에러 코드

Bean Validation이 기본적으로 제공하는 오류 메시지를 변경하기 위해선 

Bean Validation을 적용하고 `bindingResult`에 등록된 검증 오류 코드를 보자.

![](https://i.imgur.com/3TxZ6SP.png){: .align-center}

오류 코드가 애노테이션 이름으로 등록된다. 아이템 이름은 `@NotBlank`를 사용했기 때문에 NotBlank로 시작한다.

마치 `typeMismatch`랑 비슷하다.

`NotBlank`라는 오류 코드를 기반으로 `MessageCodesResolver`를 통해 다양한 메시지 코드가 순서대로 생성된다.


**@NotBlank** (구체적인 것 부터, 범용성 으로)
- NotBlank.item.itemName
- NotBlank.itemName
- NotBlank.java.lang.String
- NotBlank

자 이걸 보면 이제 메시지 프로퍼티를 등록하면 되지 않을까?

`errors.properties`
```
#Bean Validation 추가 
NotBlank={0} 공백X 
Range={0}, {2} ~ {1} 허용 
Max={0}, 최대 {1}
```

![](https://i.imgur.com/N4PusRA.png){: .align-center}

자 내가 원하는 방향으로 변경 다.

`{0}`은 필드 명이고, `{1}, {2}...` 는 각 애노테이션 마다 다르다.

### BeanValidation 메시지 찾는 순서

1. 생성된 메시지 코드 순서대로 `messageSource`에서 메시지 찾기
2. 애노테이션의 `message`속성 사용 → `@NotBlank(message = "공백X {0}")`
3. 라이브러리가 제공하는 기본 값 사용 → 공백일 수 없습니다.


### 애노테이션의 message 사용 예

![](https://i.imgur.com/UeZsBUS.png){: .align-center}


이런 식으로 NotBlank 애노테이션에 message를 넣어줬다.

![](https://i.imgur.com/qNtUHVw.png){: .align-center}

잘 된다. 

그런데 만약에 `errors.properties`에도 메시지를 지정한다면???


![](https://i.imgur.com/ihZEo30.png){: .align-center} 

지금 둘 다 메시지가 설정 되어 있다.

![](https://i.imgur.com/qwGumzn.png){: .align-center}


메시지 순서에 따라 프로퍼티 먼저 찾고, 그 후에 애노테이션의 `message`속성을 찾는 걸 확인할 수 있었다!




## Bean Validation - 오브젝트 오류


Bean Validation에서 특정 필드(`FieldError`)가 아닌 해당 오브젝트 관련 오류(`ObjectError`)는 어떻게 처리할 수 있을까?

다음과 같이 `@ScriptAssert()`를 사용하면 된다.


![](https://i.imgur.com/J2vudWx.png){: .align-center}

![](https://i.imgur.com/hPQkDGl.png){: .align-center}

그런데 [JDK 14 이후 버전 부터는  ScriptAssert에서 javascript를 사용할 수 없다고 한다.](https://www.inflearn.com/questions/910700/scriptassert-%EC%8A%A4%ED%94%84%EB%A7%81-3-0-1-%EC%9D%B4%EC%83%81-jdk-17-%EB%B2%84%EC%A0%84-%EC%9D%B4%EC%83%81-%EC%8B%A4%ED%96%89-%EB%B6%88%EA%B0%80-%EC%9E%84%EC%8B%9C%EB%B0%A9%ED%8E%B8)

그리고 실제로 사용한다 해도 제약이 많고 복잡하다. 

실무에서는 검증 기능이 해당 객체의 범위를 넘어서는 경우도 있어서 그럼 대응이 어렵다.

따라서 오브젝트 오류(글로벌 오류)의 경우 `@ScriptAssert`를 쓰는 것보다 

자바 코드로 직접 작성하는 것이 바람직하다.


```java
@PostMapping("/add")  
public String addItem(@Validated @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {  
  
    //특정 필드 예외가 아닌 전체 예외  
    if (item.getPrice() != null && item.getQuantity() != null) {  
        int resultPrice = item.getPrice() * item.getQuantity();  
        if (resultPrice < 10000) {  
            bindingResult.reject("totalPriceMin", new Object[]{10000,  
                    resultPrice}, null);  
        }  
    }  
    // 검증에 실패하면 다시 입력 폼으로 가야함.  
    if (bindingResult.hasErrors()) {  
        log.info("errors: {}", bindingResult);  
        return "validation/v3/addForm";  
    }  
  
    // 성공 로직  
    Item savedItem = itemRepository.save(item);  
    redirectAttributes.addAttribute("itemId", savedItem.getId());  
    redirectAttributes.addAttribute("status", true);  
    return "redirect:/validation/v3/items/{itemId}";  
}
```


그래서 예전으로 돌아가서 글로벌 에러는 컨트롤러 안에 직접 구현했다. (메서드로 뽑아도 될 듯.)


### Bean Validation - 수정에 적용

상품 수정에도 빈 검증을 적용해 보자.

```java 
@PostMapping("/{itemId}/edit")  
public String edit(@PathVariable Long itemId, @ModelAttribute Item item) {  
    itemRepository.update(itemId, item);  
    return "redirect:/validation/v3/items/{itemId}";  
}
```

원래 메서드 이었고, 이제 바꿔보자.

```java
@PostMapping("/{itemId}/edit")  
public String edit(@PathVariable Long itemId, @Validated @ModelAttribute Item item, BindingResult bindingResult) {  
  
    //특정 필드 예외가 아닌 전체 예외  
    if (item.getPrice() != null && item.getQuantity() != null) {  
        int resultPrice = item.getPrice() * item.getQuantity();  
        if (resultPrice < 10000) {  
            bindingResult.reject("totalPriceMin", new Object[]{10000,  
                    resultPrice}, null);  
        }  
    }  
  
    if (bindingResult.hasErrors()) {  
        log.info("errors: {}", bindingResult);  
        return "validation/v3/editForm";  
    }  
  
    itemRepository.update(itemId, item);  
    return "redirect:/validation/v3/items/{itemId}";  
}
```

자 `@Validated`을  Item객체에 붙여줘서 검증하도록 했고, 오브젝트 에러는 자바 코드로 직접 넣어 줬다.

그리고 오류를 담을 `BindingResult bindingResult`도 선언해 놨다.

마지막으로 오류 가 있다면 다시 수정 폼으로 돌아가도록 `return "validation/v3/editForm"` 라고 작성했다.

이제 뷰에도 오류가 나올 수 있도록 수정하자.

![](https://i.imgur.com/8AWpyUY.png){: .align-center}

[addForm](https://github.com/iamminseongKim/spring-mvc-study-2/blob/main/validation/src/main/resources/templates/validation/v2/addForm.html)을 잘 따라가면 된다.


```html
...
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
        <h2 th:text="#{page.updateItem}">상품 수정</h2>  
    </div>  
  
    <form action="item.html" th:action th:object="${item}" method="post">  
  
        <div th:if="${#fields.hasGlobalErrors()}">  
            <p class="field-error" th:each="err : ${#fields.globalErrors()}" th:text="${err}">전체 오류 메시지</p>  
        </div>  
  
        <div>  
            <label for="id" th:text="#{label.item.id}">상품 ID</label>  
            <input type="text" id="id" th:field="*{id}" class="form-control" readonly>  
        </div>  
        <div>  
            <label for="itemName" th:text="#{label.item.itemName}">상품명</label>  
            <input type="text" id="itemName" th:field="*{itemName}"  
                   th:errorclass="field-error" class="form-control">  
            <div class="field-error" th:errors="*{itemName}">  
                상품명 오류  
            </div>  
        </div>  
        <div>  
            <label for="price" th:text="#{label.item.price}">가격</label>  
            <input type="text" id="price" th:field="*{price}"  
                   th:errorclass="field-error" class="form-control">  
            <div class="field-error" th:errors="*{price}">  
                가격 오류  
            </div>  
        </div>  
        <div>  
            <label for="quantity" th:text="#{label.item.quantity}">수량</label>  
            <input type="text" id="quantity" th:field="*{quantity}"  
                  th:errorclass="field-error" class="form-control">  
            <div class="field-error" th:errors="*{quantity}">  
                수량 오류  
            </div>  
        </div>
    ...
    
```

이렇게 작성 해줬다.

![](https://i.imgur.com/epyDtia.png){: .align-center}

메시지가 잘 나온다!


