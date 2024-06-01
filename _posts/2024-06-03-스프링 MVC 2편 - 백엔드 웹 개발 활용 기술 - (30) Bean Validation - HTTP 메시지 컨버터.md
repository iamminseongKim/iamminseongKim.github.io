---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (30) Bean Validation - HTTP 메시지 컨버터
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


`@Valid`, `@Validated`는 `HttpMessageConverter`(`@RequestBody`)에도 적용할 수 있다.

> **참고**<br>`@ModelAttribute`는 HTTP 요청 파라미터(URL 쿼리 스트링, POST Form)를 다룰 때 사용한다.<br>`@RequestBody`는 HTTP Body의 데이터를 객체로 변환할 때 사용한다. 주로 API JSON 요청을 다룰 때 사용한다.


### ValidationItemApiController 생성 

```java
@Slf4j  
@RestController  
@RequestMapping("/validation/api/items")  
public class ValidationItemApiController {  
  
    @PostMapping("/add")  
    public Object addForm(@RequestBody @Validated ItemSaveForm form, BindingResult bindingResult) {  
  
        log.info("API 컨트롤러 호출");  
  
        if (bindingResult.hasErrors()) {  
            log.info("검증 오류 발생 errors={}", bindingResult);  
            return bindingResult.getAllErrors();  
        }  
  
        log.info("성공 로직 실행");  
        return form;  
    }  
  
}
```

자 이렇게 만들었고 간단하게 흐름을 보도록 로그를 남겼다.

그리고 먼저 성공할 데이터를 포스트맨으로 넘겨 봤다.

![](https://i.imgur.com/u1n2XCU.png){: .align-center}

자 잘 된다.  이제 실패 데이터를 넣어보자.

![](https://i.imgur.com/SUfH7gb.png){: .align-center}

가격에 문자를 넣어서 실패를 유도했다.

보이는가? `400 에러`가 났는데, 컨트롤러에 로그가 남지도 않고 **그냥 컨트롤러 오지도 않고 실패 돼버렸다**.


**API의 경우 3가지 경우를 나누어 생각해야 한다.**

- 성공 요청 : 성공
- 실패 요청 : JSON을 객체로 생성하는 것 자체를 실패함
- 검증 오류 요청 : JSON을 객체로 생성하는 것은 성공했고, 검증에서 실패함.


자 그럼 이번엔 검증 오류에 걸려보도록 요청해보자.


```
{
    "itemName" : "hello",
    "price" : 1000,
    "quantity" : 10000
}
```

다음과 같이 수량 최대 수 9999에 걸리도록 10000을 요청했다.

![](https://i.imgur.com/hpZKRYs.png){: .align-center}

자 이번엔 컨트롤러를 탔고, 다양한 오류가 나왔다. (`bindingResult.getAllErrors()` 로 그냥 다 출력함.)


```json
[
    {
        "codes": [
            "Max.itemSaveForm.quantity",
            "Max.quantity",
            "Max.java.lang.Integer",
            "Max"
        ],
        "arguments": [
            {
                "codes": [
                    "itemSaveForm.quantity",
                    "quantity"
                ],
                "arguments": null,
                "defaultMessage": "quantity",
                "code": "quantity"
            },
            9999
        ],
        "defaultMessage": "9999 이하여야 합니다",
        "objectName": "itemSaveForm",
        "field": "quantity",
        "rejectedValue": 10000,
        "bindingFailure": false,
        "code": "Max"
    }
]
```

실제로 이렇게 다 출력하면 안되고, 내가 필요한 스펙에 맞게 메시지를 바꿔서 리턴하자.


### @ModelAttribute vs @RequestBody

HTTP 요청 파라미터를 처리하는 `@ModelAttribute`는 각각의 필드 단위로 세밀하게 적용된다. 그래서 특정 필드에 타입이 맞지 않는 오류가 발생해도 나머지 필드는 정상 처리할 수 있다.

`HttpMessageConverter`는 `@ModelAttribute`와 다르게 각각의 필드 단위로 적용되는 것이 아니라, 전체 객체 단위로 적용된다.

따라서 메시지 컨버터의 작동이 성공해서 `ItemSaveForm` 객체를 만들어야 `@Valid` `@Validated`가 적용된다.

- `@ModelAttribute`는 필드 단위로 정교하게 바인딩이 적용된다. 특정 필드가 바인딩 되지 않아도 나머지 필드는 정상 바인딩 되고, Validator를 통한 검증도 적용할 수 있다.
- `@RequestBody`는 **HttpMessageConverter 단계에서 JSON데이터를 객체로 변경하지 못하면 이후 단계 자체가 진행되지 않고 예외가 발생한다.** 컨트롤러가 호출되지도 않고, Validator도 적용할 수 없다.

참고로 `@RequestBody`단계에서 바인딩 에러로 발생한 예외 문구는 커스텀텀이 가능하다. 