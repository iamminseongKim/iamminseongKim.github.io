---
title: 스프링 MVC 2편 - 백엔드 웹 개발 활용 기술 - (28) Bean Validation - Form 전송 객체 분리
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

## 프로젝트 준비 V4 

`ValidationItemControllerV4` 컨트롤러 생성 

- `hello.itemservice.web.validation.ValidationItemControllerV3` 복사
- `hello.itemservice.web.validation.ValidationItemControllerV4` 붙여넣기
- URL 경로 변경 : `validation/v3/` → `validation/v4/`


### 템플릿 파일 복사

![](https://i.imgur.com/Y2zZjTE.png){: .align-center}  

v3 폴더 복사

![](https://i.imgur.com/SFqeuJ8.png){: .align-center}

v4로 붙여넣기

v4 폴더 클릭 후 `Ctrl + Shift + R`

![](https://i.imgur.com/u6BcFu6.png){: .align-center}

- URL 경로 변경 : `validation/v3/` → `validation/v4/`


![](https://i.imgur.com/S7gpOg0.png){: .align-center}

/vaidation/v4/items 접근 시 잘 된다.


## Form 전송 객체 분리

실무에서는 `groups`를 잘 사용하지 않는다. 

그 이유는 폼에서 전달하는 데이터가 `Item`도메인 객체와 딱 맞지 않기 때문이다.

실무에서 회원 등록 시 회원과 관련된 데이터만 넘어 오는게 아니라, 약관 정보 등 추가적인 정보도 같이 들어온다. 

그래서 보통 `Item`을 직접 전달 받는 것이 아니라, 복잡한 폼의 데이터를 컨트롤러까지 전달할 별도의 객체를 만들어서 전달한다. 

예를 들면 `ItemSaveForm`이라는 폼을 전달 받는 전용 객체를 만들어서 `@ModelAttribute`로 사용한다.

이것을 통해 컨트롤러에서 폼 데이터를 전달 받고, 이후 컨트롤러에서 필요한 데이터를 사용해서 `Item`을 생성한다.


다음 두 가지 과정을 보자.

**폼 데이터 전달에 Item 도메인 객체 사용**

- `HTML Form → Item → Controller → Item → Repository`
	- 장점 : Item 도메인 객체를 컨트롤러, 리포지토리까지 직접 전달해서 중간에 Item을 만드는 과정이 없어서 간단하다.
	- 단점 : 간단한 경우에만 적용할 수 있다. 수정 시 검증이 중복 될 수 있고, groups를 사용해야 한다.

**폼 데이터 전달을 위한 별도의 객체 사용**

- `HTML Form → ItemSaveForm → Controller → Item 생성 → Repository `
	- 장점 : 전송하는 폼 데이터가 복잡해도 거기에 맞춘 별도의 폼 객체를 사용해서 데이터를 전달 받을 수 있다. 보통 등록과, 수정 용으로 별도의 폼 객체를 만들기 때문에 검증이 중복되지 않는다.
	- 단점 : 폼 데이터를 기반으로 컨트롤러에서 Item 객체를 생성하는 변환 과정이 추가된다.


생각해 보자. 수정의 경우 등록과 다른 데이터들이 넘어온다. 회원 가입 시 다루는 데이터와 수정 시 다루는 데이터는 범위에 차이가 있다. 

예를 들면 등록 시에는 로그인 id, 주민번호 등을 받지만, 수정 시에는 이런 부분이 빠진다. 

그리고 검증 로직도 많이 달라진다. 

그래서 `ItemUpdateForm`이라는 별도의 **객체**로 데이터를 전달 받는 것이 좋다.


> **Q. 이름은 어떻게 지어야 하는가?**<br>이름은 의미 있게 지으면 된다. `ItemSave`라 해도 되고, `ItemSaveForm`, `ItemSaveDto` 등으로 사용해도 된다. <br> 중요한 것은 **일관성**이다.

<br>

> **Q. 등록, 수정용 뷰 템플릿이 비슷한데 합쳐도 될까?**<br>한 페이지에 등록과 수정을 어설프게 합치면 수 많은 분기문 때문에 나중에 유지 보수가 힘들다.<br>어설픈 분기문이 보이기 시작하면 분리 해야 한다는 신호다.