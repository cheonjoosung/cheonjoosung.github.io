---
title: Android jitpack 라이브러리 배포
tags: [Android]
style: fill
color: dark
description: Android jitpack 라이브러리 배포
---

## 설명
팝업 메뉴를 보여줄 때 안드로이드에서 제공하는 menu 를 활용하여 개발이 가능하다. 하지만, 아이콘을 넣고 추가적인 액션이 필요한 팝업 메뉴는 안드로이드 기본 메뉴로만은 불가능 하다.


# 1. 라이브러리 만들기
1. 새로운 프로젝트를 생성한다.
2. File > new > new Module > Android Library 선택하고 만들면 끝
3. MyLog 클래스 만들고 log(msg T) { log.e(”MyLog”, “$T”) } 처럼 확인가능한 간단한 코드 추가
4. 깃헙에 가서 Relase 선택하여 TAG 생성 (그래들에서 의존성 추가할때 버전)

![preview](https://github.com/cheonjoosung/cheonjoosung/blob/master/image/jitpack/jitpack1.png?raw=true)

# 2. jitpack 사용
![preview](https://github.com/cheonjoosung/cheonjoosung/blob/master/image/jitpack/jitpack2.png?raw=true)

1. [https://jitpack.io/](https://jitpack.io/#cheonjoosung/Android-Simple-Popup) 으로 들어가기
2. 위 사진에 주소 : 본인 깃헙의 오픈소스 및 라이브러리 URL + Look Up 클릭
3. Version & Log 확인
4. 프로젝트에서 Gradle 버전에 따라 maven { url ‘https://jitpack.io’ } 추가


```gradle
// 과거버전
allprojects {
    repositories {
        ...
        maven { url 'https://jitpack.io' }
    }
}

// 최신버전
pluginManagement {
    repositories {
        gradlePluginPortal()
        google()
        maven { url "https://jitpack.io" } // add this line
        mavenCentral()
    }

}
dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        google()
        maven { url "https://jitpack.io" } // add this line

    }
}

dependencies {
	implementation 'com.github.{githubID}:{Project:TAG}'
	//implementation 'com.github.cheonjoosung:Android-Simple-Popup:0.0.2'
}
```
