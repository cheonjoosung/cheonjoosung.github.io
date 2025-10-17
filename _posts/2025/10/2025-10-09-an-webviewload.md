---
title: Android WebView 로딩 방식 총정리
tags: [Android]
style: fill
color: dark
description: Android WebView 로딩 방식 총정리 – `loadUrl`, `postUrl`, `loadDataWithBaseURL` 제대로 쓰기 (Kotlin)
---

## ✨ 개요

WebView로 페이지를 불러오는 방법은 여러 가지가 있습니다.  
**GET/헤더 추가 → `loadUrl`**, **간단한 폼 POST → `postUrl`**, **HTML 문자열 렌더링/상대경로 지원 → `loadDataWithBaseURL`**.  
각 메서드는 쓰임새와 제약이 다르니, 상황에 맞게 선택해야 합니다.

---

## 1 언제 무엇을 쓰나?

| 시나리오 | 추천 메서드 | 비고 |
|---|---|---|
| 일반 URL GET, 커스텀 헤더 첨부 | `loadUrl(url, headers)` | 토큰/리퍼러 등 추가 가능 |
| 간단한 **폼 데이터** POST(`x-www-form-urlencoded`) | `postUrl(url, postData)` | 바디는 **URL-encoded 바이트**. **JSON POST엔 부적합** |
| 앱에서 만든 HTML 문자열 렌더링(상대 경로·쿠키·CSP 유지) | `loadDataWithBaseURL(baseUrl, data, mime, enc, history)` | `baseUrl`에 **도메인**을 주면 쿠키/상대경로가 정상 동작 |
| 아주 간단한 문자열 렌더링(상대경로 필요 없음) | `loadData(data, mime, enc)` | 상대 경로·쿠키 연동 없음(권장 X) |

> **핵심**: 서버 페이지는 `loadUrl`, 폼 POST만 필요하면 `postUrl`,  
> **HTML 문자열 렌더링·상대경로·쿠키 연동**이 필요하면 `loadDataWithBaseURL`.

---

## 2 `loadUrl` – 가장 기본적인 GET 로딩 (+헤더)

```kotlin
@SuppressLint("SetJavaScriptEnabled")
fun WebView.setupBasic() = apply {
        settings.javaScriptEnabled = true
        settings.domStorageEnabled = true
    }

val headers = mapOf(
    "Authorization" to "Bearer ${token()}",
    "Referer" to "https://app.example.com"
)

webView.setupBasic()
webView.loadUrl("https://m.example.com/article/123", headers)
```
- 쿠키 연동 필요하다면
  + ```kotlin
    val cm = CookieManager.getInstance()
    cm.setAcceptCookie(true)
    cm.setCookie("https://m.example.com", "sid=abcdef; Path=/; Secure")
    CookieManager.getInstance().flush()
    ``` 

## 3 `postUrl` – 간단 폼 POST (application/x-www-form-urlencoded)

```kotlin
val form = "id=${URLEncoder.encode(id, "UTF-8")}&pw=${URLEncoder.encode(pw, "UTF-8")}"
val postData = form.toByteArray(Charsets.UTF_8)

webView.postUrl("https://m.example.com/login", postData)
```

- postUrl은 폼 인코딩 전제(바디를 직접 바이트로 넣는 형태). `Content-Type`을 임의로 바꾸기 어렵고, JSON 바디 전송엔 부적합합니다.
- JSON 등을 POST해야 한다면:
  + 대안 1: loadDataWithBaseURL로 로드한 HTML에서 fetch() 로 API 호출
  + 대안 2: shouldInterceptRequest + OkHttp로 프록시(고급, 신중히)
  + 대안 3: 서버에서 GET→리다이렉션→필요 파라미터 처리 설계

---

## 4 `loadDataWithBaseURL` – HTML 문자열 + 상대경로/쿠키/CSP 유지

```kotlin
val html = """
<!doctype html>
<html>
  <head>
    <meta charset="utf-8">
    <link rel="stylesheet" href="/assets/styles.css">
  </head>
  <body>
    <h1>Hello WebView</h1>
    <img src="/images/logo.png">
    <script src="/assets/app.js"></script>
  </body>
</html>
""".trimIndent()

webView.loadDataWithBaseURL(
    /* baseUrl   = */ "https://m.example.com", // ★ 도메인을 주면 쿠키·상대경로 OK
    /* data      = */ html,
    /* mimeType  = */ "text/html",
    /* encoding  = */ "UTF-8",
    /* historyUrl= */ null
)
```
- 왜 baseUrl이 중요한가?
  + 상대 경로(/images/...)를 어느 호스트로 해석할지 기준 제공
  + 쿠키 도메인(m.example.com)의 세션/로그인 상태 공유
  + CSP/리소스 정책도 해당 오리진 기준으로 적용

> 반면 **loadData**는 base 없이 렌더링하므로 상대경로/쿠키 연동이 깨집니다.
> 특별한 이유가 없다면 loadDataWithBaseURL을 쓰세요.   

 
---

## 5. 결론

- 서버 페이지/헤더 추가 → loadUrl
- 간단 폼 POST → postUrl(폼 인코딩 전제, JSON엔 부적합)
- HTML 문자열+상대경로/쿠키 → loadDataWithBaseURL(권장)
- 보안은 HTTPS·혼합콘텐츠 차단·SSL 우회 금지·XSS 방지로 기본값을 안전하게.