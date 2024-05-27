---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (24) Bean Validation
aliases: 
tags:
  - spring
  - validation
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-05-28
last_modified_at: 2024-05-28
---

>  인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.

## Bean Validation - 소개

검증 기능을 매번 코드로 작성하는 것은 상당히 번거롭다.

특히 특정 필드에 대한 검증 로직은 대부분 빈 값인지 아닌지, 특정 크기를 넘는지 아닌지와 같이 매우 일반적인 로직이다.

다음 코드를 보자.

```java
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
    //...  
}
```

이런 검증 로직을 모든 프로젝트에 적용할 수 있게 공통화하고, 표준화 한 것이 바로 `Bean Validation`이다.

`Bean Validation`을 잘 활용하면, 애노테이션 하나로 검증 로직을 매우 편리하게 적용할 수 있다.

### Bean Validation 이란?
`Bean Validation`은 특정한 구현체가 아니라 Bean Validation 2.0(JSR-380)이라는 기술 표준이다.

쉽게 이야기 해서 검증 애노테이션과 여러 인터페이스의 모음이다. (JPA 표준기술 -> 구현체 하이버네이트 같은 느낌)

Bean Validation을 구현한 기술중에 일반적으로 사용하는 구현체는 하이버네이트 Validator이다. 이름이 하이버네이트가 붙어서 그렇지 `ORM`과는 관련이 없다.

#### 하이버네이트 Validator 관련 링크

- [공식 사이트](https://hibernate.org/validator/)
- [공식 메뉴얼](https://docs.jboss.org/hibernate/validator/6.2/reference/en-US/html_single/)
- [검증 애노테이션 모음](https://docs.jboss.org/hibernate/validator/6.2/reference/en-US/html_single/#validator-defineconstraints-spec)


## Bean Validation - 시작

Bean Validation 기능을 어떻게 사용하는지 코드로 알아보자. 

먼저 스프링 말고, 순수하게 Bean Validation을 사용해보기 위해 테스트 코드로 알아보자.

### Bean Validation 의존관계 추가

Bean Validation을 사용하려면 다음 의존 관계를 추가해야 한다.

`build.gradle`
```
implementation 'org.springframework.boot:spring-boot-starter-validation'
```


먼저 `Item` 도메인 클래스에 Bean Validation을 적용하자.

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

**검증 애노테이션**
- `@NotBlank` : 빈 값 + 공백만 있는 경우를 허용하지 않는다.
- `@NotNull` : `null`을 허용하지 않는다.
- `@Range(min = 1000, max = 1000000)` : 이 범위 안에 값이어야 함.
- `@Max(9999)` : 최대 9999까지만 허용

>**참고**<br>`jakarta.validation.constraints.NotNull`<br>`org.hibernate.validator.constraints.Range`<br><br>`jakarta.validation`으로 시작하면 특정 구현체와 관계 없이 제공되는 표준 인터페이스이고, <br>`org.hibernate.validator`로 시작하면 하이버네이트 validator 구현체를 사용할 때만 제공되는 검증 기능이다.<br>실무에서 대부분 하이버네이트 validator를 사용하므로 크게 상관 없긴 하다.


이제 테스트 코드를 작성해 보자.

### Bean Validation 테스트 

![](https://i.imgur.com/mQhFSne.png){: .align-center}


```java
public class BeanValidationTest {  

    @Test  
    void beanValidationTest() throws Exception {  
  
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();  
        Validator validator = factory.getValidator();  
  
        Item item = new Item();  
        item.setItemName(" "); // 공백  
        item.setPrice(0);  
        item.setQuantity(10000);  
  
        Set<ConstraintViolation<Item>> validations = validator.validate(item);  
  
        for (ConstraintViolation<Item> validation : validations) {  
            System.out.println("validation = " + validation);  
            System.out.println("validation.getMessage() = " + validation.getMessage());  
        }  
    }  
}
```

**검증기 생성**

```java
ValidatorFactory factory = Validation.buildDefaultValidatorFactory();  
Validator validator = factory.getValidator();  
```

다음 코드와 같이 검증기를 생성한다. 스프링과 통합하면 우리가 직접 코드를 작성하지 않으므로 참고만 하자.

**검증 실행**

```java
Set<ConstraintViolation<Item>> validations = validator.validate(item);  
```

검증 대상(`item`)을 직접 검증기에 넣고 그 결과를 받는다. 

`Set`에는 `ConstraintViolation`이라는 검증 오류가 담긴다.

따라서 결과가 비어있으면 검증 오류가 없는 것이다.


실행 결과를 보자.

![](https://i.imgur.com/KpqiUY8.png){: .align-center}


```
08:01:07.682 [main] INFO org.hibernate.validator.internal.util.Version -- HV000001: Hibernate Validator 8.0.1.Final

validation = ConstraintViolationImpl{interpolatedMessage='공백일 수 없습니다', propertyPath=itemName, rootBeanClass=class hello.itemservice.domain.item.Item, messageTemplate='{jakarta.validation.constraints.NotBlank.message}'}
validation.getMessage() = 공백일 수 없습니다

validation = ConstraintViolationImpl{interpolatedMessage='9999 이하여야 합니다', propertyPath=quantity, rootBeanClass=class hello.itemservice.domain.item.Item, messageTemplate='{jakarta.validation.constraints.Max.message}'}
validation.getMessage() = 9999 이하여야 합니다

validation = ConstraintViolationImpl{interpolatedMessage='1000에서 1000000 사이여야 합니다', propertyPath=price, rootBeanClass=class hello.itemservice.domain.item.Item, messageTemplate='{org.hibernate.validator.constraints.Range.message}'}
validation.getMessage() = 1000에서 1000000 사이여야 합니다
```

`ConstraintViolation`출력 결과를 보면, 검증 오류가 발생한 객체, 필드, 메시지 정보 등 다양한 정보를 확인할 수 있다.


