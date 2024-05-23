---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (22) 검증 Validation - 오류 코드와 메시지 처리 3
aliases: 
tags:
  - spring
  - validation
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-05-24
last_modified_at: 2024-05-24
---
>  인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.


## 오류 코드와 메시지 처리5

### 오류 코드 관리 전략

**핵심은 구체적인 것에서! 덜 구체적인 것으로!**

`MessageCodeResolver`는 `required.item.itemName`처럼 구체적인 것 먼저 만들고, 마지막에 `required`처럼 덜 구체적인 걸 만듬.

이렇게 하면 메시지와 관련된 공통 전략을 편리하게 도입할 수 있다.

**왜 이렇게 복잡하게 사용하는가?**

모든 오류 코드에 대해서 메시지를 각각 다 정의하면 개발자 입장에서 관리하기 너무 힘들다.

크게 중요하지 않은 메시지는 범용성 있는 `requried`같은 메시지로 끝내고, 

정말 중요한 메시지는 꼭 필요할 때 구체적으로 적어서 사용하는 방식이 더 효과적이다.

`errors.properties`
```
#==ObjectError==  
#Level1  
totalPriceMin.item=상품의 가격 * 수량의 합은 {0}원 이상이어야 합니다. 현재 값 = {1}  

#Level2 - 생략  
totalPriceMin=전체 가격은 {0}원 이상이어야 합니다. 현재 값 = {1} 

#==FieldError==  
#Level1  
required.item.itemName=상품 이름은 필수입니다.  
range.item.price=가격은 {0} ~ {1} 까지 허용합니다.  
max.item.quantity=수량은 최대 {0} 까지 허용합니다.  
  
#Level2 - 생략  
  
#Level3  
required.java.lang.String = 필수 문자입니다.  
required.java.lang.Integer = 필수 숫자입니다.  
min.java.lang.String = {0} 이상의 문자를 입력해주세요.  
min.java.lang.Integer = {0} 이상의 숫자를 입력해주세요.  
range.java.lang.String = {0} ~ {1} 까지의 문자를 입력해주세요.  
range.java.lang.Integer = {0} ~ {1} 까지의 숫자를 입력해주세요.  
max.java.lang.String = {0} 까지의 문자를 허용합니다.  
max.java.lang.Integer = {0} 까지의 숫자를 허용합니다.  
  
#Level4  
required = 필수 값 입니다.  
min= {0} 이상이어야 합니다.  
range= {0} ~ {1} 범위를 허용합니다.  
max= {0} 까지 허용합니다.
```

크게 객체 오류와 필드 오류로 나누고,  그 안에서 범용성에 따라 레벨을 나누었다.

레벨이 높을수록 범용성이 높다. `MessageCodeResolver`는 레벨 1 부터 만들 것이다.


여기서 레벨 1을 주석 처리 해버리면.. 

![](https://i.imgur.com/EuLxZgg.png){: .align-center}


```
#Level3  
required.java.lang.String = 필수 문자입니다.  
required.java.lang.Integer = 필수 숫자입니다.  
min.java.lang.String = {0} 이상의 문자를 입력해주세요.  
min.java.lang.Integer = {0} 이상의 숫자를 입력해주세요.  
range.java.lang.String = {0} ~ {1} 까지의 문자를 입력해주세요.  
range.java.lang.Integer = {0} ~ {1} 까지의 숫자를 입력해주세요.  
max.java.lang.String = {0} 까지의 문자를 허용합니다.  
max.java.lang.Integer = {0} 까지의 숫자를 허용합니다.  
```

이게(레벨 2) 나와버렸다! 그럼 레벨 3도 주석 처리하면??


![](https://i.imgur.com/2su2rgW.png){: .align-center}

가장 범용적인 레벨 4가 나와버렸다. 

애플리케이션 코드를 하나도 건드리지 않고 문구를 바꿔버렸다!

구체적인 것에서 덜 구체적인 순서대로 찾는다. 

메시지에 1번이 없으면 2번을 찾고, 2번이 없으면 3번을 찾는다. 이렇게 되면 만약에 크게 중요하지 않은 오류 메시지는 기존에 정의된 것을 그냥 **재활용** 하면 된다!

### ValidationUtils

`ValidationUtils` 사용 전
```java
if (!StringUtils.hasText(item.getItemName())) { 
	bindingResult.rejectValue("itemName", "required", "기본: 상품 이름은 필수입니다."); 
}
```

`ValidationUtils` 사용 후
```java
ValidationUtils.rejectIfEmptyOrWhitespace(bindingResult, "itemName", "required");
```

다음과 같이 한 줄로 가능, 제공하는 기능은 `Empty`, `공백` 같은 단순한 기능만 제공


### 정리

1. `rejectValue()` 호출
2. `MessageCodesResolver`를 사용해서 검증 오류 코드로 메시지 코드들을 생성
3. `new FieldError()`를 생성하면서 메시지 코드들을 보관
4. `th:errors`에서 메시지 코드들로 메시지를 순서대로 메시지에서 찾고 노출




## 오류 코드와 메시지 처리6

### 스프링이 직접 만든 오류 메시지 처리

지금까지 배운 검증 오류 코드는 다음과 같이 2가지로 나눌 수 있다.
- 개발자가 직접 설정한 오류 코드 → `rejectVaule()`를 직접 호출
- 스프링이 직접 검증 오류에 추가한 경우 (주로 타입 정보가 안 맞을 때)

![](https://i.imgur.com/OqX82aa.png){: .align-center}

가격 `integer` 필드에 문자를 넣어 버리면 다음과 같이 오류가 나와버린다.

![](https://i.imgur.com/2T8gDLX.png){: .align-center}

로그에서도 내가 안 만든 로그가 나온다.

`codes[typeMismatch.item.price,typeMismatch.price,typeMismatch.java.lang.Integer,ty peMismatch]`

다음과 같이 4가지 메시지 코드가 입력되어 있다.


- `typeMismatch.item.price`
- `typeMismatch.price` 
- `typeMismatch.java.lang.Integer` 
- `typeMismatch`

자 이것도 딱 보니깐 **범용성 순서** 가 느껴진다.. `MessageCodesResolver`를 쓴 것 같다!


`error.properties` 에 다음 내용을 추가하자  
```
typeMismatch.java.lang.Integer=숫자를 입력해주세요. 
typeMismatch=타입 오류입니다.
```

![](https://i.imgur.com/a8HKB92.png){: .align-center}

바뀌었다...   코드를 수정을 안 했다.. 그저 프로퍼티에 추가한 것 뿐.

결국 스프링도 `MessageCodesResolver`를 사용해서 만든 것 이었다.




