---
title: Android Installed Build Tools revision 31.0.0 is corrupted 해결하기
tags: [Android]
style: fill
color: dark
description: Android Installed Build Tools revision 31.0.0 is corrupted 해결하기
---

## 무엇이 문제일까?
최신 버전의 Android Studio 에서 New Proejct 후 앱 실행을 클릭하면 빌드를 하다가 아래의 문제가 발생한다.
![preview](notion://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2Fa2f156d8-f4ab-4e34-853b-3b77fb3e3fc8%2F%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-03-07_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_9.28.23.png?table=block&id=28368089-c1fd-4085-80f3-78b0e1360db0&spaceId=7cbe6c62-fb41-46f0-a9c4-08c979944f25&width=2000&userId=eb5ca5dc-904b-4779-bb7c-8114482b1122&cache=v2)
**Installed Build Tools revision 31.0.0 is corrupted** 라고 하는 원인 모를 문제가 발생한다.

이를 해결하는 방법에는 2가지가 있다.

**Gradle BuildVersion 수정**
```javascript
compileSdkVersion 30
buildToolsVersion "30.0.3"

defaultConfig {
    applicationId "com.example.patternapp"
    minSdkVersion 16
    targetSdkVersion 30
    versionCode 1
    versionName "1.0"

    testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"
}
```
compileSdkVersion & targetSdkVersion 을 30으로 맞추고 buildToosVersion 을 “30.0.3”으로 변경하는 것이다. 낮은 버전으로 변경하여 빌드를 하는 방법인데 해당 방법으로 바꾼경우 될 수도 있지만 안되는 경우가 있다.

31.0.0에 맞게 자동으로 dependancy (ktx, appcompact 등)이 맞춰져 있어서 모두 수정해야 빌드가 될 것이다. 그러므로 변경 후 동작이 잘 되면 이대로 유지를 하면 된다. 해결이 안될 경우 아래의 코드를 더 읽어보자


**Gradle Version Update**
Android Studio 에서 사용중인 Gradle 버전을 확인해보자. 만약 4.x 버전을 사용하고 있다면 7.1.x 이상으로 업데이트를 해준다.
![preview](notion://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F1222b145-4c60-4524-a30d-2b4e164b8234%2F%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-03-07_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_9.55.03.png?table=block&id=749d739d-1cde-4ff5-8e88-5f179a1f860e&spaceId=7cbe6c62-fb41-46f0-a9c4-08c979944f25&width=2000&userId=eb5ca5dc-904b-4779-bb7c-8114482b1122&cache=v2)

업데이트를 한 후에 실행을 하면 “Android Gradle plugin requires Java 11 to run. You are currently using Java 1.8” 메시지가 뜰 수도 있다.
![preview](notion://www.notion.so/image/https%3A%2F%2Fs3-us-west-2.amazonaws.com%2Fsecure.notion-static.com%2F84991909-30df-4452-a467-6dcdd0280485%2F%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-03-07_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_10.02.12.png?table=block&id=853ad47a-982f-4fdf-9c73-ffc2072e5b8e&spaceId=7cbe6c62-fb41-46f0-a9c4-08c979944f25&width=2000&userId=eb5ca5dc-904b-4779-bb7c-8114482b1122&cache=v2)

좌측상단 - AndroidStudio > Prefrerence > Build, Execution, Deployment > Build Tools> Grdle 에서 Gradle JDK를 11로 변경한 후 OK를 누른다.

빌드 후 앱을 시작하면 정상적으로 작동하는 것을 알 수 있다.
