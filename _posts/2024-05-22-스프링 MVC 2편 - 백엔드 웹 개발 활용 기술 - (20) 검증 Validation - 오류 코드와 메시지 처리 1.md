---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (20) 검증 Validation - 오류 코드와 메시지 처리 1
aliases: 
tags:
  - spring
  - validation
  - message
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-05-22
last_modified_at: 2024-05-22
---
>  인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.

## 오류 코드와 메시지 처리 1

### FieldError 생성자 
`FieldError`는 두 가지 생성자를 제공한다.
```java
public FieldError(String objectName, String field, String defaultMessage) {  
    this(objectName, field, (Object)null, false, (String[])null, (Object[])null, defaultMessage);  
}  
  
public FieldError(String objectName, String field, @Nullable Object rejectedValue, boolean bindingFailure, @Nullable String[] codes, @Nullable Object[] arguments, @Nullable String defaultMessage) {  
    super(objectName, codes, arguments, defaultMessage);  
    Assert.notNull(field, "Field must not be null");  
    this.field = field;  
    this.rejectedValue = rejectedValue;  
    this.bindingFailure = bindingFailure;  
}
```

**파라미터 목록**
- `objectName` : 오류가 발생한 객체 이름
- `field` : 오류 필드
- `rejectedValue` : 사용자가 입력한 값(거절된 값)
- `bindingFailire` :  타입 오류 같은 바인딩 실패인지, 검증 실패인지 구분 값
- `codes` : 메시지 코드
- `arguments` : 메시지에서 사용하는 인자
- `defaultMessage` : 기본 오류 메시지

`FiledError`, `ObjectError`의 생성자는 `code`, `arguments`를 제공한다. 이것은 오류 발생 시 오류 코드로 메시지를 찾기 위해 사용된다.

### errors 메시지 파일 생성

`messages.properties`를 사용해도 되지만, 오류 메세지를 구분하기 쉽게 `errors.properties`라는 별도의 파일로 관리해보자.

먼저 스프링 부트가 해당 메시지 파일을 인식할 수 있게 다음 설정을 추가한다. 
이렇게 하면 `messages.properties`, `errors.properties` 두 파일을 모두 인식한다. (생략하면 `messages.properties`를 기본으로 인식한다.)


#### 스프링 부트 메시지 설정 추가

`application.properties`
```
spring.messages.basename=messages,errors
```


#### errors.properties 추가
`src/main/resources/errors.properties`
```
required.item.itemName=상품 이름은 필수입니다.  
range.item.price=가격은 {0} ~ {1} 까지 허용합니다.  
max.item.quantity=수량은 최대 {0} 까지 허용합니다.  
totalPriceMin=가격 * 수량의 합은 {0}원 이상이여야 합니다. 현재 값 = {1}
```

> 참고 : `errors_en.properties` 파일을 만들면 오류도 국제화 처리 가능하다.

이제 `errors`에 등록한 메시지를 사용하도록 코드를 변경해보자.


### ValidationItemControllerV2 - addItemV3() 추가

