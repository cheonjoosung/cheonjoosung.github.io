---
layout: post
title:  "Angular Architecture(앵귤러 아키텍처)"
date:   2017-07-04
categories: angular
highlight: false
image: /images/it/170703_angular/angular.png
tag: angular
---

 Angular 는 JS와 HTML 를 가지고 클라이언트 앱을 만들기 위한 프레임워크이다. 해당 프레임워크는 몇 개의 라이브러리를 포함하고 있고 그중에는 핵심코어도 포함되어 있다. Angular 앱을 만들기 위해서는 HTML Templates, Components class, Services, modules 등 다양한 구조를 사용해야 한다. 아래에서는 Angular 에서 핵심적인 8개의 Architecture 에 대한 내용이다.

<br><br>
 ![Dev Image](/images/it/170704_angular/angular_arch1.png)

 <br><b>Angular Architecture</b>
 - 위는 Angular Architecture 의 구조를 한눈에 보여주는 사진이다. Modules, Templates, Components, Metadata, Directive, Data binding, Services, DI(Dependency injection) 이 있다. <br>Templates 은 화면에 나오는 View로 생각하면 되고 Components 는 View 에 관한 처리를 담당하고 서비스는 각 컴포넌트 마다 필요한 데이터 처리를 관리하는 부분이라고 생각하면 된다. Templates {{title}} - 변수, (click) - 액션 에 대한 내용은 Components 에서 관리하고 있다. hero 에 대한 내용은 Components 가 아니라 Services 에서 관리를 하고 있다.<br><b>구글에서는 Components 에서 데이터 처리를 하는 것을 권장하지 않고 Services 에서 처리하는 것을 권장한다. 즉 Components 는 Services를 호출하고 그 값을 binding 하여 Templates에 전달을 하는 역할을 하면 된다.</b> 

 <br><b>Modules</b>
 - 모듈

<br><b>Components</b>
 - 모듈

<br><b>Templates</b>
 - 모듈

<br><b>Metadata</b>
 - 모듈

<br><b>Data binding</b>
 - 모듈

<br><b>Directives</b>
 - 모듈

<br><b>Services</b>
 - 모듈

<br><b>Dependency injection</b>
 - 모듈
