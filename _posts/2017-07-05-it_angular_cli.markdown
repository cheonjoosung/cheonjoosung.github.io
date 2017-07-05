---
layout: post
title:  "Angular CLI 설치"
date:   2017-07-05
categories: angular
highlight: false
image: /images/it/170705_angular/angular_cli.jpg
tag: angular
---

 Angular 2 프로젝트의 설정파일은 상당히 많다. package.json 을 이용하여 일일히 설정하는 것은 비효율적이라고 생각하는 편이다.(물론 프로젝트 설정에 대한 이해도는 올라감) 효율적인 Angular CLI 를 통해서 프로젝트 환경설정을 간편히 하고 코드 개발에 시간을 더 투자하는 것이 현명한 방법이라고 생각한다.

<br><br>
 ![Dev Image](/images/it/170705_angular/angular_cli.jpg)

 <br><b>Angular CLI 소개</b>
 - Angular CLI 는 개발에 집중할 수 있도록 도와주는 도구
 - Angular 프로젝트 생성, 중요 구성요소 추가(컴포넌트, 지시자, 파이프, 서비스), 프로젝트 빌드 & 실행, 브라우저 동기화, 테스트 환경 제공, 배포를 위한 패키징 기능 제공

<br><b>Angular CLI 설치</b>
  ![Dev Image](/images/it/170705_angular/angular_ng1.png)
<br>
 - =)`npm install -g @angular/cli` - [node](https://nodejs.org/en/download) 가 설치되어 있어야 작동, `angular-cli deprecated`
 - =)`ng` or `ng help` or `ng --help` 를 커맨드에 입력하면 설치가 제대로 되었는지 확인 가능하다. 또한, 사용가능한 명령어에 대한 설명도 확인이 가능하다.
 - =) 기존에 angular-cli 가 설치되어 있는 경우에 `npm uninstall --save-dev angular-cli && npm install --save-dev @angular/cli@latest`

<br><b>Angular CLI 이용하여 프로젝트 만들기</b>
  ![Dev Image](/images/it/170705_angular/angular_ng3.png)
<br>
- =) Angular-CLI 가 설치가 완료되었다면 `ng new project_name_you_want`
- =) `cd project_name_you_want`을 입력하여 해당 폴더로 이동
- =) `ng serve` 를 입력하면 컴파일이 진행된 후에 로컬에 HTTP 서버가 실행됩니다. 해당 서버는 클라이언트 요청에 대해 응답합니다.
- =) 브라우저 실행 후 주소창에 `localhost:4200` 을 입력하면 Angular Application 이 실행된 화면을 볼 수 있습니다.

<br><b>Angular CLI 명령어 모음</b>
<br>
- =) `ng g component component_name` or `ng g directive directive_name` or `ng g pipe pipe_name` or `ng g service service_name`. Component 의 경우 css, html, spec.ts, ts 파일 자동 생성 / Service, pipe, directive 의 경우 spec.ts, ts 파일 자동 생성
- =) `ng build` 는 Test 용으로 파일크기 최적화가 아님. src/environments/enviroment.ts 파일의 개발환경 설정 정보를 이용하여 빌드 함
- =) `ng build --prod --env=prod or ng build --prod` 는 프로젝트를 프로덕션용 빌드를 위해서 사용하는 명령어 => dist 디렉토리가 생성되고 결과가 저장됨
- =) `ng test` - 단위 테스트 실행,  `ng e2e` - 종단 테스트(end to end test) 실행
- =) `ng serve --port 4300 --live-reload port 49000` 서버 포트를 4300으로 변경, 라이브 리로드 서버 포트를 49000 으로 변경
