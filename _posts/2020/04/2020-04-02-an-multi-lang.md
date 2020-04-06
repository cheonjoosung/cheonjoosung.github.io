---
title: Android 다국어 처리 설정 방법 - Translations Editor
tags: [Android]
style: fill
color: dark
description: 안드로이드의 translations Editor 를 활용하여 다국어 처리 설정방법을 알아보자
---

## 다국어 처리
다국어 처리란 스마트폰의 국가 설정에 따라 정해진 문자열이 해당 국가의 언어로 바뀌도록 만들어주는 역할을 한다.

아래의 코드로 예를 들어보자
```javascript
<TextView
    android:id="@+id/tv"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="value"
    android:text="@string/value"/>
```

텍스트 뷰에 값을 넣을 때 value로 넣을 수 있고 string.xml 파일에 value라는 값을 가져오도록 할 수도 있다. 앱이 커지면 레이아웃도 많아지고 일일이 텍스트 값을 변경하는 것은 쉽지 않기에 string.xml 파일을 이용하여 문자값을 관리한다. 또한, 여러 국가의 서비스를 하는 경우 한국어 뿐만 아니라 영어를 지원해야 하기에 문자열을 통해 다국어 처리 설정이 필요하다.

## 다국어 처리 설정하기
이전에는 string.xml (ko) string.xml (en) 버전을 하나씩 직접 만들어야 했었다. 하지만, 안드로이드 스튜디오의 버전이 높아지면서 **translation editor** 기능을 활용하면 쉽게 다국어 처리 설정이 가능하다.

- 자동으로 생성이 되는 res/layout/strings.xml 을 클릭한다.
- 레이아웃 상단에 Edit translations for all locations in the translations editor 의 팝업이 뜨면 우측에 open editor 라는 버튼을 클릭한다.
- 만약에 그 창을 껐거나 안나온다면 res/layout/strings.xml 파일에서 우클릭 후 - open translations editor 클릭
- '+' 버튼을 이용해서 key-value 를 추가할 수 있고 그 옆에 지구모양처럼 생긴것을 클릭하면 여러 국가를 추가할 수 있다.
- 다국어 처리가 완료되면 strings.xml 파일이 추가적으로 생긴다.

![preview](https://i.imgur.com/I6HSSmY.png)

**참고자료**
- [구글 Translation Editor](https://developer.android.com/studio/write/translations-editor?hl=ko) 공식문서 참조

## 결론
- **strings.xml 을 다국어 처리뿐만 아니라 앱에서 사용하는 문자열을 한눈에 쉽게 관리**할 수 있다.
- 구글 공식 개발 문서를 사랑합시다.