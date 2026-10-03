---
title: (Android/Hilt) Hilt 의존성 주입 기초 — @HiltAndroidApp, @Inject
tags: [ Android, Hilt ]
style: fill
color: dark
description: Hilt의 기본 개념과 설정 방법, @HiltAndroidApp, @AndroidEntryPoint, @Inject 생성자 주입 사용법을 정리합니다.
---

---

## 1. 의존성 주입(DI)이란?

클래스가 필요로 하는 객체를 **직접 만들지 않고 외부에서 전달받는** 설계 방식입니다.

```kotlin
// ❌ 직접 생성: Repository 구현이 바뀌면 ViewModel도 수정, 테스트 시 가짜 객체 주입 불가
class UserViewModel {
    private val repository = UserRepository(ApiService(), UserDao())
}

// ✅ 외부 주입: ViewModel은 Repository가 어떻게 만들어지는지 모름
class UserViewModel(private val repository: UserRepository)
```

객체 생성과 연결을 수작업으로 하면 코드가 복잡해지는데, **Hilt**는 이를 자동화해주는 Android 전용 DI 라이브러리입니다. Dagger를 기반으로 하며 Android 컴포넌트(Activity, Fragment, ViewModel 등)의 생명주기에 맞춘 설정을 미리 제공합니다.

---

## 2. 설정

**프로젝트 build.gradle.kts**:
```kotlin
plugins {
    id("com.google.dagger.hilt.android") version "2.51.1" apply false
    id("com.google.devtools.ksp") version "2.0.0-1.0.22" apply false
}
```

**앱 모듈 build.gradle.kts**:
```kotlin
plugins {
    id("com.google.devtools.ksp")
    id("com.google.dagger.hilt.android")
}

dependencies {
    implementation("com.google.dagger:hilt-android:2.51.1")
    ksp("com.google.dagger:hilt-android-compiler:2.51.1")
}
```

> **kapt vs ksp**: 예전에는 `kapt`를 썼지만 현재는 빌드가 더 빠른 **KSP**가 권장됩니다.

---

## 3. @HiltAndroidApp — Hilt 진입점

Application 클래스에 붙이면 Hilt가 앱 전체에서 사용할 의존성 컨테이너를 생성합니다.

```kotlin
@HiltAndroidApp  // Hilt 코드 생성 트리거 — 앱 수준 컨테이너 초기화
class MyApplication : Application()
```

```xml
<!-- AndroidManifest.xml: 반드시 등록해야 함 -->
<application
    android:name=".MyApplication"
    ... >
```

---

## 4. @AndroidEntryPoint — 주입받을 Android 클래스 표시

Activity, Fragment, Service 등에서 Hilt 주입을 받으려면 이 어노테이션이 필요합니다.

```kotlin
@AndroidEntryPoint  // 이 Activity에서 @Inject 필드 주입 가능
class MainActivity : AppCompatActivity() {

    // @Inject 필드 주입: onCreate 이전에 Hilt가 자동으로 채워줌
    // lateinit var + 반드시 public (private 불가)
    @Inject lateinit var analytics: AnalyticsTracker

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        analytics.logEvent("main_opened")  // 바로 사용 가능
    }
}
```

지원 대상: `Application`, `Activity`, `Fragment`, `View`, `Service`, `BroadcastReceiver`
(ViewModel은 `@HiltViewModel`을 사용 — 다음 글에서 다룹니다)

> **주의**: Fragment에 `@AndroidEntryPoint`를 붙이면 그 Fragment가 속한 **Activity도 반드시** `@AndroidEntryPoint`여야 합니다.

---

## 5. @Inject 생성자 주입

가장 기본적이고 권장되는 방식입니다. 생성자에 `@Inject`를 붙이면 Hilt가 **객체 생성 방법을 학습**합니다.

```kotlin
// AnalyticsTracker 생성 방법을 Hilt에게 알려줌
// 생성자 파라미터(Logger)도 Hilt가 재귀적으로 찾아서 주입
class AnalyticsTracker @Inject constructor(
    private val logger: Logger
) {
    fun logEvent(name: String) = logger.log("event: $name")
}

class Logger @Inject constructor() {  // 파라미터 없어도 @Inject constructor() 필요
    fun log(message: String) = Log.d("App", message)
}
```

```
MainActivity 요청: AnalyticsTracker
    → Hilt: AnalyticsTracker는 Logger가 필요 → Logger 생성 → AnalyticsTracker 생성 → 주입
```

의존성 그래프를 Hilt가 컴파일 타임에 분석해 **연결이 불가능하면 빌드 에러**로 알려줍니다 (런타임 크래시 방지).

---

## 6. 주입 방식 비교

| 방식 | 코드 | 사용 시점 |
|------|------|----------|
| 생성자 주입 | `class A @Inject constructor(b: B)` | **기본 선택** — 불변, 테스트 용이 |
| 필드 주입 | `@Inject lateinit var b: B` | Activity/Fragment처럼 생성자를 직접 못 쓰는 클래스 |

---

## 7. 정리

- **DI**: 객체를 직접 만들지 않고 외부에서 받아 결합도를 낮추는 설계
- `@HiltAndroidApp`: Application에 붙여 Hilt 초기화
- `@AndroidEntryPoint`: Activity/Fragment 등 주입을 받을 Android 클래스에 표시
- `@Inject constructor`: Hilt에게 객체 생성 방법을 알려주는 기본 방법
- 의존성 그래프 오류는 **컴파일 타임**에 검출됨
- 인터페이스나 외부 라이브러리 객체는 `@Inject`만으로 불가 → 다음 글들의 Module로 해결
