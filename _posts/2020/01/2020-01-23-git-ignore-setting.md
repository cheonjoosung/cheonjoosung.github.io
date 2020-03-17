---
title: gitignore 설정하기/사용하기 + 반영되지 않을 때 해결방법
tags: [git, gitignore]
style: fill
color: dark
description: gitignore 파일 설정 및 반용되지 않을 때 해결방법
---

## gitignore ??
 `gitignore`란 Git의 버전을 관리하는 파일의 대상 중 제외시킬 목록을 저장할 때 사용하는 기능이다. 
 
예를 들면 테스트에 사용되는 파일 및 로그 등은 계속 쌓이기에 용량이 늘어나기에 깃에 굳이 저장할 필요가 없다. 또한, 자바 프로젝트는 컴파일을 하게 되면 bin 폴더에 .class 파일들이 생성되는데 깃에 이러한 파일들은 올릴 필요가 없습니다. 

 즉, **gitignore은 말 그래도 git 이 ignore 하는 목록**을 만들기 위한 기능을 수행한다.


## gitignore 설정/사용하기
 오픈소스의 경우 .gitignore가 포함되어 있지만 직접 프로젝트를 만들때 자동으로 생성되지 않는다.
 
 그렇기에 파일을 직접 만들어야 한다.
 1. 터미널에서 `vim/vi .gitignore` or `직접 텍스트파일`로 생성
 2. [gitignore 파익 목록 자동 생성 사이트](https://www.gitignore.io/) 이용
 
자바 샘플 프로젝트의 .gitignore 목록
<script src="https://gist.github.com/cheonjoosung/50e5866261899ace524d7c5a2d5ad443.js"></script>
 
알고리즘 문제 풀이를 위해 버전 관리를 사용하고 있고 클래스 파일이 많이 생성되기에 /bin폴더를 푸시하도록 추가했습니다. 폴더이외에 *.bmp, *.txt 등의 모든 파일을 무시하도록 추가할 수 있습니다.


## gitignore 반영되지 않을 때 해결 방법
깃의 캐쉬 역할로 인해 .gitignore가 제대로 반영되지 않을 때가 있습니다. 
1. `git rm -r --cached` 를 입력하여 캐쉬 삭제
2. `git add . && git commit -m "remove cached" && git push` 그리고 푸쉬를 한다면 반영이 돕니다.


## gitignore 관련 자료
 1. [gitignore 자동 생성](https://www.gitignore.io/)
 2. [gitignore 깃헙 주소](https://github.com/github/gitignore)
 3. [gitignore doc 문서](https://git-scm.com/docs/gitignore)


