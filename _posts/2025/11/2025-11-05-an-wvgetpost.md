---
title: Android/안드로이드 WebView loadUrl() vs postUrl() 정리
tags: [Kotlin]
style: fill
color: dark
description: Android/안드로이드 WebView loadUrl() vs postUrl() 완벽 가이드 — 헤더, 세션, 보안, 문제해결
---

## ✨ 개요

WebView로 페이지를 여는 방법은 크게 **GET 요청(`loadUrl`)** 과 **간단한 폼 POST(`postUrl`)** 두 가지입니다.  
둘의 차이·제약·보안 이슈를 정확히 이해하면 “로그인 유지, 헤더/토큰 전달, 폼 전송” 같은 실전 요구사항을 안전하게 처리할 수 있습니다.

---

## 1. 핵심 요약

- **일반 페이지/GET + 헤더 필요** → `loadUrl(url, headers)`
- **폼 데이터 `application/x-www-form-urlencoded` POST** → `postUrl(url, bodyBytes)`
- **JSON POST**나 복잡한 헤더 제어가 필요하면 → **WebView 내부 JS(fetch)** 또는 **네이티브 프록시(shouldInterceptRequest+OkHttp)**

---

## 2 `loadUrl()` — GET + (선택) 헤더

#### 1-1. 기본 사용
```kotlin
webView.loadUrl("https://m.example.com/home")
```

#### 1-2. 커스텀 헤더

```kotlin
val headers = mapOf(
  "Authorization" to "Bearer ${token()}",
  "Referer"       to "https://app.example.com"
)
webView.loadUrl("https://m.example.com/article/123", headers)
```

#### 1-3. 쿠키/세션과 함께 스기

```kotlin
val cm = CookieManager.getInstance()
cm.setAcceptCookie(true)
cm.setCookie("https://m.example.com", "sid=abcdef; Path=/; Secure; HttpOnly")
cm.flush()

webView.loadUrl("https://m.example.com/home")
```
- 헤더는 최초 요청에만 적용(리다이렉션 후엔 보장 X).
- 세션 유지가 목적이면 쿠키 세팅을 우선 고려하세요.

---

## 3 postUrl() — 폼 인코딩 기반 POST

postUrl()은 application/x-www-form-urlencoded 바디를 바이트 배열로 넘기는 API입니다.
즉, 폼 POST 전송에 최적화되어 있고 JSON 바디를 편하게 보내려는 용도는 아닙니다.

#### 3-1 로그인 폼 예

```kotlin
val form = "id=${URLEncoder.encode(id, "UTF-8")}&pw=${URLEncoder.encode(pw, "UTF-8")}"
val body = form.toByteArray(Charsets.UTF_8)

webView.postUrl("https://m.example.com/login", body)
// 이후 같은 도메인의 페이지는 세션 쿠키로 로그인 상태 유지
```

#### 3-2 왜 JSON엔 부적합할까?

- postUrl()은 컨텐트 타입을 강제 설정하기 어렵고,
- 리다이렉션/후속 요청 헤더 제어가 제한적입니다.
- 서버가 폼 인코딩을 기대할 때 가장 안정적으로 동작합니다.

---

## 4 JSON POST가 꼭 필요하다면 (대안) 

#### 4-1 WebView 내부 JS로 fetch 수행

```kotlin
val js = """
fetch('https://api.example.com/login',{
  method:'POST',
  headers:{'Content-Type':'application/json'},
  body: JSON.stringify({ id: 'john', pw: '***' })
}).then(r=>r.text()).then(t=>{
  document.body.innerHTML = t;
});
""".trimIndent()

webView.evaluateJavascript(js, null)
```

#### 4-2 네이티브에서 값 읽기

- shouldInterceptRequest에서 OkHttp로 직접 요청 → WebResourceResponse로 응답 전달
- 토큰/헤더/캐시 제어가 자유롭지만, 성능·보안·예외 처리가 복잡하므로 정말 필요한 경로에만 적용

---

## 5. 로딩/네비게이션과 함께 쓰기 (필수 패턴)

```kotlin
@SuppressLint("SetJavaScriptEnabled")
fun WebView.setup(onLoading: (Boolean) -> Unit = {}) = apply {
    settings.javaScriptEnabled = true
    settings.domStorageEnabled = true

    webViewClient = object : WebViewClient() {
        override fun shouldOverrideUrlLoading(v: WebView, r: WebResourceRequest): Boolean {
            val u = r.url.toString()
            return when {
                u.startsWith("tel:") || u.startsWith("mailto:") || u.startsWith("intent:") -> {
                    try { v.context.startActivity(Intent.parseUri(u, 0)) } catch (_: Exception) {}
                    true
                }
                else -> false // WebView 내부 로딩
            }
        }
        override fun onPageStarted(v: WebView, url: String?, f: Bitmap?) = onLoading(true)
        override fun onPageFinished(v: WebView, url: String?) = onLoading(false)

        override fun onReceivedSslError(v: WebView, h: SslErrorHandler, e: SslError) {
            h.cancel() // 보안 우선: 우회 금지
            v.loadUrl("file:///android_asset/ssl_blocked.html")
        }
    }
}

webView.setup { loading -> progress.isVisible = loading }
webView.loadUrl("https://m.example.com")
```

---

## 6. 결론

- 일반 페이지/헤더 필요 → loadUrl
- 폼 기반 POST → postUrl
- JSON/복잡 헤더 → JS fetch 또는 네이티브 프록시