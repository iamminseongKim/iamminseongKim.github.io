---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (27) Bean Validation - 한계, groups
aliases: 
tags:
  - spring
  - validation
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-05-31
last_modified_at: 2024-05-31
---
>  인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.

## Bean Validation - 한계

기획자 : 데이터를 등록할 때와 수정할 때 요구 사항이 다를 수 있다..

**등록 시 요구 사항**

- 타입 검증
	- 가격, 수량에 문자가 들어가면 검증 오류 처리
- 필드 검증
	- 상품 명 : 필수, 공백X  
	- 가격 : 1000원 이상, 1백만원 이하
	- 수량 : 최대 9999
- 특정 필드의 범위를 넘어서는 검증
	- 가격 * 수량 >= 10,000

**수정 시 요구 사항**

- 등록 시에는 `quantity` 수량을 최대 9999까지 등록할 수 있지만 **수정 시에는 수량을 무제한**으로 변경할 수 있다.
- 등록 시에는 `id`에 값이 없어도 되지만, **수정 시에는 id 값이 필수**이다.


### 수정 요구 사항 적용


```java
@Data  
public class Item {  
  
    @NotNull  // 수정 요구 사항
    private Long id;  
  
    @NotBlank(message = "공백은 입력할 수 없습니다!!!!")  
    private String itemName;  
  
    @NotNull  
    @Range(min = 1000, max = 1000000)  
    private Integer price;  
  
    @NotNull  
    //@Max(9999)  // 수정 요구 사항
    private Integer quantity;  
  
    public Item() {  
    }  
    public Item(String itemName, Integer price, Integer quantity) {  
        this.itemName = itemName;  
        this.price = price;  
        this.quantity = quantity;  
    }  
}
```

- `id` : `@NotNull` 추가
- `quanity` : `@Max(9999)`제거

자 이렇게 수정 요구 사항을 적용했다. 

그런데 이러면 등록에서 요구 사항을 벗어나게 된다. 즉 사이드 이펙트가 발생했다..


결과적으로 `item`은 등록과 수정에서 검증 조건의 충돌이 발생하고, 등록과 수정은 같은 BeanValidation을 적용할 수 없다. 

이 문제를 어떻게 해결해야 하는가?

> **참고**<br>현재 구조에서는 수정 시 `item`의 `id`값은 항상 들어있도록 구성되어 있다. 그래서 검증하지 않아도 된다고 생각할 수 있다. 그런데 HTTP 요청은 언제든지 악의적으로 변경해서 요청할 수 있으므로 서버에서 항상 검증을 해야 한다. <br>예를 들어서 HTTP 요청을 변경해서 `item`의 `id`값을 삭제하고 요청할 수도 있다. <br>따라서 **최종 점검은 서버에서 진행하는 것이 안전하다.**



## Bean Validation - groups

동일한 모델 객체를 등록할 때와 수정할 때 각각 다르게 검증하는 방법을 알아보자.

**방법 2가지**

- BeanValidation의 groups 기능을 사용한다.
- Item을 직접 사용하지 않고, ItemSaveForm, ItemUpdateForm 같은 폼 전송을 위한 별도의 모델 객체를 만들어서 사용한다 (dto, 난 이게 맞는 것 같다..)

### BeanValidation groups 기능 사용

등록 시에 검증할 기능과 수정 시에 검증할 기능을 각각 그룹으로 나누어 적용할 수 있다.

**저장용 groups interface 생성**

```java
package hello.itemservice.domain.item;  
  
public interface SaveCheck {  
}
```

**수정용 groups interface 생성**

```java
package hello.itemservice.domain.item;  
  
public interface UpdateCheck {  
}
```

자 이렇게 만들고.. Item 도메인에 기능을 구분해 주면 된다.


```java
@Data  
public class Item {  
  
    @NotNull(groups = UpdateCheck.class)  
    private Long id;  
  
    @NotBlank(groups = {SaveCheck.class, UpdateCheck.class})  
    private String itemName;  
  
    @NotNull(groups = {SaveCheck.class, UpdateCheck.class})  
    @Range(min = 1000, max = 1000000)  
    private Integer price;  
  
    @NotNull(groups = {SaveCheck.class, UpdateCheck.class})  
    @Max(value = 9999, groups = SaveCheck.class)  
    private Integer quantity;  
  
    public Item() {  
    }  
    public Item(String itemName, Integer price, Integer quantity) {  
        this.itemName = itemName;  
        this.price = price;  
        this.quantity = quantity;  
    }  
}
```

자 이렇게 적용할 클래스(등록, 수정) 만  애노테이션에 적용하면 된다.

그리고 컨트롤러에 어떤 검증인지 넣어 줘야 한다.


```java
@PostMapping("/add")  
public String addItemV2(@Validated(SaveCheck.class) @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {  
  
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

@PostMapping("/{itemId}/edit")  
public String editV2(@PathVariable Long itemId, @Validated(UpdateCheck.class) @ModelAttribute Item item, BindingResult bindingResult) {  
  
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


```java
// 등록
@Validated(SaveCheck.class) @ModelAttribute Item item

// 수정
@Validated(UpdateCheck.class) @ModelAttribute Item item
```

다음 코드와 같이 add 이기 때문에 SaveCheck.class 를 등록해서 저장용 검증만 실행하도록 지정해 준 것 이다.  

수정도 마찬가지 이다.



![](https://i.imgur.com/uCVjPes.png){: .align-center}

실제 등록 때는 9,999개 까지만 등록 되지만


![](https://i.imgur.com/FtSqg1T.png){: .align-center}

수정에는 99999도 가능하다.

groups 기능을 사용해서 등록과 수정 시에 각각 다르게 검증을 할 수 있었다. 

그런데 groups 기능을 사용하니 Item 은 물론이고, 전반적으로 복잡도가 올라갔다. 

사실 groups 기능은 실제 잘 사용되지는 않는데, 그 이유는 실무에서는 주로 등록용 폼 객체와 수정용 폼 객체를 분리해서 사용하기 때문이다

