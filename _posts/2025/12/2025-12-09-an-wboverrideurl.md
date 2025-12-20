---
title: (Android/안드로이드) WebViewClient로 URL / 커스텀 스킴 가로채서 처리하는 방법
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) WebViewClient로 URL / 커스텀 스킴 가로채서 처리하는 방법
---

## ✨ 개요

Android WebView를 사용하다 보면 다음과 같은 요구사항이 거의 반드시 등장한다.
- 웹에서 특정 링크 클릭 시 앱 기능 실행
- myapp://, intent:// 같은 커스텀 스킴 처리
- 외부 브라우저, 마켓, 전화 앱 등으로 분기 처리
- 결제/인증 페이지에서 리디렉션 제어

이 모든 것을 담당하는 핵심 진입점이 바로 `WebViewClient.shouldOverrideUrlLoading()`이다.

---

## 1.  요약

- URL 로딩 여부를 제어하려면 WebViewClient를 반드시 사용해야 함
- `shouldOverrideUrlLoading()`에서 true = WebView가 로딩하지 않음
- 커스텀 스킴(`myapp://`)은 직접 파싱해서 앱 로직으로 연결
- `intent://`, `market://`, `tel:` 등은 Intent로 처리
- HTTP/HTTPS는 기본적으로 WebView에 그대로 로딩

---

## 2. WebViewClient의 역할

WebViewClient는 WebView 내부에서 발생하는 페이지 이동, URL 로딩, 에러 처리를 가로채는 역할을 한다.

```kotlin
webView.webViewClient = WebViewClient()

```

- 이 설정이 없으면: 모든 링크 클릭 → 외부 브라우저로 이동
- 이 설정이 있으면: 링크 클릭 → WebView 내부에서 처리

---

## 3. shouldOverrideUrlLoading() 기본 구조

Android API 21 이상에서는 아래 메서드를 사용해야 한다.

```kotlin
override fun shouldOverrideUrlLoading(
    view: WebView?,
    request: WebResourceRequest?
): Boolean

```

| 반환값     | 의미                                   |
| ------- | ------------------------------------ |
| `true`  | URL을 **앱에서 직접 처리** (WebView는 로딩 안 함) |
| `false` | WebView가 **그대로 URL 로딩**              |

---

## 4. 가장 기본 적인 URL 가로채기

```kotlin
webView.webViewClient = object : WebViewClient() {
    override fun shouldOverrideUrlLoading(
        view: WebView?,
        request: WebResourceRequest?
    ): Boolean {
        val url = request?.url.toString()

        return if (url.startsWith("http")) {
            false // WebView가 로딩
        } else {
            true  // 앱에서 처리
        }
    }
}
```

---

## 5. 커스텀 스킴(myapp://) 처리하기

- 스킴 호출
  + ```html
    <a href="myapp://open/profile?id=123">프로필 열기</a>
    ```
- 처리 & 스킴 파싱
  + ```kotlin
    override fun shouldOverrideUrlLoading(
      view: WebView?,
      request: WebResourceRequest?
    ): Boolean {
      val uri = request?.url ?: return false

      if (uri.scheme == "myapp") {
          handleCustomScheme(uri)
          return true
      }

      return false
    }

    private fun handleCustomScheme(uri: Uri) {
      when (uri.host) {
          "open" -> {
              when (uri.path) {
                  "/profile" -> {
                      val id = uri.getQueryParameter("id")
                      openProfile(id)
                  }
              }
          }
      }
    }
    ```

---

## 6. intent:// 스킴 처리 (카카오, 결제, 인증 등)

웹에서 자주 사용하는 intent:// 스킴은 반드시 try-catch로 처리해야 한다.

```kotlin
override fun shouldOverrideUrlLoading(
    view: WebView?,
    request: WebResourceRequest?
): Boolean {
    val url = request?.url.toString()

    if (url.startsWith("intent://")) {
        try {
            val intent = Intent.parseUri(url, Intent.URI_INTENT_SCHEME)
            startActivity(intent)
        } catch (e: Exception) {
            e.printStackTrace()
        }
        return true
    }

    return false
}
```

---

## 7. 외부 앱 스킴 처리(tel:, mailto:, market:)

```kotlin
override fun shouldOverrideUrlLoading(
    view: WebView?,
    request: WebResourceRequest?
): Boolean {
    val uri = request?.url ?: return false

    return when (uri.scheme) {
        "tel", "mailto", "sms", "market" -> {
            startActivity(Intent(Intent.ACTION_VIEW, uri))
            true
        }
        else -> false
    }
}
```

---

## 8. 특정 도메인만 WebView에서 열기

```kotlin
private val allowedHost = "www.example.com"

override fun shouldOverrideUrlLoading(
    view: WebView?,
    request: WebResourceRequest?
): Boolean {
    val uri = request?.url ?: return false

    return if (uri.host == allowedHost) {
        false // WebView에서 로딩
    } else {
        startActivity(Intent(Intent.ACTION_VIEW, uri))
        true
    }
}
```