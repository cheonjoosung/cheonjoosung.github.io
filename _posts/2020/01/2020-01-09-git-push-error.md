---
title: Git/Github requested URL returned Error 403 해결방법
tags: [git]
style: fill
color: dark
description: git push requested url returend error 403 에러 해결방법
---

## 발생이유
`git the requested URL returend error : 403`이 발생하는 이유는 다양한 편입니다. 저의 경우 코드를 작성하고 깃허브에 세팅하고 첫 푸쉬를 실행할 때 403 에러가 발생했습니다.

## 해결방법
 아래의 코드를 통해 깃허브 레퍼지토리에 대한 접근 권한/인증을 받으면 됩니다.
```javascript
git remote set-url origin https://github-username@github.com/github-username/github-repository-name.git
 ```
 1. 터미널 창을 실행하고 명령어 입력하여 깃헙 레퍼지토리와 연결
 2. github-username -> github 에서 사용하는 username 
 3. github-repository-name -> github에 코드를 추가하고자 하는 repository name 명
 
이 포스트에서 사용하는 명령어는 원격 저장소와 작성한 코드를 연결하기 위한 것과 연관이 되어 있습니다. [깃 학습 사이트](https://git-scm.com/book/ko/v2/%EC%8B%9C%EC%9E%91%ED%95%98%EA%B8%B0-%EB%B2%84%EC%A0%84-%EA%B4%80%EB%A6%AC%EB%9E%80%3F)를 이용하세요