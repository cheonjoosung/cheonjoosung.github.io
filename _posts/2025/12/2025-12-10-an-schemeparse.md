---
title: (Android/안드로이드) Scheme 파싱하는 법 - 딥링크 스킴 파싱
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) Scheme 파싱하는 법 - 딥링크 스킴 파싱
---

## ✨ 개요

안드로이드에서 “앱을 특정 화면으로 바로 열기”를 구현할 때 가장 자주 등장하는 키워드가 스킴(scheme) 딥링크입니다. 예를 들어 아래 링크를 누르면 앱이 열리면서 특정 화면으로 이동하게 만들 수 있습니다.

- `myapp://host-action?val=123`
- `myapp://open/profile?id=10`
- `https://example.com/event/42 (App Link)`

그런데 실제 구현 단계에서 많은 개발자가 헷갈려 하는 부분이 있습니다.
- scheme / host / path / query param의 역할 구분
- Intent의 action, **data(Uri)**는 무엇이고 언제 쓰는지
- `intent.dataString`을 어디까지 믿어도 되는지
- “host-action” 같은 규칙을 어떻게 파싱하면 유지보수하기 좋은지

---

## 1.  요약

- 딥링크의 핵심은 Intent.data (Uri)를 파싱하는 것
- Uri는 크게 scheme, host, path, queryParameter로 나뉜다
- Intent.action은 “이 인텐트가 어떤 목적/형태로 왔는지”를 구분하는 힌트
- 파싱은 dataString 문자열 조작보다 Uri API를 우선 사용하자
- 스킴 규칙은 “라우팅 테이블(when/Map)”로 관리하면 유지보수성이 좋아진다

---

## 2. 스킴 딥링크 구성요소: scheme / host / path / query

```kotlin
myapp://host-action/path/to/screen?val=123&foo=bar
```

| 구성요소        | 예시                | 의미                  |
| ----------- | ----------------- | ------------------- |
| scheme      | `myapp`           | 어떤 프로토콜/앱인지 식별      |
| host        | `host-action`     | 앱 내부에서 큰 분기 기준으로 사용 |
| path        | `/path/to/screen` | 세부 화면 라우팅에 사용       |
| query param | `val=123`         | 화면에 전달할 데이터         |

안드로이드에서 이 값들은 대부분 Uri로 파싱합니다.

---

## 3. Intent에서 딥링크 정보 꺼내기: action vs data(Uri)

딥링크로 앱이 실행되면 보통 Activity.onCreate() 또는 onNewIntent()에서 Intent를 받습니다.

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    handleDeepLink(intent)
}

override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    handleDeepLink(intent)
}
```

- action이란?
  + 대표적으로 딥링크는 보통 다음 액션으로 들어옵니다.
  + Intent.ACTION_VIEW
  + 즉 “어떤 URL을 열어달라”는 의미입니다.
  + ```kotlin
    val action = intent.action // 보통 ACTION_VIEW
    ```
- data(Uri)란?
  + 딥링크의 실제 URL(스킴 문자열)이 들어 있습니다. 
  + ```kotlin
    val uri: Uri? = intent.data
    ```
- Uri 파싱 핵심 API
  + ```kotlin
    val uri = intent.data ?: return

    val scheme = uri.scheme            // myapp
    val host = uri.host                // host-action
    val path = uri.path                // /path/to/screen
    val segments = uri.pathSegments    // [path, to, screen]
    val queryVal = uri.getQueryParameter("val") // "123"
    val raw = uri.toString()           // 전체 문자열\
    ``` 
  + 가능하면 지양하고, 아래처럼 Uri API로 거의 다 해결하는 것을 권장합니다.
  + 문자열 replace/removePrefix는 예외 케이스(인코딩, 슬래시 누락, 쿼리 혼재)에 약함  
- Manifest 설정 예시(스킴 딥링크)
  + ```xml
    <activity
        android:name=".DeepLinkActivity"
        android:exported="true">
        <intent-filter>
            <action android:name="android.intent.action.VIEW" />
            <category android:name="android.intent.category.DEFAULT" />
            <category android:name="android.intent.category.BROWSABLE" />
    
            <data
                android:scheme="myapp"
                android:host="host-action" />
        </intent-filter>
    </activity>
    ``` 
  + BROWSABLE이 있어야 브라우저/웹뷰에서 호출 가능
  + exported=true는 Android 12+에서 필수 조건이 될 수 있음(필터가 있으면 명시 필요)

---

## 4. 실무 파싱 패턴 1: host 기반 라우팅 (host-action 스타일)

구조
- scheme: test
- host: host-action
- param: val
- 파싱 로직: intent.dataString.removePrefix("$scheme$host")

이 방식은 가능하지만, 권장되는 안정적 접근은 아래처럼 Uri를 기준으로 분해하는 것입니다.

```kotlin
test://host-action?val=hello

private fun handleDeepLink(intent: Intent) {
    val uri = intent.data ?: return
    val scheme = uri.scheme ?: return
    val host = uri.host ?: return

    if (scheme != "test") return

    when (host) {
        "host-action" -> {
            val value = uri.getQueryParameter("val")
            // value로 화면 이동/로직 처리
            navigateHostAction(value)
        }
        else -> {
            // unknown host
        }
    }
}
```

- 쿼리 파라미터/인코딩 이슈에 강함
- 슬래시 유무(test://host-action vs test:host-action) 같은 변형에도 안전
- 라우팅 구조가 명확해짐

---

## 5. 실무 파싱 패턴 2: path 기반 라우팅 (권장 구조)

Host는 도메인처럼 고정하고, path로 화면을 나누는 방식이 유지보수에 유리합니다.
- `myapp://open/profile?id=10`
- `myapp://open/settings`
- `myapp://open/web?url=https%3A%2F%2F...`

```kotlin
when (uri.host) {
    "open" -> {
        when (uri.path) {
            "/profile" -> {
                val id = uri.getQueryParameter("id") ?: return
                openProfile(id)
            }
            "/settings" -> openSettings()
            "/web" -> {
                val url = uri.getQueryParameter("url") ?: return
                openWeb(url)
            }
        }
    }
}
```

---

## 6. 파싱 시 주의사항 (실무에서 많이 터지는 것들)

- URL 인코딩(특히 url 파라미터)
  + ?url=https%3A%2F%2F... 같은 값은 getQueryParameter()가 디코딩해서 줍니다.
  + 문자열 조작으로 파싱하면 인코딩 이슈로 깨질 수 있습니다.
- onNewIntent() 미처리
  + singleTask/singleTop 구성에서 딥링크가 새로 들어오면 onCreate()가 아니라 onNewIntent()로 들어옵니다.
  + 딥링크 처리 로직은 재사용 가능하게 분리하세요.
- “규칙”이 늘어날수록 when 지옥
  + 딥링크가 많아지면 라우팅 테이블을 별도 클래스로 분리하고, sealed class로 모델링하는 편이 좋습니다.