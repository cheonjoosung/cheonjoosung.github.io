---
layout: post
title:  "Angular Framework 개요"
date:   2017-07-03
categories: angular
highlight: false
image: /images/it/170703_angular/angular.png
tag: angular
---

 AngularJS 로 시작하였고 MEAN Stack 의 하나로써 기술을 잡음으로써 많은 사람들에게 알려짐. 이때는 잦은 업데이트로 인해서 불편함이 많이 있었다고 한다. 필자도 1.x 버전으로 오픈소스 채팅(Sendbird)을 개발헀을 때 업데이트 변경요소도 짜증이 났지만 그냥 러닝커브가 상당히 길었다. 두달정도 하다가 2가 나오고 확 바뀐다는 말에 접었다.

 ![Dev Image](/images/it/170703_angular/angular.png)

 <br><b>Angular 2 특징<b>
  - <b>1)</b> AngularJS 1 에서는 단순히 자바스크립트 프레임워크로써(JS 붙음) 되었지만 2부터는 JS 라는 단어가 빠지고 “One Framework for mobile & desktop” 타이틀을 가지고 갔다.

  - <b>2)</b> Angular 2는 스프린트를 도입하고 업데이트 주기를 정한것 같아서 개발 도중에 확 변할만 한 이슈는 사라진듯 하다.

  - <b>3)</b> AngularJS 1 에서 사용한 개념임 Controllers -> 클래스 중심, $scope -> 컴포넌트에 의해 영역이 구분 됨, angular.module -> 향상된 모듈 시스템 제공, jQlite -> DOM 제어 모듈 제공

  - <b>4)</b> C++ or Java 와 같은 OO 구조 느낌의 코딩형식을 제공(기존 객체 지향 개발자에게 어느정도 익숙해짐 & 러닝커브가 생각보다 낮아짐 - 1에서 사용되던 많은 지시자들 제거됨)

  - <b>5)</b> 주로 사용언어 타입스크립트(MS 만듬), Dart, Javascript 다 사용 가능 함.

  - <b>6)</b> Lazy Loading 을 사용함으로써 1에 비해 많은 성능 향상. 현재 페이지에 필요한 모듈만 로딩하는 방식.(1에 비해 2.5배 로딩시간 빠름, 바인딩을 통한 렌더링 성능 4.2배 빨라짐)

  - <b>7)</b> AoT 컴파일(Ahead of Compile)을 통해 템플릿을 실행 가능한 자바스크립트로 변환해 ㅋ코드에 적재하고 구동될 때 컴파일 과정없이 바로 실행 됨.<br>1 : 컴파일 - 렌더링 - 컴파일 -렌더링 ….. <br> 2 : 컴파일 - 화면 표시 - 화면 표시 - 화면 표시<br>코드용량 50%이상 최적화 & Angular 라이브러리 용량의 40~50% 달하는 JiT 컴파일러(just in time compiler) 적재 없어짐.

  - <b>8)</b> AngularJS 1 에서는 버전은 1.1 1.2 1.3 으로 관리 했지만 Angular 2 버전부터는 Angular 3(공개 안함??), 4(현재), 5 로 진행 두 번째 Major 버전이라고 보면 됨
￼

<br><br><b>Angular 2 관심도<b>
 ![Dev Image](/images/it/170703_angular/angular_interest1.png)

- Github 및 StackOverFlow 에서 각 프레임워크에 대한 활동을 바탕으로 만들어진 <a href="https://hotframeworks.com" target="_blank">프레임워크 랭킹 자료</a>
입니다.

 ![Dev Image](/images/it/170703_angular/angular_interest2.png)

- 구글 트렌드에서 연관 프레임워크를 검색한 결과입니다. <a href="https://trends.google.com/trends/explore?q=angularjs,backbone.js,asp.net,emberjs,spring%20framework/?target=_blank" target="_blank">구글 트렌드 검색 페이지로 이동</a>

<br><b>Angular 2 IDEs<b>
 ![Dev Image](/images/it/170703_angular/angular_tool.png)

- Angular IDE 에는 위와 같은데 유료 버전 중에는 웹 스톰이 많은 자바스크립트 개발자들이 사용한다고 합니다. 가격도 제일 비쌉니다. 제가 Atom 과 Visual Studio Code 를 사용해봤는데 개발하는데 큰 지장은 없어보입니다.

<br><b>Angular 2 Resources<b>
 ![Dev Image](/images/it/170703_angular/angular_resource.png)

- Angular UI Component 및 Cross-Platform 연관된 리소스입니다.
