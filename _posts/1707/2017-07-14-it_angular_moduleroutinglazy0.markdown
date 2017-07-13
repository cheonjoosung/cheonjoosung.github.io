---
layout: post
title:  "Angular Modules : Routing & Lazy"
date:   2017-07-14
categories: angular
highlight: false
image: /images/it/angular/angular_logo.jpg
tag: angular
---

 `Angular Module 중 Routing & Lazy 모듈 사용에 관한 포스팅입니다.` Angular 애플리케이션은 많은 모듈로 구성되어 있습니다. 쉽게 말하면 모듈은 특정 기능을 제공하는 역할이에요. Angular 는 모듈을 쉽게 관리할 수 있도록 시스템을 제공을 해요. Angular 에서 모듈을 잘 사용하면 프래그래밍의 유지보수성 및 효율성을 높여주는데 큰 도움이 된다고 생각해요. 그러니 반드시 숙지!!!!!

<br><b>1) `Routing 모듈 사용하기`</b><br>
<p>=) 종류 : 지시자, 파이브, 장식자, 클래스, 인터페이스, 함수, Enum, 타입 별칭, 상수</p>
<p>=) 모듈을 패키지 단위로 정의 - 주요 패키지 아래 사진 참고</p>
![Dev Image](/images/it/angular/angular_module.png)
<p>=) 외부로 공개할 모듈은 `export 키워드 사용`</p>

 <br><b>2) `Lazy 모듈 사용하기`</b><br>
 <p>=) 애플리케이션의 최상위 모듈로 모듈 구성하는데 도움을 주는 역할을 함 . [NgModules 정보](https://angular.io/guide/ngmodule)</p>
 <p>=) `@NgModule(){}` 키워드를 사용하여 imports(호출한 모듈), providers(서비스 주입), declarations(사용 컴포넌트, 지시자, 파이프). bootstrap(구동할 컴포넌트)</p>
{% highlight ruby %}
@NgModule({ })
{% endhighlight %}
<p>=) Angular 가 브라우저 위에서 작동하려면 BrowserModule 가 필수</p>
