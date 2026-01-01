---
title: (Android/안드로이드) WebView JavaScript 호출하는 방법
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) WebView JavaScript 호출하는 방법 - evaluateJavascript, loadUrl("javascript:"), 반환값 처리, 스레드/타이밍 이슈까지
---

## ✨ 개요

WebView 연동에서 가장 기본이면서도 자주 쓰는 기능이 네이티브(Android)에서 웹의 JavaScript 함수를 호출하는 것입니다. 
- 앱에서 로그인 토큰을 웹에 전달
- 네이티브에서 이벤트 발생 → 웹 UI 갱신
- WebView 로딩 완료 후 웹 초기화 함수 호출
- 웹에서 요구하는 브릿지 프로토콜(JSON)을 네이티브가 전달
- 이 글에서는 Android에서 Native → Web로 JS를 호출하는 대표 방식은 2가지가 있다
  - evaluateJavascript() (권장)
  - loadUrl("javascript:...") (레거시/제약)

---

## 1. 요약

- Android 4.4(API 19)+: evaluateJavascript()가 정석 (비동기, 반환값 받기 가능)
- loadUrl("javascript:...")는 반환값 불가/히스토리 문제/제약이 있어 보조적으로만 사용
- JS 호출은 반드시 메인 스레드에서 해야 함 → webView.post { } 권장
- onPageFinished 이후(또는 DOM Ready 이후)에 호출해야 “함수 없음” 문제를 피할 수 있음
- 문자열 인젝션은 반드시 JSON 인코딩/이스케이프 처리

---

## 2. 사전 준비: WebView 기본 설정

```kotlin
webView.settings.javaScriptEnabled = true
webView.settings.domStorageEnabled = true // 필요 시
```

---

## 3. 가장 권장: evaluateJavascript()로 호출하기 (API 19+)

### 3.1 단순 호출 (반환값 필요 없음)

```html
window.appInit = function() {
    console.log("init called");
}
```

```kotlin
fun callAppInit(webView: WebView) {
    webView.post {
        webView.evaluateJavascript("window.appInit && window.appInit();", null)
    }
}
// webView.post {}로 메인 스레드 보장 + WebView attach 상태에서 실행하는 습관이 안전합니다.
```

### 3.2 파라미터 전달 (문자열/숫자/JSON)

```html
window.setToken = function(token) {
    console.log("token:", token);
}
```

```kotlin
import org.json.JSONObject

fun callSetToken(webView: WebView, token: String) {
    val safeToken = JSONObject.quote(token) // JS 문자열 안전 처리 (따옴표/이스케이프)
    val js = "window.setToken && window.setToken($safeToken);"

    webView.post {
        webView.evaluateJavascript(js, null)
    }
}

fun callSetUser(webView: WebView, id: String, name: String) {
    val json = JSONObject().apply {
        put("id", id)
        put("name", name)
    }
    val js = "window.setUser && window.setUser(${json.toString()});"

    webView.post {
        webView.evaluateJavascript(js, null)
    }
}
```

### 3.3 반환값 받기 (evaluateJavascript의 강점)

```html
window.getVersion = function() {
    return "1.2.3";
}
```

```kotlin
fun getWebVersion(webView: WebView, onResult: (String?) -> Unit) {
    webView.post {
        webView.evaluateJavascript("window.getVersion && window.getVersion();") { value ->
            // value는 JSON 형태 문자열로 들어옴. 예: "\"1.2.3\"" 또는 "null"
            onResult(value)
        }
    }
}
```

---

## 4. 보조 방법: loadUrl("javascript:...") (레거시)

```kotlin
webView.post {
    webView.loadUrl("javascript:window.appInit && window.appInit();")
}
```
단점
- 반환값을 받을 수 없음
- 히스토리/로딩 상태에 따라 예측이 어려움
- 길고 복잡한 JS 호출에 취약
- 가능하면 evaluateJavascript()를 우선 사용하세요.

---

## 5. 결론

- Android WebView에서 Native → Web(JS) 호출은 evaluateJavascript + (main thread 보장) + (올바른 타이밍) 세 가지가 핵심입니다.
  - 가장 권장: webView.post { webView.evaluateJavascript(js, callback) }
  - 문자열/JSON은 반드시 안전하게 인코딩
  - 로딩 완료/DOM Ready 이후에 호출