```java
@PostMapping("/add")  
public String addItemV3(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {  
  
    // 검증 로직  
    if (!StringUtils.hasText(item.getItemName())) {  
        bindingResult.addError(new FieldError("item", "itemName", item.getItemName(), false, new String[]{"required.item.itemName"}, null, null));  
    }  
    if (item.getPrice()==null || item.getPrice() < 1000 || item.getPrice() > 1000000) {  
        bindingResult.addError(new FieldError("item", "price", item.getPrice(), false, new String[]{"range.item.price"}, new Object[]{1000, 1000000}, null));  
    }  
    if (item.getQuantity()==null || item.getQuantity() >= 9999) {  
        bindingResult.addError(new FieldError("item", "quantity", item.getQuantity(), false, new String[]{"max.item.quantity"}, new Object[]{9999}, null));  
    }  
  
    // 특정 필드가 아닌 복합 룰 검증  
    if (item.getPrice() != null && item.getQuantity() != null) {  
        int resultPrice = item.getPrice() * item.getQuantity();  
        if (resultPrice < 10000) {  
            bindingResult.addError(new ObjectError("item", new String[]{"totalPriceMin"}, new Object[]{10000, resultPrice}, null));  
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

자 기존에 

```java
new FieldError("item", "itemName", item.getItemName(), false, null, null, "상품 이름은 필수 입니다.")
```

이렇게 `codes`, `arguments` 자리에 `null`을 넣어놓고 `defaultMessage`에 원하는 문구를 넣었다면

이젠 

```java
new FieldError("item", "itemName", item.getItemName(), false, new String[]{"required.item.itemName"}, null, null
```

이걸 `errors.properties`의 값으로 넣어준 거다.

그리고 `arguments`가 있는 메시지라면 다음과 같이 넣으면 된다.

```java
new FieldError("item", "price", item.getPrice(), false, new String[]{"range.item.price"}, new Object[]{1000, 1000000}, null)
```

- `codes` : `required.item.itemName`를 사용해서 메시지 코드를 지정한다. 메시지 코드는 하나가 아니라 배열로 여러 값을 전달할 수 있는데, 순서대로 매칭해서 처음 매칭되는 메시지가 사용된다.
- `arguments` : `Object[] {1000, 1000000}`를 사용해서 코드의 `{0}, {1}`로 치환할 값을 전달한다.

자 실행해 보면

![](https://i.imgur.com/lEcQbKC.png){: .align-center}

properites가 다 깨져 버렸다.

![](https://i.imgur.com/qz9IJPr.png){: .align-center}

다음과 같이 인텔리제이 세팅 - `Editor` - `File Encodings`에  `Default encoding for properties files`를  `utf-8`로 변경하자.

그리고 재기동 하면

![](https://i.imgur.com/FzpkK5u.png){: .align-center}

다음과 같이 잘 나온다.


## 오류 코드와 메시지 처리 2

- `FiledError`, `ObjectError`는 다루기 너무 번거롭다.
- 오류 코드도 좀 더 자동화 할 수 있지 않을까? 


컨트롤러에서 `BindingResult`는 검증 해야 할 객체인 `target`바로 다음에 온다.

따라서 `BindingResult`는 이미 본인이 검증 해야 할 객체인 `target`을 알고 있다.

### rejectValue() , reject()

`BindingResult`가 제공하는 `rejectValue()`, `reject()`를 사용하면 `FieldError`, `ObjectError`를 직접 생성하지 않고, 깔끔하게 검증 오류를 다룰 수 있다.

`rejectValue()`, `reject()`를 사용해서 기존 코드를 단순화 해보자.


### ValidationItemControllerV2 - addItemV4() 추가 
```java
@PostMapping("/add")  
public String addItemV4(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {  
  
    // 검증 로직  
    if (!StringUtils.hasText(item.getItemName())) {  
        bindingResult.rejectValue("itemName", "required");  
    }  
    if (item.getPrice()==null || item.getPrice() < 1000 || item.getPrice() > 1000000) {  
        bindingResult.rejectValue("price", "range", new Object[]{1000, 1000000}, null);  
    }  
    if (item.getQuantity()==null || item.getQuantity() >= 9999) {  
        bindingResult.rejectValue("quantity", "max", new Object[]{9999}, null);  
    }  
  
    // 특정 필드가 아닌 복합 룰 검증  
    if (item.getPrice() != null && item.getQuantity() != null) {  
        int resultPrice = item.getPrice() * item.getQuantity();  
        if (resultPrice < 10000) {  
            bindingResult.reject("totalPriceMin", new Object[]{10000, resultPrice}, null);  
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


![](https://i.imgur.com/scS4XiB.png){: .align-center}

잘 된다. 

#### rejectValue()
```java
void rejectValue(@Nullable String field, String errorCode, @Nullable Object[] errorArgs, @Nullable String defaultMessage);
```

- `filed` : 오류 필드 명 
- `errorCode` : 오류 코드(이 오류 코드는 메시지에 등록된 코드가 아니다. 뒤에서 설명할 messageResolver를 위한 오류 코드이다.)
- `errorArgs` : 오류 메시지에서 `{0}`을 치환하기 위한 값
- `defaultMessage` : 오류 메시지를 찾을 수 없을 때 사용하는 기본 메시지


```java
// AS-IS
bindingResult.addError(new FieldError("item", "price", item.getPrice(), false, new String[]{"range.item.price"}, new Object[]{1000, 1000000}, null));

// TO-BE
bindingResult.rejectValue("price", "range", new Object[]{1000, 1000000}, null); 
```

`BindingResult`는 어떤 객체를 대상으로 검증하는지 `target`을 이미 알고 있다. 

따라서 target(item)에 대한 정보는 없어도 된다. 오류 필드명은 `price`를 동일하게 했다.


#### 축약된 오류 코드 
`FieldError()`를 직접 다룰 때는 오류 코드를 `new String[]{"range.item.price"}` 이렇게 사용했다. 그런데 `rejectValue()`를 사용할 땐 `"range"` 이게 끝이다. 

그래도 잘 된다. 규칙이 있다. 이는 `MessageCodesResolver`을 알아야 한다. 다음에 알아보자.

`arguments`는 `FieldError()`랑 동일하고, 객체가 필요 없을 땐 `bindingResult.reject()`를 사용하면 된다.

```java
bindingResult.reject("totalPriceMin", new Object[]{10000, resultPrice}, null);  
```