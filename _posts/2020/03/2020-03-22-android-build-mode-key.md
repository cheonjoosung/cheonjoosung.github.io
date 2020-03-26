---
title: Android debug/release API Key automatically 설정
tags: [Android]
style: fill
color: dark
description: 안드로이드의 그래들 설정을 통해 debug/relase에 따라 자동으로 api key 사용하기
---

## 빌드 구성
안드로이드 앱을 일반적으로 개발을 할 때 Debug 모드로 코딩을 하고 테스트를 합니다. 그리고 앱을 스토어에 출시를 할 때 release 모드로 앱을 만들어 배포를 합니다.

구글의 애드센스를 테스트 하기 위해서는 실제 사용할 API 값과 테스트 API 값이 다르기 때문에 빌드 구성에 따라 일일히 변경해야 하는 작업을 거쳐야 합니다. 실수로 수정하지 않고 배포를 하게 된다면 광고가 정상적으로 작동하지 않을 수도 있습니다.

**1단계 : 코드추가**
- buildConfigField() or resValue() 를 통해 빌드 모드에 따라서 값을 달리 불러올 수 있습니다.
- build.gradle(Moduble: app) 파일에 아래의 코드 추가
```javascript
buildTypes {
    debug {
        minifyEnabled false

        buildConfigField("String", "BUILD_TIME", "\"debug_key\"")
        resValue("string", "build_time", "debug_key_2")
    }

    release {
        minifyEnabled false
        proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'

        buildConfigField("String", "BUILD_TIME", "\"${"API-KEY"}\"")
        resValue("string", "build_time", "${"API-KEY_2"}")
    }
}
```
- gradle sync now 클릭하여 동기화 진행
- **주의 사항** buildConfigField를 사용할 때 \"value\" 

**2단계 : 설정값 불러오기**
- gradle에서 동기화된 파일이 생성되어서 BuilfConfig 파일에 저장되어 값을 불러오거나 getString을 통해서 값을 가져올 수 있다.
```javascript
Log.i("KEY_TEST", BuildConfig.BUILD_TIME);
Log.i("KEY_TEST", getString(R.string.build_time));

/*
디버그 모드라면 로그에 debug_key 와 debug_key2가 출력
릴리즈 모드라면 로그에 API-KEY 와 API-KEY_2가 출력
*/
```
- 디버그 모드라면 debug_key값을 사용하고 릴리즈 모드라면 api-key 값을 자동적으로 사용함

**참고자료**
- [구글 gradle](https://developer.android.com/studio/build/gradle-tips?hl=ko)]

## 결론
- **키 값 활용뿐만 아니라 gradle 에서 빌드모드에 따라 다양한 설정에 대한 배움이 필요하다.**