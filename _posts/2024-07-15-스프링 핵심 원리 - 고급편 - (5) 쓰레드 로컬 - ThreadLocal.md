---
title: 스프링 핵심 원리 - 고급편 - (5) 쓰레드 로컬 - ThreadLocal
aliases: 
tags:
  - spring
  - thread
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-07-15
last_modified_at: 2024-07-15
---

>  인프런 스프링 핵심 원리 - 고급편을 학습하고 정리한 내용 입니다.

## ThreadLocal - 소개

쓰레드 로컬은 해당 쓰레드만 접근할 수 있는 특별한 저장소를 말한다. 쉽게 이야기 해서 물건 보관 창구를 떠올리면 된다.

여러 사람이 같은 물건 보관 창구를 사용하더라도 창구 직원은 사용자를 인색해서 사용자별로 확실하게 물건을 구분해 준다.

사용자A, 사용자B 모두 창구 직원을 통해 물건을 보관하고, 꺼내지만 창구 직원이 사용자에 따라 물건을 구분해 주는 것이다.

### 일반적인 변수 필드

여러 쓰레드가 같은 인스턴스의 필드에 접근하면 처음 쓰레드가 보관한 데이터가 사라질 수 있다.

![](https://i.imgur.com/k5ifx0O.png){: .align-center}

`thread-A`가 `userA`라는 값을 저장하고

![](https://i.imgur.com/plCCOSu.png){: .align-center}

`thread-B`가 `userB`라는 값을 저장하면 직전에 `thread-A`가 저장한 `userA`값은 사라진다.


### 쓰레드 로컬

쓰레드 로컬을 사용하면 각 쓰레드마다 별도의 내부 저장소를 제공한다. 따라서 같은 인스턴스의 쓰레드 로컬 필드에 접근해도 문제 없다.

![](https://i.imgur.com/fMB3uzm.png){: .align-center}

`thread-A`가 `userA`라는 값을 저장하면 쓰레드 로컬은 `thread-A` 전용 보관소에 데이터를 안전하게 보관한다.

![](https://i.imgur.com/81fIteG.png){: .align-center}

`thread-B`가 `userB`라는 값을 저장하면 쓰레드 로컬은 `thread-B` 전용 보관소에 데이터를 안전하게 보관한다.

![](https://i.imgur.com/zh9s50N.png){: .align-center}

쓰레드 로컬을 통해서 데이터를 조회할 때도 `thread-A`가 조회하면 쓰레드 로컬은 `thread-A`전용 보관소에서 `userA`데이터를 반환해준다. 물론 `thread-B`가 조회하면 `thread-B`전용 보관소에서 `userB`데이터를 반환해 준다.

자바는 언어 차원에서 쓰레드 로컬을 지원하기 위한 `java.lang.ThreadLocal`클래스를 제공한다.


## ThreadLocal - 예제 코드

예제 코드를 통해서 `ThreadLocal`을 학습해보자.

![](https://i.imgur.com/TEw9NRu.png){: .align-center}

해당 위치에 `ThreadLocalService`를 만들어서 테스트 해보자.


```java
@Slf4j  
public class TheadLocalService {  
  
    private ThreadLocal<String> nameStore = new ThreadLocal<>();  
  
    public String logic(String name) {  
  
        log.info("저장 name={} --> nameStore={}", name, nameStore.get());  
        nameStore.set(name);  
        sleep(1000);  
        log.info("조회 nameStore={}", nameStore.get());  
        return nameStore.get();  
    }  
  
    private void sleep(long millis) {  
        try {  
            Thread.sleep(millis);  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
    }  
}
```

기존에 있던 `FieldService`와 거의 같은 코드인데, `nameStore`필드가 일반 `String`타입에서 `ThreadLocal`을 사용하도록 변경되었다

**ThreadLocal 사용법**
- 값 저장 : `ThreadLocal.set(xxx)`
- 값 조회 : `ThreadLocal.get()`
- 값 제거 : `ThreadLocal.remove()`

> **주의**<br>해당 쓰레드가 쓰레드 로컬을 모두 사용하고 나면 `ThreadLocal.remove()`를 호출해서 쓰레드 로컬에 저장된 값을 제거해주어야 한다. 


이제 테스트 코드를 작성해 보자.

### ThreadLocalServiceTest


```java
@Slf4j  
class ThreadLocalServiceTest {  
    private ThreadLocalService service = new ThreadLocalService();  
  
    @Test  
    void field() {  
        log.info("main start");  
        Runnable userA = () -> {  
            service.logic("userA");  
        };  
  
        Runnable userB = () -> {  
            service.logic("userB");  
        };  
  
        Thread threadA = new Thread(userA);  
        threadA.setName("thread-A");  
  
        Thread threadB = new Thread(userB);  
        threadB.setName("thread-B");  
  
        threadA.start();  
        //sleep(2000); // 동시성 문제가 발생 안하는 코드  
        sleep(100);    // 동시성 문제가 발생하는 코드
        threadB.start();  
  
        sleep(3000); // 메인 쓰레드 종료 대기  
        log.info("main exit");  
    }  
  
    private void sleep(long millis) {  
        try {  
            Thread.sleep(millis);  
        } catch (InterruptedException e) {  
            e.printStackTrace();  
        }  
    }  
}
```


지금같이 A와 B 사이 간격을 0.1초로 두고 테스트를 해보았다.

![](https://i.imgur.com/FTyGj3A.png){: .align-center}

이제 각각의 데이터 저장소에 저장하는 모습을 볼 수 있다. 

userB가 처음 저장할 때 nameStore에 값이 없다. userB의 nameStore는 userA와 다른 nameStore이기 때문.

결과적으로 동시성 문제가 해결 되었다.



