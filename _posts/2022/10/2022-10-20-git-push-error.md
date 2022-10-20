---
title: Android Studio git Invocation failed Unexpected end of file from server java.lang.RuntimeException: Invocation failed Unexpected end of file from 해결방법
tags: [git]
style: fill
color: dark
description: git Invocation failed Unexpected end of file from server java.lang.RuntimeException: Invocation failed Unexpected end of file from 해결방법
---

## Git push 문제
- git Invocation failed Unexpected end of file from server java.lang.RuntimeException: Invocation failed Unexpected end of file from
- Push failed remote: Invalid username or password. Authentication failed for

2가지에 대한 해결 방법을 알려드릴께요.

## git Invocation failed Unexpected end of file from server...

![preview](https://user-images.githubusercontent.com/13310269/196841776-44f43f9c-ee65-4041-8fb6-609d10d7cd4e.png)
Android Studio 에서 preferences > Version Control > Git - Use credential Hepler 설정을 체크해주세요

이렇게 하고 push 를 해도 정확한 에러가 표시가 되지 않을 경우
```gradle
git config credential.helper store
```
터미널을 열고 위의 명령어를 입력해주신다음 git push 를 해주세요.


## Push failed remote: Invalid username or password Authentication failed for ...

git push 를 하고나서 위와 같은 에러 문구가 떨어진 이유는 토큰 만료 때문입니다.

[토큰 갱신 주소](https://github.com/settings/tokens) 여기로 들어가서 토큰을 만들어주세요.

```gradle
git remote remove origin
git remote add origin https://닉네임:토큰@github.com/repository 경로
git push --set-upstream origin master
```

- 첫번째 라인을 실행시켜서 기존 origin 설정을 지워주세요.
- 그 다음에 변경된 토큰값으로 origin 을 재설정해주세요. 이때 https:// 를 빼고 해야합니다.
- 세번째 라인으로 실행시키면 다음부터 git push 를 통해 쉽게 코드를 푸쉬할 수 있어요.

