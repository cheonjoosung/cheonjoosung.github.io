---
layout: post
title:  "Angular Component : 앵귤러 컴포넌트 구조"
date:   2017-07-11
categories: angular
highlight: false
image: /images/it/angular/angular_logo.jpg
tag: angular
---

 `Angular Component (앵귤러 컴포넌트) 구조에 관한 포스팅입니다.` Angular 에서 Component 는 상당히 중요한 개념입니다. 왜냐하면 Angular 는 Component 기반의 프로그래밍을 하기 때문이죠. 또한, Component 는 외부 의존성 없이 동작이 가능한 하나의 독립적인 프로그램이기도 합니다. 그럼 컴포넌트에 대해서 상세하게 살펴보도록 합시다.

 <br><b>Angular Component 구조</b>
 ![Dev Image](/images/it/angular/ng_component.png)
 크게 3가지의 영역(`Import 영역` , `Component 장식자 영역`, `Component 클래스 영역`)으로 나눌 수 있습니다.<br>
 1) `Import 영역`
<p>=)'@angular/core' 처럼 @키워드 사용하면 내부 모듈</p>
<p>=)'./경로.service' 로 사용하면 외부 모듈</p>

 <br>2) `Component 장식자 영역`
 <p>=)@Component({ }) 의 영역으로 Selector 속성과 Template & Style 속성을 가짐</p>
 <p>=)Selector 속성 : 컴포넌트의 이름으로 어느 컴포넌트에서 <Component_name></Component_name> 으로 접근 가능</p>
 <p>=)Template 속성 : template 을 사용하면 inline 형태로 코드 작성, templateUrl 을 사용하면 .html 파일에다가 코드 작성</p>
 <p>=)Style 속성 : styles 을 사용하면 inline 형태로 스타일 작성, styleUrls 을 사용하면 .css 파일에다가 스타일 작성</p>

 <br>3) `Component 클래스 영역`
 <p>=)템플릿 데이터 출력과 관련된 로직 처리</p>
 <p>=)서비스를 이용한 요청 결과를 받아 템플릿에 데이터 반영 로직</p>
 <p>=)템플릿으로부터 이벤트 받아서 해당 이벤트에 대한 로직을 처리 : 로직에 관한 데이터 처리는 서비스를 이용 (권장)</p>
 <p>=)바인딩 변수를 활용해 템플릿의 화면 제어</p>
