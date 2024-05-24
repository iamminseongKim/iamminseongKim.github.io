---
title: 2024-05-27-스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (23) 검증 Validation - Validation 분리
aliases: 
tags:
  - spring
  - validation
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-05-27
last_modified_at: 2024-05-27
---

>  인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.


## Validation 분리 1 

> 복잡한 검증 로직을 별도로 분리하자.


컨트롤러에서 검증 로직이 차지하는 부분은 매우 크다. 이런 경우 별도의 클래스로 역할을 분리하는 것이 좋다.

그리고 이렇게 분리한 검증 로직을 재사용 할 수도 있다.

`ItemValidator`를 만들자.

`hello.itemservice.web.validation.ItemValidator`
```java
@Component  
public class ItemValidator implements Validator {  
    @Override  
    public boolean supports(Class<?> clazz) {  
        return Item.class.isAssignableFrom(clazz);  
        // item == clazz?  
        // item == subItem 즉 isAssignableFrom 는 자식 클래스까지 확인할 수 있음  
    }  
  
    @Override  
    public void validate(Object target, Errors errors) {  
        Item item = (Item) target;  
  
        // 검증 로직  
        if (!StringUtils.hasText(item.getItemName())) {  
            errors.rejectValue("itemName", "required");  
        }  
        if (item.getPrice()==null || item.getPrice() < 1000 || item.getPrice() > 1000000) {  
            errors.rejectValue("price", "range", new Object[]{1000, 1000000}, null);  
        }  
        if (item.getQuantity()==null || item.getQuantity() >= 9999) {  
            errors.rejectValue("quantity", "max", new Object[]{9999}, null);  
        }  
  
        // 특정 필드가 아닌 복합 룰 검증  
        if (item.getPrice() != null && item.getQuantity() != null) {  
            int resultPrice = item.getPrice() * item.getQuantity();  
            if (resultPrice < 10000) {  
                errors.reject("totalPriceMin", new Object[]{10000, resultPrice}, null);  
            }  
        }  
    }  
}
```

스프링은 검증을 체계적으로 제공하기 위해 다음 인터페이스를 제공한다.

```java
public interface Validator {  
    boolean supports(Class<?> clazz);  
    void validate(Object target, Errors errors);
}
```

- `supports() {}` :  해당 검증기를 지원하는 여부
- `validate(Object target, Errors errors)` : 검증 대상 객체와 `BindingResult`

그래서 `supports()` 에선 
```java
return Item.class.isAssignableFrom(clazz); 
```

이렇게 사용했고

지금 들어온 클래스가 Item 클래스 및 Item의 자식 클래스 인지 확인하는 메서드 이다.

`item == clazz` 이렇게 했으면 Item의 자식 클래스는 통과 못했을 것이다.

> `Class.isAssignableFrom`, `instanceof` 가 있다. 

`validate()` 에서는 이제  컨트롤러에 있던 검증 로직을 넣었다.


이제 컨트롤러단 에서 사용해 보자.

```java
@PostMapping("/add")  
public String addItemV5(@ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes, Model model) {  
  
    itemValidator.validate(item, bindingResult);  
  
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

자 `ItemValidator`를 `@Component`로 등록 해놨기 때문에 맨 위에 **생성자 주입**을 해줬고,


```java
itemValidator.validate(item, bindingResult);
```

이 한 줄로 검증이 끝났다.  뭔가 있으면 `bindingResult`에 담길 것이기 때문에 저 한 줄이면 된다. 

![](https://i.imgur.com/Xm8TALM.png){: .align-center}

잘 된다.


## Validator 분리 2

스프링이 `Validator`인터페이스를 별도로 제공하는 이유는 체계적으로 검증 기능을 도입하기 위해서 이다. 

그런데 앞에서는 검증기를 직접 불러서 사용했고, 이렇게 사용해도 된다. 

하지만 `Validator` 인터페이스를 사용해서 검증기를 만들면 스프링의 추가적인 도움을 받을 수 있다.

### WebDataBinder를 통해서 사용하기

`WebDataBinder`는 스프링의 파라미터 바인딩의 역할을 해주고 검증 기능도 내부에 포함한다.

컨트롤러에 다음 코드를 추가하자.

```java
@InitBinder  
public void init(WebDataBinder dataBinder) {  
    log.info("init binder {}", dataBinder);  
    dataBinder.addValidators(itemValidator);  
}
```

이렇게 `WebDataBinder`에 검증기를 추가하면 해당 컨트롤러에서는 검증기를 자동으로 적용할 수 있다.

`@InitBinder` → 해당 컨트롤러에만 영향을 준다. (글로벌로 하고 싶으면 따로 설정 해야 함.)

**@Validated 적용** `ValidationItemControllerV2 - addItemV6()`
```java
@PostMapping("/add")  
public String addItemV6(@Validated @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {  
  
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

자 이제 메서드 내에 검증 코드는 없어졌다.

대신에 

```java
@Validated @ModelAttribute Item item
```

검증 대상인 `Item` 앞에 `@Validated` 어노테이션이 붙었다.

![](https://i.imgur.com/BBlovmW.png){: .align-center}

잘 된다.

### 동작 방식

- `@Validated`는 검증기를 실행하라는 애노테이션이다.
- 이 애노테이션이 붙으면 앞서 `WebDataBinder`에 등록한 검증기를 찾아서 실행한다.
	- 그런데 검증기가 여러 개라면 어떤 검증기가 작동할 것인가?
	- 이때 `supports()`를 쓰는 것이다. `Item`관련 객체만 Item검증기의 `supports()`를 통과 하니깐.


### 글로벌 설정 - 모든 컨트롤러에 적용

메인 메서드에 `WebMvcConfigurer`를 상속하고 메서드를 구현하면 된다. 

```java
@SpringBootApplication  
public class ItemServiceApplication implements WebMvcConfigurer {  
  
    public static void main(String[] args) {  
       SpringApplication.run(ItemServiceApplication.class, args);  
    }  
    @Override  
    public Validator getValidator() {  
       return new ItemValidator();  
    }  
}
```

![](https://i.imgur.com/Omz7lV1.png){: .align-center}

컨트롤러에서 주석 처리한 후에 동작해보면 글로벌 설정이 잘 적용됐다.

