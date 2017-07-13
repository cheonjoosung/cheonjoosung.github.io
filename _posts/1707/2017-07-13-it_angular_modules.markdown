---
layout: post
title:  "Angular Modules : 앵귤러 모듈 기초"
date:   2017-07-13
categories: angular
highlight: false
image: /images/it/angular/angular_logo.jpg
tag: angular
---

 `Angular Modules(앵귤러 모듈) 기초에 관한 포스팅입니다.` Angular 애플리케이션은 많은 모듈로 구성되어 있습니다. 쉽게 말하면 모듈은 특정 기능을 제공하는 역할이에요. Angular 는 모듈을 쉽게 관리할 수 있도록 시스템을 제공을 해요. Angular 에서 모듈을 잘 사용하면 프래그래밍의 유지보수성 및 효율성을 높여주는데 큰 도움이 된다고 생각해요. 그러니 반드시 숙지!!!!!

 <br><b>Angular Modules ?</b><br>
<br><b>1) `Angular 라이브러리 모듈`</b><br>
<p>=) 종류 : 지시자, 파이브, 장식자, 클래스, 인터페이스, 함수, Enum, 타입 별칭, 상수</p>
<p>=) 모듈을 패키지 단위로 정의 - 주요 패키지 아래 사진 참고</p>
![Dev Image](/images/it/angular/angular_module.png)
<p>=) 외부로 공개할 모듈은 `export 키워드 사용`</p>
<p>=) `new_name as module_name` 코딩 시 모듈 이름 변경하여 쉽게 사용 가능</p>
<p>=) `루트 모듈 : app.module.ts 로 이름으로 생성되고 최상위 모듈이다.` 애플리케이션 모듈을 구성한다.</p>

 <br><b>2) `루트 모듈 : root module`</b><br>
 <p>=) 애플리케이션의 최상위 모듈로 모듈 구성하는데 도움을 주는 역할을 함 . [NgModules 정보](https://angular.io/guide/ngmodule)</p>
 <p>=) `@NgModule(){}` 키워드를 사용하여 imports(호출한 모듈), providers(서비스 주입), declarations(사용 컴포넌트, 지시자, 파이프). bootstrap(구동할 컴포넌트)</p>
{% highlight ruby %}
@NgModule({
  declarations: [
    AppComponent,
    DashComponent,
    Menu1Component,
    Menu2Component,
    GuideDetailComponent,
    HeroSearchComponent,
    HeroesComponent,
    HeroDetailComponent
  ],
  imports: [
    BrowserModule,
    FormsModule,
    HttpModule,
    AppRoutingModule,
    BrowserAnimationsModule,
    MdButtonModule,
    MdCheckboxModule,
    MaterialModule.forRoot(),
    InMemoryWebApiModule.forRoot(InMemoryDataService)
  ],
  providers: [HeroService],
  bootstrap: [AppComponent]
})
{% endhighlight %}
<p>=) Angular 가 브라우저 위에서 작동하려면 BrowserModule 가 필수</p>
<p>=) FormModule : 템플릿에서 사용하는 지시자 포함 ex) NgModel</p>
<p>=) RoutingModule : 사용ㅈ가 정의 가능한 라우팅 모듈 ex) 페이지 이동</p>

모듈은 상당히 많으므로 해당 포스트에서는 간단한 개요에 대해서 설명하고 특정 모듈이 있을 경우 그 모듈에 관해서 포스팅을 따로 할. 다음 포스팅에 라우팅 모듈과 레이지 모듈에 대해서 써보도록 할께요.
