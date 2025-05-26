---
title: Android 앱의 버전 정보 확인하는 방법 (BuildConfig.VERSION_NAME, VERSION_CODE)
tags: [ Android ]
style: fill
color: dark
description: Android 앱의 버전 정보를 코드에서 확인하고 UI에 표시하는 방법을 BuildConfig를 이용해 알아봅니다.
---

## ✨ 개요

Android 앱에서 **버전 정보는 앱 관리, 업데이트, 고객 지원, 조건 분기** 등에 매우 중요하게 사용됩니다.

이번 포스트에서는 코드 내에서 **앱의 버전 이름(versionName)** 및 **버전 코드(versionCode)** 를 확인하는 방법과, 실전에서 자주 사용되는 **버전 출력 UI 예시**를 함께 소개합니다.

---

## 1. ✅ BuildConfig 란?


`BuildConfig`는 Gradle 빌드 시 자동으로 생성되는 클래스입니다.  
이 클래스는 앱의 **버전, 디버그 여부, 빌드 설정 값** 등을 포함하고 있어 다양한 환경 분기에 활용됩니다.

---

## 2. ✅ 사용 방법

```kotlin
val versionName = BuildConfig.VERSION_NAME
val versionCode = BuildConfig.VERSION_CODE

binding.versionTextView.text = "v$versionName ($versionCode)"

if (versionCode < 10) {
    Toast.makeText(this, "앱 업데이트가 필요합니다", Toast.LENGTH_SHORT).show()
} else {
    // 로직 실행
}
```

---

## 3. 🔍  BuildConfig.VERSION_NAME & VERSION_CODE 차이

| 항목             | 설명                                              |
| -------------- | ----------------------------------------------- |
| `VERSION_NAME` | 사용자가 보는 앱 버전 문자열 (예: "1.2.0")                   |
| `VERSION_CODE` | 내부적으로 사용하는 정수 버전 (예: 102), 업데이트 비교 시 사용         |
| 설정 위치          | `build.gradle(:app)`의 `defaultConfig` 블록 내에서 정의 |

- app 수준의 gradle
> defaultConfig { } 블록 안에 versionCode, versionName 확인이 가능

추가적으로
- PlayStore 배포 시 versionCode는 반드시 증가해야 함
- 앱 내부에서 버전에 따라 기능 분기 시 유용하게 활용 가능
- BuildConfig.DEBUG로 디버그 여부도 확인 가능

---

## 4.🧠 **결론**

- BuildConfig.VERSION_NAME과 VERSION_CODE를 통해 앱 버전을 쉽게 가져올 수 있습니다.
- 버전 정보는 UI에 표시하거나 조건 분기 로직에 유용하게 활용되므로, 앱 개발 시 자주 사용되는 필수 기능입니다.
- 팀 프로젝트나 배포 전 자동화된 버전 관리에도 중요한 역할을 합니다.

🚀 Tip: 버전 정보를 앱 설정 화면, 로그인 화면, 또는 앱 하단에 표시해 사용자와의 소통 품질을 높여보세요.