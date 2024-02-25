---
title: 스프링 MVC - 1편 - HTML, HTTP API, CSR, SSR
aliases: 
tags:
  - spring
categories:
  - spring
toc: true
toc_label: 목차
date: 2024-02-25
last_modified_at: 2024-02-25
---
>  인프런 스프링 MVC 1편 - 백엔드 웹 개발 핵심 기술편을 학습하고 정리한 내용 입니다.

### 정적 리소스

- 고정된 HTML 파일, CSS, JS, 이미지, 영상 등을 제공
- 주로 웹 브라우저

![](https://i.imgur.com/9Ms5Qfp.png)
{: .align-center}

### HTML 페이지

- 동적으로 필요한 HTML 파일을 생성해서 전달
- 웹 브라우저는 HTML을 해석

![](https://i.imgur.com/BVB1zcU.png)
{: .align-center}

### HTTP API

- HTML이 아니라 **데이터**를 전달
- 주로 JSON 형식 사용
- 다양한 시스템에서 호출

![](https://i.imgur.com/4ScOBu0.png)
{: .align-center}

- 데이터만 주고 받음, UI 화면이 필요하면 클라이언트가 별도 처리
- 앱, 웹 클라이언트, 서버 to 서버

![](https://i.imgur.com/oyJ3QCj.png){: .align-center}

#### API 다양한 시스템 연동

- 주로 JSON 형태로 데이터 통신
- UI 클라이언트 접점
	- 앱 클라이언트(아이폰, 안드로이드, PC)
	- 웹 브라우저에서 자바스크립트를 통한 HTTP API 호출
	- React, Vue.js 같은 웹 클라이언트
- 서버 To 서버
	- 주문서버 -> 결제 서버
	- 기업 간 데이터 통신


## 서버사이드 렌더링, 클라이언트 사이드 렌더링

#### SSR - 서버 사이드 렌더링

- HTML 최종 결과를 서버에서 만들어서 웹 브라우저에 전달
- 주로 정적인 화면에 사용
- 관련 기술 : JSP, 타임리프 -> **백엔드 개발자**

![](https://i.imgur.com/Fw33Tvw.png){: .align-center}

#### CSR - 클라이언트 사이드 렌더링

- HTML 결과를 자바스크립트를 사용해 웹 브라우저에서 동적으로 생성해서 적용
- 주로 동적인 화면에 사용, 웹 환경을 마치 앱처럼 필요한 부분부분 변경할 수 있음
- 예 ) 구글 지도, Gmail, 구글 캘린더
- 관련 기술 : React, Vue.js -> **웹 프론트엔드 개발자**


![](https://i.imgur.com/0JA6LN0.png){: .align-center}


> 참고  <br>
> React, Vue.js를 CSR + SSR 동시에 지원하는 웹 프레임워크도 있음. <br>
> SSR을 사용하더라도, 자바스크립트를 사용하여 화면 일부를 동적으로 변경 가능.


### 백엔드 개발자 입장에서 UI 기술

- **백엔드 - 서버 사이드 랜더링 기술**
	- JSP, 타임리프
	- 화면이 정적이고, 복잡하지 않을 때 사용
	- 백엔드 개발자는 서버 사이드 랜더링 기술 학습 **필수**

- **웹 프론트엔드 - 클라이언트 사이드 렌더링 기술**
	- React, Vue.js
	- 복잡하고 동적인 UI 사용
	- 웹 프론트엔드 개발자의 전문 분야

- **선택과 집중** 
	- 백엔드 개발자의 웹 프론트엔드 기술 학습은 **옵션**
	- 백엔드 개발자는 서버, DB, 인프라 등등 수 많은 백엔드 기술을 공부해야 한다.
	- 웹 프론트엔드도 깊이있게 잘 하려면 숙련에 오랜 시간이 필요하다.



