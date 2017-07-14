---
layout: post
title:  "Angular Routing Guard"
date:   2017-07-15
categories: angular
highlight: false
image: /images/it/angular/angular_logo.jpg
tag: angular
---

 `Angular 의 Routing Guard 에 관한 포스팅입니다.` 가드라는 말은 말그대로 어떤것을 지킨다의 의미겠죠? Angular 에서 가드(Guard)는 라우팅 시 접근을 제어하는 방법입니다. 네이버를 본다면 마이 페이지의 경우 로그인이 되어 있지않으면 팅겨야 하지 않을까요? 앵귤러를 통해 대규모 애플리케이션을 제작은 안해봤지만 제 생각으로는 권한이 필요한 부분에 Guard 의 개념이 쓰일 것이라 생각합니다. 가드에는 크게 `CanActivate, CanActivateChild, CanDeactivate, Resolve -> CanLoad 변경된 듯...?` 4가지 종류가 있습니다.

<br><b>1) `1 CanActivate Guard?`</b><br>
<p>=) 라우트 권한을 검사해 접근을 허용할지 말지 결정하는 가드 : 요청 URL 에 라우팅 정보가 있는지 체크 </p>

{% highlight ruby %}
class UserToken {}
class Permissions {
  canActivate(user: UserToken, id: string): boolean {
    return true; #UserToken, id 에 따라 true or false 를 리턴하는 로직 만들어야 함
  }
}

@Injectable()
class CanActivateTeam implements CanActivate {
  constructor(private permissions: Permissions, private currentUser: UserToken) {}

  canActivate(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<boolean>|Promise<boolean>|boolean {
    return this.permissions.canActivate(this.currentUser, route.params.id);
  }
}

@NgModule({
  imports: [
    RouterModule.forRoot([
      {
        path: 'team/:id',
        component: TeamCmp, #이동 페이지
        canActivate: [CanActivateTeam]
      }
    ])
  ],
  providers: [CanActivateTeam, UserToken, Permissions]
})
class AppModule {}
{% endhighlight %}
위의 코드 [Angular 공식](https://angular.io/api/router/CanActivate/) 참고
<p>=) `CanActivateTeam, UserToken, Pemissions` 을 providers 로 제공 </p>
<p>=) RouterModule.ts 로 만들어 임포트도 가능</p>
<p>=) <b>이해가 안되는 점 : UserToken, Permission 은 Injectable 없이 Providers 로 제공이 가능한 서비스 인지...?</b></p>

<br><b>1) `2. CanActivateChild Guard?`</b><br>
<p>=) 자식 라우트에 대한 접근 허용할지 말지 결정하는 가드</p>
<p>=) </p>

{% highlight ruby %}
class UserToken {}
class Permissions {
  canActivate(user: UserToken, id: string): boolean {
    return true;
  }
}

@Injectable()
class CanActivateTeam implements CanActivateChild {
  constructor(private permissions: Permissions, private currentUser: UserToken) {}

  canActivateChild(
    route: ActivatedRouteSnapshot,
    state: RouterStateSnapshot
  ): Observable<boolean>|Promise<boolean>|boolean {
    return this.permissions.canActivate(this.currentUser, route.params.id);
  }
}

@NgModule({
  imports: [
    RouterModule.forRoot([
      {
        path: 'root',
        canActivateChild: [CanActivateTeam],
        children: [
          {
             path: 'team/:id',
             component: Team
          }
        ]
      }
    ])
  ],
  providers: [CanActivateTeam, UserToken, Permissions]
})
class AppModule {}
{% endhighlight %}
위의 코드 [Angular 공식](https://angular.io/api/router/CanActivateChild) 참고
<p>=) 사용하는 부분에서는 CanActive Guard 와 거의 차이가 없음. 루트 모듈에 선언시 canActivateChild 를 사용하는 부분과 childeren: [ { } ] 부분</p>

<br><b>1) `3. CanDeactivate Guard?`</b><br>
<p>=) 저장되지 않은 상태를 버릴지 결정 : 웹 페이지에서 글 작성 시 페이지 주소가 바뀔 때 변경사항을 저장할지 말지를 사용자에게 알리기 위해서 사용</p>

{% highlight ruby %}
class UserToken {}
class Permissions {
  canDeactivate(user: UserToken, id: string): boolean {
    return true;
  }
}

@Injectable()
class CanDeactivateTeam implements CanDeactivate<TeamComponent> {
  constructor(private permissions: Permissions, private currentUser: UserToken) {}

  canDeactivate(
    component: TeamComponent,
    currentRoute: ActivatedRouteSnapshot,
    currentState: RouterStateSnapshot,
    nextState: RouterStateSnapshot
  ): Observable<boolean>|Promise<boolean>|boolean {
    return this.permissions.canDeactivate(this.currentUser, route.params.id);
  }
}

@NgModule({
  imports: [
    RouterModule.forRoot([
      {
        path: 'team/:id',
        component: TeamCmp,
        canDeactivate: [CanDeactivateTeam]
      }
    ])
  ],
  providers: [CanDeactivateTeam, UserToken, Permissions]
})
class AppModule {}
{% endhighlight %}
위의 코드 [Angular 공식](https://angular.io/api/router/CanDeactivate) 참고
[관련 추가 자료](http://projectl33t.xyz/archives/255) 라우팅에 관해 잘 정리해 놓으심
<p>=) canDeactivate() 에 페이지에 대한 정보를 저장하는 것 같다.</p>

<br><b>1) `4. CanLoad Guard?`</b><br>
<p>=) 사용자가 권한을 부여받지 않는 경우 응용 프로그램이 전체 모듈을 지연로드 하는 것을 방지하기 위해 사용</p>

{% highlight ruby %}
class UserToken {}
class Permissions {
  canLoadChildren(user: UserToken, id: string): boolean {
    return true;
  }
}

@Injectable()
class CanLoadTeamSection implements CanLoad {
  constructor(private permissions: Permissions, private currentUser: UserToken) {}

  canLoad(route: Route): Observable<boolean>|Promise<boolean>|boolean {
    return this.permissions.canLoadChildren(this.currentUser, route);
  }
}

@NgModule({
  imports: [
    RouterModule.forRoot([
      {
        path: 'team/:id',
        component: TeamCmp,
        loadChildren: 'team.js',
        canLoad: [CanLoadTeamSection]
      }
    ])
  ],
  providers: [CanLoadTeamSection, UserToken, Permissions]
})
class AppModule {}
{% endhighlight %}
위의 코드 [Angular 공식](https://angular.io/api/router/CanLoad) 참고

`전체적으로 가드에 대한 설명이 부족하다... 실전으로 하면서 겪어야 될 것 같네요. 내용에 관한 정보가 부족한 상황이라 저도 점차적으로 테스트 한 후에 포스팅을 점차 수정하도록 하겠습니다.`
