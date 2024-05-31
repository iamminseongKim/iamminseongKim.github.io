---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (29) Bean Validation - Form 전송 객체 분리
aliases: 
tags:
  - spring
  - validation
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-06-03
last_modified_at: 2024-06-03
---

>  인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.

## Form 전송 객체 분리 - 개발

`Item` 도메인 클래스 검증 코드 제거 

이제 `Item`에서 검증은 사용하지 않을 것이기 때문에 검증 코드 제거.


```java
@Data  
public class Item {  
  
    private Long id;  
    private String itemName;  
    private Integer price;  
    private Integer quantity;  
    
    public Item() {}  
  
	public Item(String itemName, Integer price, Integer quantity) {  
	    this.itemName = itemName;  
	    this.price = price;  
	    this.quantity = quantity;  
	}
}
```


### 검증 객체 만들기

먼저 패키지를 

![](https://i.imgur.com/IT06Gft.png){: .align-center}

다음과 같이 `hello.itemservice.web.validation.form`으로 만들었고

거기에 dto인 ItemSaveForm, ItemUpdateForm을 만들었다.

#### ItemSaveForm

저장할 때 사용할 dto, 그러므로 id는 필요 없다 (자동 생성)

```java
@Data  
public class ItemSaveForm {  
  
    @NotBlank  
    private String itemName;  
  
    @NotNull  
    @Range(min = 1000, max = 1000000)  
    private Integer price;  
  
    @NotNull  
    @Max(9999)  
    private Integer quantity;  
  
}
```


#### ItemUpdateForm

여기선 id가 중요하다. 그리고 수량 (`quantity`)에 제한이 없어진다.

```java
@Data  
public class ItemUpdateForm {  
  
    @NotNull  
    private Long id;  
  
    @NotBlank  
    private String itemName;  
  
    @NotNull  
    @Range(min = 1000, max = 1000000)  
    private Integer price;  
  
    // 수정에서는 수량은 자유롭게 변경 가능  
    private Integer quantity;  
  
}
```


이제 이 두 dto를 사용하도록 컨트롤러 코드를 수정하자.


### ValidationItemControllerV4 

#### 등록

```java

@PostMapping("/add")  
public String addItem(@Validated @ModelAttribute("item") ItemSaveForm form, BindingResult bindingResult, RedirectAttributes redirectAttributes) {  
  
    //특정 필드 예외가 아닌 전체 예외  
    if (form.getPrice() != null && form.getQuantity() != null) {  
        int resultPrice = form.getPrice() * form.getQuantity();  
        if (resultPrice < 10000) {  
            bindingResult.reject("totalPriceMin", new Object[]{10000,  
                    resultPrice}, null);  
        }  
    }  
  
    // 검증에 실패하면 다시 입력 폼으로 가야함.  
    if (bindingResult.hasErrors()) {  
        log.info("errors: {}", bindingResult);  
        return "validation/v4/addForm";  
    }  
  
    // 성공 로직 (setter는 지양하자..)  
    Item item = new Item();  
    item.setItemName(form.getItemName());  
    item.setPrice(form.getPrice());  
    item.setQuantity(form.getQuantity());  
  
    Item savedItem = itemRepository.save(item);  
    redirectAttributes.addAttribute("itemId", savedItem.getId());  
    redirectAttributes.addAttribute("status", true);  
    return "redirect:/validation/v4/items/{itemId}";  
}

```


`@Validated @ModelAttribute("item") ItemSaveForm form` 이렇게 바꿨고, 

이제 리포지토리에 넘길 객체를 따로 만들어 줬다.. 근데 setter말고 생성자나 builder 패턴을 사용 하는게 좋을 듯 하다. 

![](https://i.imgur.com/IHVYL4s.png){: .align-center}


#### 수정

```java
@PostMapping("/{itemId}/edit")  
public String edit(@PathVariable Long itemId, @Validated @ModelAttribute("item")ItemUpdateForm form, BindingResult bindingResult) {  
  
    //특정 필드 예외가 아닌 전체 예외  
    if (form.getPrice() != null && form.getQuantity() != null) {  
        int resultPrice = form.getPrice() * form.getQuantity();  
        if (resultPrice < 10000) {  
            bindingResult.reject("totalPriceMin", new Object[]{10000,  
                    resultPrice}, null);  
        }  
    }  
  
    if (bindingResult.hasErrors()) {  
        log.info("errors: {}", bindingResult);  
        return "validation/v4/editForm";  
    }  
  
    // 성공 로직 (setter는 지양하자..)  
    Item item = new Item();  
    item.setItemName(form.getItemName());  
    item.setPrice(form.getPrice());  
    item.setQuantity(form.getQuantity());  
  
    itemRepository.update(itemId, item);  
    return "redirect:/validation/v4/items/{itemId}";  
}
```

수정도 등록과 똑같다. 

![](https://i.imgur.com/srvjULg.png){: .align-center}


#### 주의 

`@ModelAttribute("item")` 에 `item`이름을 넣어준 부분을 주의하자. 이것을 넣지 않으면 `ItemSaveForm`의 경우 규칙에 의해 `itemSaveForm`이라는 이름으로 MVC Model에 담기게 된다..

이렇게 되면  뷰 템플릿에 이름을 다 바꿔줘야 한다.


