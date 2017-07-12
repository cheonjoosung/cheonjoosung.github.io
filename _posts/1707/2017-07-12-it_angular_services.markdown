---
layout: post
title:  "Angular Service : 앵귤러 서비스 기초"
date:   2017-07-12
categories: angular
highlight: false
image: /images/it/angular/angular_logo.jpg
tag: angular
---

 `Angular Service(앵귤러 서비스) 기초에 관한 포스팅입니다.` Angular 에서 Service 는 외부에서 정의된 의존성 주입(Dependeny Injection) 대상이 되는 클래스에요. 서비스를 사용하기 위해서는 몇가지 세팅이 필요해요. Providers 선언, injectable 등 서비스에 전반적인 개념과 사용법에 대한 글을 써볼께요.

 <br><b>Angular Service 역할</b><br>
<br><b>1) `애플리케이션 관심사와 개별 컴포넌트 관심사 분리`</b><br>
<p>=) 컴포넌트에 공통적으로 쓰일 수 있는 기능을 서비스로 개발 - 컴포넌트 : 독립적인 특성을 지닌 기능</p>
<p>=) 중복되는 코드를 서비스로 옮기는 작업을 통해 컴포넌트의 응집성 높여야 함</p>

 <br><b>2) `컴포넌트 중간에서 데이터 중개자의 역할 수행`</b><br>
 <p>=) 컴포넌트 to 컴포넌트 의 데이터 교환은 서비스를 통해 이루어 짐</p>
 <p>=) @Input 장식자나 EventEmitter 객체 이용 : 일관속 있는 자료구조가 없어서 개별적인 속성 값만 주고 받음</p>

 <br><b>2) `관심사 분리`</b><br>
 <p>=) 관심사 분리는 프로그래밍 기법 중에 하나</p>
 <p>=) 전역 관심사 : 데이터 서비스, 로깅 서비스, 비즈니스 로직 서비스 등은 대부분의 컴포넌트에서 공통적으로 사용이 가능</p>
 <p>=) 각 서비스는 최소한의 기능을 갖는 단위로 작성하는 것이 구글 가이드</p>
 <p>=) 관심사 분리를 통해 중복 코드가 줄어들고 재사용성 향상 됨</p>

<br><b>Angular Service 사용</b><br>
<br><b>1) `Hello Service`</b><br>
<p>=) Angular CLI 를 통해 `ng g service service_name` 를 입력하면 service_name.ts, service_name.spec.ts 파일이 생성 됨</p>
<p>=) app.module.ts 의 providers : [ service_name ] 등록 && import { service_name } from ./경로</p>

{% highlight ruby %}
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';
import { AppComponent } from './app.component';
import { HelloService } from './hello.service';

@NgModule({
  declarations: [ AppComponent ],
  imports: [ rowserModule ],
  providers: [ HelloService ],
  bootstrap: [ AppComponent ]
})
export class AppModule { }

{% endhighlight %}

<p>=) hello.service.ts 파일 오픈 인사 리턴하는 함수 작성</p>

{% highlight ruby %}
import { Injectable } from '@angular/core';

@Injectable()
export class HelloService {
  constructor() {  }

  say() {
    return "Hello Sydeny!!!"
  }
}
{% endhighlight %}

<p>=) app.component.ts 파일을 오픈 Service 를 사용해보자  </p>

{% highlight ruby %}
import { Component } from '@angular/core';
import { HelloService } from './hello.service'

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {
  title = '';
  constructor(private helloService : HelloService) {
    this.title = helloService.say();
  }
}
{% endhighlight %}

<p>=) template 에서 {{ title }} 을 호출하면 Hello Sydeny!! 가 호출되는 것을 확인할 수 있다. </p>
<p>=) 서비스를 생성자를 통해 초기화 할 수 있고 new 연산자를 사용하여 초기화도 가능하다. 하지만 new 연산자를 사용하면 객체 공유 문제 및 메모리 관련 문제가 생길 수 있다.</p>

<br><b>2) `데이터 객체 사용`</b><br>
<p>=) 사용할 데이터 형 설정 && 데이터 객체 설정</p>
{% highlight ruby %}
export class Hero {
  position : string;
  name: string;
}

import { Hero } from './hero';

export var HERO : Hero[] = [
  {position: 'top' , name:'티모'}, {position: 'mid' , name:'리 신'}, {position: 'AD' , name:'이즈리얼'}
];
{% endhighlight %}

<p>=) 서비스에서 데이터 객체 전달하는 함수 추가 : getHero() </p>
{% highlight ruby %}
import { Injectable } from '@angular/core';
import { HERO } from './hero-data';

@Injectable()
export class HelloService {

  constructor() {
  }

  say() {
    return "Hello Sydeny!!!"
  }

  getUser() {
    return HERO;
  }
}
{% endhighlight %}

<p>=) 컴포넌트에서 서비스 함수 호출 && this.heroList 에 저장 </p>
{% highlight ruby %}
import { Component } from '@angular/core';
import { HelloService } from './hello.service'
import { Hero } from './hero'

@Component({
  selector: 'app-root',
  templateUrl: './app.component.html',
  styleUrls: ['./app.component.css']
})
export class AppComponent {
  title = '';
  heroList:Hero[];

  constructor(private helloService : HelloService) {
    this.title = helloService.say();
    this.heroList = helloService.getUser();
  }
}
{% endhighlight %}

<br><b>3) `Promise 사용`</b><br>
<p>=) Promise 는 비동기 코드를 사용할 때 주로 사용</p>
<p>=) 위에서 사용된 User 데이터 형과 USER 객체는 그대로 사용하고 service 만 일부 수정</p>
{% highlight ruby %}
import { Injectable } from '@angular/core';
import { HERO } from './hero-data';
import { Hero } from './hero';

@Injectable()
export class HelloService {

  constructor() {
  }

  say() {
    return "Hello Sydeny!!!"
  }

  getHero(): Promise<Hero[]> {
    return Promise.resolve(HERO);
  }

  getHeroDelay(): Promise<Hero[]> {
    return new Promise<Hero[]>(resolve => setTimeout(resolve, 1000)).then(() => this.getHero());
  }
}

{% endhighlight %}

<p>=) Component 에서 호출 방법 : 성공시에 값을 할당한다는 의미</p>
{% highlight ruby %}
constructor(private helloService : HelloService) {
  this.title = helloService.say();
  helloService.getHero().then(hero => this.heroList = hero);
  helloService.getHeroDelay().then(hero => this.heroList2 = hero);
}
{% endhighlight %}
