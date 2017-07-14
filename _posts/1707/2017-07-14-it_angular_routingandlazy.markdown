---
layout: post
title:  "Angular Modules : Routing & Lazy"
date:   2017-07-14
categories: angular
highlight: false
image: /images/it/angular/angular_logo.jpg
tag: angular
---

 `Angular Module 중 Routing & Lazy 모듈 사용에 관한 포스팅입니다.` Lazy 개념은 Angular Application 의 효율을 높여주는 중요한 특성? 특징? 중 하나입니다. 모든 걸 불러오는 것이 아니라 필요한 시점에 불러오는 개념!!! 인것 같긴한데... 저도 확신이 없어서 같이 배워봐요


<br><b>1) `Routing & Lazy 모듈 사용하기`</b><br>
<p>=) Routing 은 특정 페이지(URL)로 전환? 이동? ex) 상단 메뉴를 클릭시 해당 메뉴에 관한 페이지를 부를 때 사용</p>
<p>=) 먼저 2개의 컴포넌트(페이지)를 부르기 위해서 2개의 컴포넌트를 생성해봅시다. 'ng g component component_name'를 통해 2개를 생성</p>
<p>=) 2개의 페이지를 부를 수 있도록 routing.ts 파일을 생성해볼께요.</p>
{% highlight ruby %}
import { ModuleWithProviders } from '@angular/core';
import { Routes, RouterModule } from '@angular/router';

import { EagerComponent } from './eager.component';

const routes: Routes = [
  { path: '', redirectTo: 'eager', pathMatch: 'full' },
  { path: 'eager', component: EagerComponent },
  { path: 'lazy', loadChildren: './lazy/lazy.module#LazyModule' }
];

export const routing: ModuleWithProviders = RouterModule.forRoot(routes);
{% endhighlight %}

<p>=) 라우터 패키지에서 라우터 모듈을 추가해줬고 호출할 컴포넌트 EagerComponent 가 임포트 시켰구요</p>
<p>=) path: '경로', redirectTo: '디폴트 페이지', pathMatch: '매칭 정도'. 그리고 첫번째는 eager, lazy 를 클릭하면 그 때 LazyModule 을 불러서 lazy component 를 호출시키겠다는 의미에요. </p>
<p>=) 마지막 줄을 위의 선언한 변수를 바탕으로 routing Service 를 제공하겠다는 의미에요.</p>
<p>=) {path : 'list/:id, ....'} 으로 사용하면 url : list/id 를 사용하며 id값을 전달 가능</p>
<p>=) {path : '**', NotFoundComponent} 으로 사용해서 매칭되지 않을 때의 페이지로 이동시켜도 된다</p>

{% highlight ruby %}
import { BrowserModule } from '@angular/platform-browser';
import { NgModule } from '@angular/core';

import { AppComponent } from './app.component';

import { HelloService } from './hello.service';
import { EagerComponent } from './eager/eager.component';

import { routing } from './eager/routing';

@NgModule({
  declarations: [
    AppComponent,
    EagerComponent
  ],
  imports: [
    BrowserModule,
    routing
  ],
  providers: [HelloService],
  bootstrap: [AppComponent]
})
export class AppModule { }
{% endhighlight %}
<p>=) 루트 모듈로 와서 선언된 라우팅 모듈을 imports 에 추가해야 되요.</p>

{% highlight ruby %}
<nav>
<a routerLink="eager">Eager</a>
<a routerLink="lazy">Lazy</a>
</nav>
<router-outlet></router-outlet>
{% endhighlight %}
<p>=) 그리고 Angular Application 의 시작점인 app.component.html 에 위의 코드를 추가합니다.</p>
<p>=) 클릭을 하면 routerLink 의 이벤트는 eager or lazy 가 되고 발생된 이벤트를 routing service 가 캐치를 합니다. 그 후 해당 컴포넌트를 <routerLink-outlet></router-outlet> 태그 안에 결과를 보여줍니다. eager 를 클릭할 때는 나오지만 lazy 는 구현을 추가적으로 해야합니다.</p>

{% highlight ruby %}
import { NgModule } from '@angular/core';

import { LazyComponent }   from './lazy.component';
import { routing } from './lazy.routing';

@NgModule({
  imports: [routing],
  declarations: [LazyComponent]
})
export class LazyModule {
}
{% endhighlight %}
<p>=) 먼저 위의 routing.ts 에서 path: 'lazy', loadChildren: './lazy/lazy.module#LazyModule' lazy 를 클릭하면 LazyModule 를 부르게 됩니다. 이 때 lazyrouting 를 세팅하면서 해당 컴포넌트를 부를 수 있게됩니다.</p>

{% highlight ruby %}
import { ModuleWithProviders } from '@angular/core';
import { Routes, RouterModule } from '@angular/router';
import { LazyComponent }   from './lazy.component';

const routes: Routes = [
  { path: '', component: LazyComponent }
];

export const routing: ModuleWithProviders = RouterModule.forChild(routes);
{% endhighlight %}
<p>=) 코드는 routing.ts 와 비슷하나 RouterModule.forChild(routes) 를 사용하는것이 조금 다릅니다. </p>
<p>=) 해당 코드까지 작성하면 lazy 를 클릭하면 해당 컴포넌트를 부를 수 있게됩니다.</p>

<p>Lazy 개념을 사용하는 이유는 Angular 가 모든 모듈을 구동 시에 부르면 느릴 수 있습니다. 첫 화면에 필요한 부분만 부르고 특정 페이지에 들어갔을 때 해당 모듈을 부르게 된다면 속도도 빨리고 앱의 안정성도 높아질 것입니다. 애플리케이션의 규모가 커질수록 이 부분에 대한 고려가 많이 필요해질 것이라 생각합니다.</p>
