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
  ![Dev Image](/images/it/170704_angular/arch_modules.png)
<br>
 - =) 특징을 가진 클래스를 모듈이라 하고 최소 하나 이상의 Angular Module(the root module or AppModule) 클래스를 가진다.
 - =) @NgModule decortor 를 사용하여 클래스를 정의함. 응용 플로그램을 구성하는데 도움을 주는 응집력 있는 블록 -> { }


<br><b>Components</b>
   ![Dev Image](/images/it/170704_angular/arch_components.png)
<br>
- =) 스크린(View)의 일부분을 컨트롤. 즉 View를 지원하기 위한 application logic 을 정의하는 곳
- =) 클래스 안에 선언 된 properties & method 의 API 를 통해 뷰와 정보를 주고 받음
- =) Angular 는 User & Application 의 이동함에 따라 컴포넌트를 생성, 갱신, 제거함

<br><b>Templates</b>
   ![Dev Image](/images/it/170704_angular/arch_templates.png)
<br>
- =) HTML 의 형태이며 Angular 가 Components 를 렌더링 하는 방법을 알려줌
- =) HTML 을 사용하나 일부의 차이점 존재 -> Angular's template syntax 가 존재(ex : ngFor, (click), ngif 등)
- =) <hero-detail></hero-detail> 을 통해 뷰를 재사용. metadata 의 selector 를 통해 선언하고 해당 네이밍을 통해 접근 함. 검색 Components 를 구현하고 selector : search 라 정의하면 검색 뷰를 쓰고 싶은 어느 곳에서 <search> </search> 태그를 사용하면 해당 뷰와 컨트롤 로직을 사용할 수 있다. 즉, 재 사용성이 높아짐.

<br><b>Metadata</b>
   ![Dev Image](/images/it/170704_angular/arch_metadata.png)
<br>
- =) Angular 에게 클래스를 처리하는 방법을 알려줌
- =) @Component decorator 를 사용하여 metadata 추가
- =) selector 는 Templates 에서 언급한 내용으로 <hero-detail></hero-detail> 을 통해 뷰를 재사용.
- =) templateUrl : Component 의 HTML template 주소
- =) providers : Component 가 사용하는 서비스를 위한 dependency injection providers

<br><b>Data binding</b>
   ![Dev Image](/images/it/170704_angular/arch_databinding.png)
<br>
- =) Component 와 Templates 사이에 데이터를 Push & Controll 하는 역할
- =) Two-way Data binding 지원 - [(ng-model)] 사용
- =) 4가지 형태의 데이터 바인딩 구문 존재.

<br><b>Directives</b>
   ![Dev Image](/images/it/170704_angular/arch_directive.png)
<br>
- =) @Directive decorator 를 가진 클래스로 directives 에 따라 DOM(Document Object Model) 변환
- =) Structural & Attriute 2가지 형태가 존재 함

<br><b>Services</b>
   ![Dev Image](/images/it/170704_angular/arch_services.png)
<br>
- =) Application 에 필요한 values, function, feature 포함. (구글 권장 가이드)
- =) nrrow & well-defined 된 목적을 가진 클래스 - 각 서비스는 하나의 기능만을 담당하도록 구성 (구글 권장 가이드)
- =) 기본 클래스 처럼 코딩을 하고 DI(Dependency injection) 를 사용 함
- =) Data fetch(http 통신 포함), 유저 input 검증, 로그 등의 역할을 수행(Component 가 작업 위임)

<br><b>Dependency injection</b>
   ![Dev Image](/images/it/170704_angular/arch_di.png)
<br>
- =) 클래스의 인스턴스에 필요한 DI를 제공하는 방법(대부분의 DI는 서비스)
- =) Component 에 필요한 서비스를 제공하기 위해서 DI 사용
- =) AppModule 에 providers 를 통해 서비스 등록 가능. 각 컴포넌트마다 서비스를 개별적으로 사용해도 되지만 구글은 AppModule 에 전역적으로 선언하고 사용하는 것을 권장


<br><br><a href="https://angular.io/guide/architecture" target="_blank">Angular Architecture 자료 출처</a>
