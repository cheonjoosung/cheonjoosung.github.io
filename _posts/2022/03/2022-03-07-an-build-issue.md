---
title: Android Installed Build Tools revision 31.0.0 is corrupted 해결하기
tags: [Android]
style: fill
color: dark
description: Android Installed Build Tools revision 31.0.0 is corrupted 해결하기
---

## 무엇이 문제일까?
최신 버전의 Android Studio 에서 New Proejct 후 앱 실행을 클릭하면 빌드를 하다가 아래의 문제가 발생한다.
![preview](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/a2f156d8-f4ab-4e34-853b-3b77fb3e3fc8/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-03-07_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_9.28.23.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220307%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220307T133404Z&X-Amz-Expires=86400&X-Amz-Signature=2e4ed0e09cb52ee10c5e03e2fb64894beda9a517e3570733ac46d0de5f0be86d&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA%25202022-03-07%2520%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE%25209.28.23.png%22&x-id=GetObject)
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
![preview](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/1222b145-4c60-4524-a30d-2b4e164b8234/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-03-07_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_9.55.03.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220307%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220307T131200Z&X-Amz-Expires=86400&X-Amz-Signature=a468918781e48bbbdc5b8696ce98487602d4de55178a1db02b7ce3de7c47c31d&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA%25202022-03-07%2520%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE%25209.55.03.png%22&x-id=GetObject)

업데이트를 한 후에 실행을 하면 “Android Gradle plugin requires Java 11 to run. You are currently using Java 1.8” 메시지가 뜰 수도 있다.
![preview](https://s3.us-west-2.amazonaws.com/secure.notion-static.com/84991909-30df-4452-a467-6dcdd0280485/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-03-07_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_10.02.12.png?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Content-Sha256=UNSIGNED-PAYLOAD&X-Amz-Credential=AKIAT73L2G45EIPT3X45%2F20220307%2Fus-west-2%2Fs3%2Faws4_request&X-Amz-Date=20220307T133158Z&X-Amz-Expires=86400&X-Amz-Signature=9a49d8cf0861d02f061a6875d46c243fdbd229d34a64f92440d5203b0b2aa154&X-Amz-SignedHeaders=host&response-content-disposition=filename%20%3D%22%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA%25202022-03-07%2520%25E1%2584%258B%25E1%2585%25A9%25E1%2584%2592%25E1%2585%25AE%252010.02.12.png%22&x-id=GetObject)

좌측상단 - AndroidStudio > Prefrerence > Build, Execution, Deployment > Build Tools> Grdle 에서 Gradle JDK를 11로 변경한 후 OK를 누른다.

빌드 후 앱을 시작하면 정상적으로 작동하는 것을 알 수 있다.
