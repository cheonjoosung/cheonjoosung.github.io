---
title: Youtube Data API "Getting Error 403 forbidden" or "quotaExceeded"
tags: [It, Youtube]
style: fill
color: dark
description: Youtube Data API 를 사용하면서 발생할 수 있는 오류 문제
---

## Youtube Data API "Getting Error 403 forbidden"
Youtube Data API를 사용하기 위해서는 API 키를 생성하고 사용자 인증정보가 필요하다. [Google API](https://console.developers.google.com/apis) 이곳에서 애플리케이셔 제한사항, API 제한사항 등의 기본 설정을 한다.

필자의 경우 애플리케이션 제한사항 - Android 앱을 하고 SHA1 을 추가해서 **Getting Error 403 forbidden**문제가 발생했다.

![preview](https://i.imgur.com/RuGPUOn.png)
위의 사진처럼 **애플리케이션 제한사항 - 없음** / API 제한사항 키 제한 API 1개 로 변경한 후 Youtube Data API를 조회했을 때 정상적으로 원하는 값을 얻을 수 있었다.

## quotaExceeded
**Youtube Data API 할당량은 하루 10,000**으로 정해져있다. 할당량이 초과되면 Youtube Data API를 요청해도 quotaExceeded 를 반환한다.

[Youtube Data API 할당량 체크](https://console.cloud.google.com/apis/api)의 사이트에 들어가면 현재까지 사용한 할당량을 확인할 수 있다.

![preview](https://i.imgur.com/0bVpNra.png)

![preview](https://i.imgur.com/jsHG1oo.png)

위의 사이트에 들어가면 Youtube Data API 중 사용한 메소드 별로 할당량을 확인할 수 있다. 그리고 모든 요청을 더한 값이 하루 할당량을 넘었는지 쉽게 볼 수 있다.

[각 메소드 별 할당량 사용치](https://developers.google.com/youtube/v3/getting-started)에 들어가면 메소드별 사용량이 다르고 요청 매개변수에 따라서도 할당량이 다르게 측정됩니다.

**할당량의 초기화는 태평양 표준시 자정**이다. [세계 시간 확인](https://kr.piliapp.com/time-now/)에 들어가면 태평양 표준시 24:00 or 00:00 의 시간이 되었을 때 자신이 속한 나라를 확인하면 된다. 한국의 경우 오후 4시인 16:00에 Youtube Data API 할당량이 초기화 된다.

**참고자료**
- [구글 Youtube Data API Error](https://developers.google.com/youtube/v3/docs/errors?hl=koo) 공식문서 참조

## 결론
- **Youtube Data API v3 재밌네요.**