---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (53) 스프링 부트 - 스프링 타입 컨버터 - 컨버전 서비스 - ConversionService
aliases: 
tags:
  - spring
  - typeConverter
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-06-28
last_modified_at: 2024-06-28
---

> 인프런 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술편을 학습하고 정리한 내용 입니다.

컨버터를 열심히 만들었는데, 하나하나 직접 찾아서 타입 변환에 사용하는 것은 매우 불편하다.

그래서 스프링은 개별 컨버터를 모아두고 그것들을 편리하게 사용할 수 있는 기능을 제공하는데, 이것이 바로 컨버전 서비스(`ConversionService`)다. 

```java
public interface ConversionService {  
    boolean canConvert(@Nullable Class<?> sourceType, Class<?> targetType);  
  
    boolean canConvert(@Nullable TypeDescriptor sourceType, TypeDescriptor targetType);  
  
    @Nullable  
    <T> T convert(@Nullable Object source, Class<T> targetType);  
  
    @Nullable  
    default Object convert(@Nullable Object source, TypeDescriptor targetType) {  
        return this.convert(source, TypeDescriptor.forObject(source), targetType);  
    }  
  
    @Nullable  
    Object convert(@Nullable Object source, @Nullable TypeDescriptor sourceType, TypeDescriptor targetType);  
}
```

인터페이스인데 컨버팅이 가능한가? 컨버팅 기능 이 있는 것 같다.

코드로 확인하자.

### ConversionServiceTest - 컨버전 서비스 테스트 코드 

```java
public class ConversionServiceTest {  
  
    @Test  
    void conversionService() {  
  
        // 등록  
        DefaultConversionService conversionService = new DefaultConversionService();  
        conversionService.addConverter(new StringToIntegerConverter());  
        conversionService.addConverter(new IntegerToStringConverter());  
        conversionService.addConverter(new StringToIpPortConverter());  
        conversionService.addConverter(new IpPortToStringConverter());  
  
        // 사용  
        assertThat(conversionService.convert("10", Integer.class)).isEqualTo(10);  
        assertThat(conversionService.convert(10, String.class)).isEqualTo("10");  
  
        IpPort ipPort = conversionService.convert("127.0.0.1:8080", IpPort.class);  
        assertThat(ipPort).isEqualTo(new IpPort("127.0.0.1", 8080));  
  
        String ipPortString = conversionService.convert(new IpPort("127.0.0.1", 8080), String.class);  
        assertThat(ipPortString).isEqualTo("127.0.0.1:8080");  
  
    }  
}
```

`DefaultConversionService`는 `ConversionService`인터페이스를 구현했는데, 추가로 컨버터를 등록하는 기능도 제공한다.

![](https://i.imgur.com/RfqTek5.png){: .align-center}

입 출력만 잘 정해주니 알아서 컨버터들을 잘 찾아서 사용했다.


### 등록과 사용 분리

컨버터를 등록할 때는 `StringToIntegerConverter`같은 타입 컨버터를 명확하게 알아야 한다.

반면에 컨버터를 사용하는 입장에서는 타입 컨버터를 전혀 몰라도 된다.

타입 컨버터들은 모두 컨버전 서비스 내부에 숨어서 제공된다.

따라서 타입 변환을 원하는 사용자는 컨버전 서비스 인터페이스에만 의존하면 된다.  

물론 컨버전 서비스를 등록하는 부분과 사용하는 부분을 분리하고 의존 관계 주입을 사용해야 한다.

#### 컨버전 서비스 사용

```java
Integer value = conversionService.convert("10", Integer.class)
```

서비스 안에 어떤 게 있는지 모르지만 문자열을 넣을 테니 숫자를 줘라. 


#### 인터페이스 분리 원칙 - ISP(Interface Segregation Principle)

인터페이스 분리 원칙은 클라이언트가 자신이 이용하지 않는 메서드에 의존하지 않아야 한다.

`DefaultConversionService`는 다음 두 인터페이스를 구현했다.
- `ConversionService` : 컨버터 사용에 초점
- `ConverterRegistry` : 컨버터 등록에 초점

이렇게 인터페이스를 분리하면 컨버터를 사용하는 클라이언트와 컨버터를 등록하고 관리하는 클라이언트의 관심사를 명확하게 분리할 수 있다.

특히 컨버터를 사용하는 클라이언트는 `ConversionService`만 의존하면 되므로, 컨버터를 어떻게 등록하고 관리하는지 전혀 몰라도 된다.

결과적으로 컨버터를 사용하는 클라이언트는 꼭 필요한 메서드만 알게 된다.

이렇게 인터페이스를 분리하는 것을 `ISP`라 한다.

> 참고 : [ISP](https://ko.wikipedia.org/wiki/%EC%9D%B8%ED%84%B0%ED%8E%98%EC%9D%B4%EC%8A%A4_%EB%B6%84%EB%A6%AC_%EC%9B%90%EC%B9%99)

스프링은 내부에서 `ConversionService`를 사용해서 타입을 변환한다. 예를 들어서 `@RequestParam`같은 곳에서 이 기능을 사용해서 타입을 변환한다.


## 스프링에 Converter 적용하기

### WebConfig - 컨버터 등록

`hello.typeconverter.WebConfig`
```java
@Configuration  
public class WebConfig implements WebMvcConfigurer {  
  
    @Override  
    public void addFormatters(FormatterRegistry registry) {  
        registry.addConverter(new StringToIntegerConverter());  
        registry.addConverter(new IntegerToStringConverter());  
        registry.addConverter(new StringToIpPortConverter());  
        registry.addConverter(new IpPortToStringConverter());  
    }  
}
```

스프링은 내부에서 `ConversionService`를 제공한다.

우리는 `WebMvcConfigurer`가 제공하는 `addConverter()`를 사용해서 추가하고 싶은 컨버터를 등록하면 된다.

이렇게 하면 스프링은 내부에서 사용하는 `ConversionService`에 컨버터들을 추가해 준다.

### 컨트롤러로 테스트


```java
@GetMapping("/hello-v2")  
public String helloV2(@RequestParam(name = "data") Integer data) {  
    System.out.println("data = " + data);  
    return "ok";  
}
```

자 이제 문자를 넣고 Integer로 받을 때 어떤 일이 벌어지는지 확인해 보자.

![](https://i.imgur.com/ZYbp13k.png){: .align-center}

내가 등록한 `StringToIntegerConvertor` 가 실행되었다.

근데 사실 문자를 숫자로 바꾸는 컨버터는 기존에도 스프링 내부에 있었다. 

내가 구현해서 등록한 컨버터가 우선순위가 더 높아지는 걸 알 수 있다.

그럼 IpPort 컨버터를 테스트 해보자.


```java
@GetMapping("/ip-port")  
public String ipPort(@RequestParam(name = "ip-port") IpPort ipPort) {  
    System.out.println("ipPort IP = " + ipPort.getIp());  
    System.out.println("ipPort Port = " + ipPort.getPort());  
    return "ok";  
}
```

http://localhost:8080/ip-port?ip-port=127.0.0.1:8080 요청

![](https://i.imgur.com/0xDZ7HS.png){: .align-center}

잘 동작한다.


### 처리 과정

`@RequestParam`은 

`@RequestParam`을 처리하는 

`ArgumentResolver`인 `RequstParamMethodArgumentResolver`에서 

`ConversionService`를 사용해서 타입을 반환한다.

![](https://i.imgur.com/9TcgRj6.png){: .align-center}

`RequstParamMethodArgumentResolver` 내부에 디버그 걸어 놓으면  흐름을 볼 수 있다.



