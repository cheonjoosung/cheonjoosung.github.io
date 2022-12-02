---
title: Android buildSrc 의존성 관리하기 with Kotlin DSL | Dependency 관리
tags: [Android, Kotlin]
style: fill
color: dark
description: Android buildSrc 의존성 관리하기 with Kotlin DSL | Dependency 관리
---

## buildSrc 디렉토리 만들기
![preview](https://user-images.githubusercontent.com/13310269/205278160-065d0abf-5eca-4c0b-a0cb-95d3d0de4ff4.png)
- Android Studio 에서 파일구조를 보여주는 것을 Project 로 변경 
- 루트 프로젝트 우클릭 > Directory 추가 
- 파일명 buildSrc

## build.gradle.kts 파일 추가
buildSrc 우클릭하고 File 생성하여 build.gradle.kts 파일을 만들고 아래의 내용르 복사붙여넣기 해주세요.

```gradle
plugins {
    `kotlin-dsl` // enable the Kotlin-DSL
}

repositories {
    google()
    mavenCentral()
}
```

## src/main/kotlin or src/main/java 추가
![preview](https://user-images.githubusercontent.com/13310269/205278165-476820bd-9d3c-4a1b-9f86-207dc3b64f6a.png)
- buildSrc 에 사용하는 언어에 따라 src/main/java or src/main/kotlin 을 추가

## Versions.kt 파일생성
```kotlin
object AndroidX {

    const val CORE_KTX = "androidx.core:core-ktx:1.9.0"
    const val APPCOMPAT = "androidx.appcompat:appcompat:1.5.1"
    const val CONSTRAINTLayout = "androidx.constraintlayout:constraintlayout:2.1.4"
    const val LIFECYCLE_LIVEDATA_KTX = "androidx.lifecycle:lifecycle-livedata-ktx:2.5.1"
    const val LIFECYCLE_VIEWMODEL_KTX = "androidx.lifecycle:lifecycle-viewmodel-ktx:2.5.1"
    const val NAVIGATION_FRAGMENT_KTX = "androidx.navigation:navigation-fragment-ktx:2.5.3"
    const val NAVIGATION_UI_KTX = "androidx.navigation:navigation-ui-ktx:2.5.3"
}

object Google {
    const val MATERIAL = "com.google.android.material:material:1.7.0"
}
```

```gradle
dependencies {

    implementation(Google.MATERIAL)

    implementation(AndroidX.CORE_KTX)
    implementation(AndroidX.APPCOMPAT)
    implementation(AndroidX.CONSTRAINTLayout)
    implementation(AndroidX.LIFECYCLE_LIVEDATA_KTX)
    implementation(AndroidX.LIFECYCLE_VIEWMODEL_KTX)
    implementation(AndroidX.NAVIGATION_FRAGMENT_KTX)
    implementation(AndroidX.NAVIGATION_UI_KTX)
}
```
- gradle 파일에서 클래스명.변수로 사용하면 된다.

## 결론
- 이렇게 사용하면 라이브러리 의존성 관리에 편하기 때문이라고 한다.
