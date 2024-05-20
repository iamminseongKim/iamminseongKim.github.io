---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (18) 검증 Validation - V2, BindingResult
aliases: 
tags:
  - spring
  - validation
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-05-21
last_modified_at: 2024-05-21
---

>  인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.

## 프로젝트 준비 V2

앞서 만든 기능을 유지하기 위해, 컨트롤러와 템플릿을 복사.

![](https://i.imgur.com/5a0oG2u.png){: .align-center}


### ValidationItemControllerV2 컨트롤러 생성

- `hello.itemservice.web.validation.ValidationItemControllerV1` 복사
- `hello.itemservice.web.validation.ValidationItemControllerV2` 붙여넣기
- URL 경로 변경: `validation/v1/`→ `validation/v2/`

![](https://i.imgur.com/9uKNQJq.png){: .align-center}


### 템플릿 파일 복사

`validation/v1` 디렉토리 모든 파일 `validation/v2`로 복사  

하위 모든 URL 경로 변경 : `validation/v1/`→ `validation/v2/`

![](https://i.imgur.com/am3PFOE.png){: .align-center}

인텔리제이에서 디렉토리를 클릭 후 

`Ctrl + Shift + R` 누르면 모든 경로 한번에 바꿀 수 있긴 하다.

![](https://i.imgur.com/OMtcgzw.png){: .align-center}

url이 v2로 잘 나오면 성공


## BindingResult1

지금부터 스프링이 제공하는 검증 오류 처리 방법을 알아보자. 여기서 핵심은 `BindingResult`이다. 

우선 코드로 확인 해보자.

`ValidationItemControllerV2` - `addItemV1`
```java
@PostMapping("/add")  
public String addItemV1(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {  
  
  
    // 검증 로직  
    if (!StringUtils.hasText(item.getItemName())) {  
        bindingResult.addError(new FieldError("item", "itemName", "상품 이름은 필수 입니다."));  
    }  
    if (item.getPrice()==null || item.getPrice() < 1000 || item.getPrice() > 1000000) {  
        bindingResult.addError(new FieldError("item", "price", "가격은 1,000 ~ 1,000,000 까지 허용됩니다."));  
    }  
    if (item.getQuantity()==null || item.getQuantity() >= 9999) {  
        bindingResult.addError(new FieldError("item", "quantity", "수량은 최대 9,999 까지 허용됩니다."));  
  
    }  
  
    // 특정 필드가 아닌 복합 룰 검증  
    if (item.getPrice() != null && item.getQuantity() != null) {  
        int resultPrice = item.getPrice() * item.getQuantity();  
        if (resultPrice < 10000) {  
            bindingResult.addError(new ObjectError("item", "가격 * 수량의 합은 10,000원 이상이여야 합니다. 현재 값 = " + resultPrice));  
        }  
    }  
  
    // 검증에 실패하면 다시 입력 폼으로 가야함.  
    if (bindingResult.hasErrors()) {  
        log.info("errors: {}", bindingResult);  
        return "validation/v2/addForm";  
    }  
  
    // 성공 로직  
    Item savedItem = itemRepository.save(item);  
    redirectAttributes.addAttribute("itemId", savedItem.getId());  
    redirectAttributes.addAttribute("status", true);  
    return "redirect:/validation/v2/items/{itemId}";  
}
```

**주의**

`BindingResult bindingResult` 파라미터 위치는 `@ModelAttribute Item item` **다음에 와야한다.**


### 필드 오류 - FieldError 
```java
if (!StringUtils.hasText(item.getItemName())) {  
	bindingResult.addError(new FieldError("item", "itemName", "상품 이름은 필수 입니다."));  
}
```

![](https://i.imgur.com/L2mtd3M.png)

필드에 오류가 있으면 `FieldError`객체를 생성해서 `bindingResult`에 담아두면 된다.

- `objectName` : `@ModelAttribute`
- `field` : 오류가 발생한 필드 이름
- `defaultMessage` : 오류 기본 메세지

### 글로벌 오류 - ObjectError

```java
// 특정 필드가 아닌 복합 룰 검증  
if (item.getPrice() != null && item.getQuantity() != null) {  
	int resultPrice = item.getPrice() * item.getQuantity();  
	if (resultPrice < 10000) {  
		bindingResult.addError(new ObjectError("item", "가격 * 수량의 합은 10,000원 이상이여야 합니다. 현재 값 = " + resultPrice));  
	}  
}  
```

특정 필드를 넘어서는 오류가 있으면 `ObjectError`객체를 생성해서 `bindingResult` 에 담아두면 된다.
- `objectName` : `@ModelAttribute`의 이름
- `defaultMessage` : 오류 기본 메세지


### 템플릿 수정

`validation/v2/addForm.html` 수정 
```html
<div th:if="${#fields.hasGlobalErrors()}">  
    <p class="field-error" th:each="err : ${#fields.globalErrors()}" th:text="${err}">전체 오류 메시지</p>  
</div>  
  
<div>  
    <label for="itemName" th:text="#{label.item.itemName}">상품명</label>  
    <input type="text" id="itemName" th:field="*{itemName}"  
           th:errorclass="field-error" class="form-control" placeholder="이름을 입력하세요">  
    <div class="field-error" th:errors="*{itemName}">  
        상품명 오류  
    </div>  
</div>  
<div>  
    <label for="price" th:text="#{label.item.price}">가격</label>  
    <input type="text" id="price" th:field="*{price}"  
           th:errorclass="field-error"  
           class="form-control" placeholder="가격을 입력하세요">  
    <div class="field-error" th:errors="*{price}">  
        가격 오류  
    </div>  
</div>  
<div>  
    <label for="quantity" th:text="#{label.item.quantity}">수량</label>  
    <input type="text" id="quantity" th:field="*{quantity}"  
           th:errorclass="field-error"  
           class="form-control" placeholder="수량을 입력하세요">  
    <div class="field-error" th:errors="*{quantity}">  
        수량 오류  
    </div>
```


#### 타임리프 스프링 검증 오류 통합 기능

타임리프는 스프링의 `BindingResult`를 활용해서 편리하게 검증 오류를 표현하는 기능을 제공한다.

- `#fields` : `#fields`로 `BindingResult`가 제공하는 검증 오류에 접근할 수 있다.
	- 글로벌 오류가 여러 개 있을 수도 있어서 `th:each="err : ${#fields.globalErrors()}" th:text="${err}"` 이런 식으로 모두 출력하게 만들었다.
- `th:errors` : 해당 필드에 오류가 있는 경우 태그를 출력한다. `th:if`의 편의 버전이다.
	- `th:errors="*{itemName}"` 이렇게 사용해도 되는 이유는 `new FieldError("item", "itemName","메세지")` 여기서 `filed`를 따라가기 때문에
- `th:errorclass` : `th:field`에서 지정한 필드에 오류가 있으면 `class` 정보를 추가한다.

- [검증과 오류 메시지 공식 메뉴얼](https://www.thymeleaf.org/doc/tutorials/3.0/thymeleafspring.html#validation-and-error-messages) 


![](https://i.imgur.com/7pdLcDH.png){: .align-center}

되긴 되는데 가격에 0 입력하고 수량 99999 입력했는데 입력 값이 유지가 안된다.


## BindingResult2

- 스프링이 제공하는 검증 오류를 보관하는 객체이다. 검증 오류가 발생하면 여기에 보관하면 된다.
- `BindingResult`가 있으면 `@ModelAttribute`에 데이터 바인딩 시 오류가 발생해도 컨트롤러가 호출된다.

**예) @ModelAttribute에 바인딩 시 타입 오류가 발생하면?**

- `BindingResult`가 없으면 → 400 오류 발생하면서 컨트롤러가 호출되지 않고, 오류 페이지로 이동
- `BindingResult`가 있으면 → 오류 정보(`FieldError`)를 `BindingResult`에 담아서 컨트롤러를 정상 호출한다.

**BindingResult에 검증 오류를 적용하는 3가지 방법**

- `@ModelAttribute`의 객체에 타입 오류 등으로 바인딩이 실패하는 경우 스프링이 `FieldError`생성해서 `BindingResult`에 넣어준다.
- 개발자가 직접 넣어준다.
- `Validator` 사용 → 나중에 설명

**타입 오류 확인**

숫자가 입력되어야 할 곳에 문자를 입력해서 타입을 다르게 해서 `BindingResult`를 호출하고 `BindingResult`의 값을 확인해 보자.

![](https://i.imgur.com/VEgx8JK.png){: .align-center}

**주의**

- `BindingResult`는 검증할 대상 바로 다음에 와야 한다. 순서가 중요하다. 예를 들어서 `@ModelAttribute Item item` 바로 다음에 `BindingResult`가 와야한다.
- `BindingResult`는 Model에 자동으로 포함된다


### BindingResult와 Errors

- `org.springframework.validation.Errors`
- `org.springframework.validation.BindingResult`

`BindingResult`는 인터페이스이고, `Errors`인터페이스를 상속 받고 있다.

실제 넘어오는 구현체는 `BeanPropertyBindingResult`라는 것인데, 둘다 구현하고 있으므로 `BindingResult`대신에 `Errors`를 사용해도 된다. `Errors`인터페이스는 단순한 오류 저장과 조회 기능을 제공한다.

`BindingResult`는 여기에 더해서 추가적인 기능을 제공한다. 

`addError()`도 `BindingResult`가 제공하므로 여기서는 `BindingResult`를 사용하자.

주로 관례상 `BindingResult`를 많이 사용한다.

