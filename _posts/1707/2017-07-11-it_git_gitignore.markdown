---
layout: post
title:  "gitignore 파일 설정 및 반영되지 않을 때"
date:   2017-07-11
categories: git
highlight: false
image: /images/it/170710_github/github.png
tag: git
---

 `gitignore 파일 설정 및 반영되지 않을 때' 에 관한 포스팅입니다`. .gitignore 깃 영역안에서 자신의 깃주소로 푸쉬를 할 때 파일 or 디렉토리의 커밋을 제한하는 역활은 합니다. 예를 들면 자바 프로젝트는 컴파일을 하게 되면 bin 폴더에 .class 파일들이 생성되는데 깃헙에는 .class 파일없이 .java 파일만 올리는 것이 좋지 않을까요? class 파일들이 깃이 커밋되지 않도록 해주는 역할을 하는 것이 바로 `.gitignore` 입니다.

 <br><b>1..gitignore 생성 및 설정하기 </b>
 - =) 오픈소스나 샘플 프로젝트를 세팅하면 대부분이 .gitignore 가 포함되어 있지만 본인이 직접 프로젝트를 생성하면 없다.
 - =) 즉, 직접 만들어야 한다. 터미널에서 `vim .gitignore` or `vi .gitignore` or 직접 텍스트파일로 만들어도 됨. 본인의 선택!!!
 - =) 만들었으면 이제 반은 성공. Java 프로젝트를 예제로 진행할께요.

<b>Java 컴파일 했을 때</b>
   ![Dev Image](/images/it/170711_gitignore/gitignore1.png)
<br>
 - =) .gitignore 를 열어서 입력을 해봅시다.
<br>
![Dev Image](/images/it/170711_gitignore/gitignore3.png)
<br>
  - =) .gitignore 에 `*.class` or `bin/` 을 사용해서 클래스 파일들을 무시할 수 있다.
  - =) `bin/` 을 추가한 후에 깃헙에 푸쉬를 하면 아래의 사진과 같이 결과를 확인 가능하다.

<b>gitignore 세팅 후 깃헙에 Push 했을 때</b>
   ![Dev Image](/images/it/170711_gitignore/gitignore2.png)
<br>

 <br><b>2. gitignore 반영되지 않을 때</b>
 - =) .gitignore 를 세팅해도 반영이 되지 않을때가 발생한다. 깃의 캐쉬 역할 때문에 발생한다.
 - =) 해당 문제는 터미널을 켜서 `git rm -r --cached` 를 입력한 후에 `git add . && git commit -m "remove cached" && git push` 하면 반영이 된다.

<br><b>gitignore 추가자료</b><br>
 - =) [gitignore 자동 생성](https://www.gitignore.io/) <br>
 - =) [gitignore 깃헙 주소](https://github.com/github/gitignore)<br>
 - =) [gitignore doc 문서](https://git-scm.com/docs/gitignore)
