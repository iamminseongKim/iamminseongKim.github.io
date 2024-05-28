---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (25) Bean Validation - 스프링 적용
aliases: 
tags:
  - spring
  - validation
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-05-29
last_modified_at: 2024-05-29
---

>  인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.

## 프로젝트 준비 - V3

앞서 만든 기능을 유지하기 위해, 컨트롤러, 템플릿 파일을 복사함.

- `hello.itemservice.web.validation.ValidationItemControllerV2` 복사
- `hello.itemservice.web.validation.ValidationItemControllerV3` 붙여넣기
- URL 경로 변경: `validation/v2/` → `validation/v3/`

### 템플릿 파일 복사

- `/resources/templates/validation/v2/` 폴더 복사 →  `/resources/templates/validation/v3/`
	- addForm.html 
	- editForm.html 
	- item.html 
	- items.html


![](https://i.imgur.com/g7nBTrW.png){: .align-center}

v2 만들 때처럼 v3 폴더 클릭 후 `Ctrl + Shift + R` 누르면 모든 경로 한번에 바꿀 수 있다.


![](https://i.imgur.com/sY2klEj.png){: .align-center}

/v3/api  잘 된다.


## Bean Validation - 스프링 적용

`ValidationItemControllerV3` 코드 수정 
```java
@Controller  
@RequestMapping("/validation/v3/items")  
@RequiredArgsConstructor  
@Slf4j  
public class ValidationItemControllerV3 {  
  
    private final ItemRepository itemRepository;  
  
    @GetMapping  
    public String items(Model model) {  
        List<Item> items = itemRepository.findAll();  
        model.addAttribute("items", items);  
        return "validation/v3/items";  
    }  
  
    @GetMapping("/{itemId}")  
    public String item(@PathVariable long itemId, Model model) {  
        Item item = itemRepository.findById(itemId);  
        model.addAttribute("item", item);  
        return "validation/v3/item";  
    }  
  
    @GetMapping("/add")  
    public String addForm(Model model) {  
        model.addAttribute("item", new Item());  
        return "validation/v3/addForm";  
    }  
  
    @PostMapping("/add")  
    public String addItem(@Validated @ModelAttribute Item item, BindingResult bindingResult, RedirectAttributes redirectAttributes) {  
  
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
  
    @GetMapping("/{itemId}/edit")  
    public String editForm(@PathVariable Long itemId, Model model) {  
        Item item = itemRepository.findById(itemId);  
        model.addAttribute("item", item);  
        return "validation/v3/editForm";  
    }  
  
    @PostMapping("/{itemId}/edit")  
    public String edit(@PathVariable Long itemId, @ModelAttribute Item item) {  
        itemRepository.update(itemId, item);  
        return "redirect:/validation/v3/items/{itemId}";  
    }  
  
}
```

자 기존 V2 에 있던 검증 과정을 싹 지우고 만들었다.


```java
private final ItemValidator itemValidator; 

@InitBinder public void init(WebDataBinder dataBinder) { 
	log.info("init binder {}", dataBinder); 
	dataBinder.addValidators(itemValidator); 
}
```

이 코드를 삭제했다.

![](https://i.imgur.com/z51LNHx.png){: .align-center}

검증 과정이 다 빠졌는데 잘 된다..


```java
@Data  
public class Item {  
  
    private Long id;  
  
    @NotBlank  
    private String itemName;  
  
    @NotNull  
    @Range(min = 1000, max = 1000000)  
    private Integer price;  
  
    @NotNull  
    @Max(9999)  
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

Item 도메인에 **애노테이션 기반으로 validator**를 사용했기 때문!


```java
public String addItem(@Validated @ModelAttribute Item item,...) {   
   ...
}
```

`@Validated` 에노테이션도 필수다.

### 스프링 부트의 Bean Validator 

스프링 부트가 `spring-boot-starter-validation` 라이브러리를 넣으면 자동으로 Bean Validator를 인지하고 스프링에 통합한다.

**스프링 부트는 자동으로 글로벌 Validator로 등록한다.**

`LocalValidatorFactoryBean`을 글로벌 Validator로 등록한다.

이 Validator는 `@NotNull`같은 애노테이션을 보고 검증을 수행한다. 이렇게 글로벌 Validator가 적용되어 있기 때문에, `@Valid`, `@Validated`만 적용하면 된다.

검증 오류가 발생하면, `FieldError`, `ObjectError`를 생성해서 `BindingResult`에 담아준다.

#### 주의 !

다음과 같이 직접 글로벌 `Validator`를 직접 등록하면 스프링 부트는 Bean Validator를 글로벌 `Validator` 로 등록 하지 않는다. 

따라서 애노테이션 기반의 빈 검증기가 동작하지 않는다. 다음 부분은 제거하자.


```java
@SpringBootApplication 
public class ItemServiceApplication implements WebMvcConfigurer { 

	// 글로벌 검증기 추가 
	@Override public Validator getValidator() { 
		return new ItemValidator(); 
	} 
	// ... 
}
```


### 검증 순서

1. `@ModelAttribute` 각각의 필드에 타입 변환 시도
	1. 성공하면 다음
	2. 실패하면 `typeMismatch`로 `FieldError` 추가
2. Validator 적용

**바인딩에 성공한 필드만 Bean Validation 적용**

BeanValidator는 바인딩에 실패한 필드는 BeanValidator을 적용하지 않는다.

타입 변환에 성공해서 바인딩에 성공한 필드여야 BeanValidation 적용이 의미 있다.

(일단 모델 객체에 바인딩 받는 값이 정상으로 들어와야 검증도 의미가 있다.)

`@ModelAttibute` → 각각의 필드 타입 변환 시도 → 변환에 성공한 필드만 BeanValidation 적용

예)
- `itemName`에 문자 "A" 입력 → 타입 변환 성공 → `itemName` 필드에 BeanValidation 적용
- `price`에 문자 "A" 입력 → "A"를 숫자 타입 반환 시도 실패 → typeMismatch FeldError 추가 → `price`필드는 BeanValidation 적용 X 


