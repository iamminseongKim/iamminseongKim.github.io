---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (21) 검증 Validation - 오류 코드와 메시지 처리 2
aliases: 
tags:
  - spring
  - validation
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-05-23
last_modified_at: 2024-05-23
---
>  인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.

## 오류 코드와 메시지 처리 3

오류 코드를 만들 때 다음과 같이 자세히 만들 수도 있고, <br>`required.item.itemName` : 상품 이름은 필수 입니다.<br>`range.item.price` : 상품의 가격 범위 오류 입니다.


또는 다음과 같이 단순하게 만들 수도 있다.<br>`required` : 필수 값 입니다.<br>`range` : 범위 오류 입니다.

단순하게 만들면 범용성이 좋아서 여러 곳에서 사용 가능하지만, 메시지를 세밀하게 작성하기 어렵다.

반대로 너무 자세하게 만들면 범용성이 떨어진다.

가장 좋은 방법은 범용성으로 사용하다가, 세밀한게 필요할 때 새로 만들 수 있도록 **메시지에 단계를 두는 방법이다.**

예를 들어서 `required`라고 오류 코드를 사용한다고 가정해보자.
다음과 같이 `required`라는 메시지만 있으면 이 메시지를 선택해서 사용하는 것이다.

```
required: 필수 값 입니다. 
```

그런데 오류 메시지에 `required.item.itemName`와 같이 객체명과 필드명을 조합한 세밀한 메시지 코드가 있다면 

이 메시지를 높은 우선 순위로 사용하는 것이다.


```
#Level1 
required.item.itemName: 상품 이름은 필수 입니다. 

#Level2 
required: 필수 값 입니다.
```

물론 이렇게 객체명과 필드명을 조합한 메시지가 있는지 확인하고, 없으면 범용 메시지를 선택할 수 있도록 추가 개발을 해야겠지만, 범용성 있게 잘 개발해두면, 메시지 추가 만으로 매우 편리하게 오류 메시지를 관리할 수 있을 것이다.

스프링은 `MessageCodesResolver`라는 것으로 이러한 기능을 지원한다.


## 오류 코드와 메시지 처리 4

`MessageCodesResolver`를 테스트 코드를 통해 이해해 보자.

`validation/src/test/java/hello/itemservice/validation/MessageCodesResolverTest.java`
```java
public class MessageCodesResolverTest {  
  
    MessageCodesResolver codesResolver = new DefaultMessageCodesResolver();  
  
    @Test  
    void messageCodesResolverObject() {  
        String[] messageCodes = codesResolver.resolveMessageCodes("required", "item");  
        assertThat(messageCodes).containsExactly("required.item", "required");  
    }  
  
    @Test  
    void messageCodesResolverField() {  
        String[] messageCodes = codesResolver.resolveMessageCodes("required", "item", "itemName", String.class);  

        assertThat(messageCodes).containsExactly(  
                "required.item.itemName",  
                "required.itemName",  
                "required.java.lang.String",  
                "required");  
    }  
  
}
```

첫 번째 테스트는 

![](https://i.imgur.com/hn3aDL5.png){: .align-center}

다음과 같이 `required.item`, `required` 이렇게 두 개가 나온다.

두 번째 필드 오류 테스트는

```java
codesResolver.resolveMessageCodes("required", "item", "itemName", String.class);
```

이렇게 넣었으니 

- required.item.itemName
- required.itemName
- required.java.lang.String
- required

이 조합과 순서로 나올 것 이다.

![](https://i.imgur.com/zEyej5c.png){: .align-center}

`DefaultMessageCodesResolver`가 만드는 규칙 같은 건데 알아보자.

### MessageCodesResolver
- 검증 오류 코드로 메시지 코드들를 생성한다.
- `MessageCodesResolver`는 인터페이스 이고, `DefaultMessageCodesResolver`는 기본 구현체이다.
- 주로 다음과 함께 사용 `ObjectError`, `FieldError`


### DefaultMessageCodesResolver의 기본 메시지 생성 규칙

#### 객체 오류

```
객체 오류의 경우 다음 순서로 2가지 생성 
1.: code + "." + object name 
2.: code 

예) 오류 코드: required, object name: item 
1.: required.item 
2.: required
```


#### 필드 오류

```
필드 오류의 경우 다음 순서로 4가지 메시지 코드 생성 
1.: code + "." + object name + "." + field 
2.: code + "." + field 
3.: code + "." + field type 
4.: code 

예) 오류 코드: typeMismatch, object name "user", field "age", field type: int 
1. "typeMismatch.user.age" 
2. "typeMismatch.age" 
3. "typeMismatch.int" 
4. "typeMismatch"
```

#### 동작 방식

- `rejectValue()`, `reject()`는 내부에서 `MessageCodesResolver`를 사용한다. 여기에서 메시지 코드를생성함.
- `FieldError`, `ObjectError`의 생성자를 보면, 오류 코드를 하나가 아니라 여러 오류 코드를 가질 수 있다. `MessageCodesResolver`를 통해 생성된 순서대로 오류 코드 보관한다.


![](https://i.imgur.com/52VBsm4.png){: .align-center} 

`ObjectError` 는 `DefaultMessageSourceResolvable`를 상속 받고 있다. (`FieldError`는 `ObjectError`를 상속 받음. )

`FieldError : rejectValue("itemName", "required")` 

다음 4가지 오류 코드를 자동으로 생성

- `required.item.itemName`
- `required.itemName`
- `required.java.lang.String`
- `required`

`ObjectError : reject("totalPriceMin")`

다음 2가지 오류 코드를 자동으로 생성

- `totalPriceMin.item`
- `totalPriceMin`

#### 오류 메시지 출력

타임리프 화면을 렌더링 할 때 `th:errors`가 실행된다. 만약 이 때 오류가 있다면 생성된 오류 메시지 코드를 순서대로 돌아가면서 메시지를 찾는다. 

그리고 없으면 디폴트 메시지를 출력한다.