---
title: Android√ WebViewClient 완벽 가이드 – 네비게이션/에러/SSL/리소스 제어 (Kotlin)
tags: [Android]
style: fill
color: dark
description: Android√ WebViewClient 완벽 가이드 – 네비게이션/에러/SSL/리소스 제어 (Kotlin)
---

## ✨ 개요

`WebViewClient`는 **페이지 탐색 흐름과 로딩 수명주기**를 다루는 핵심 콜백 집합입니다.  
링크 열기 경로(내부/외부), 에러 화면, SSL 처리, 리디렉션, 리소스 가로채기까지 모두 `WebViewClient` 영역이죠.  
이 글에서는 **실무에서 바로 쓰는 기본 세팅 + 콜백별 베스트 프랙티스**를 정리합니다.
---

## 1 언제 `WebViewClient`를 쓰나?

- **내부/외부 링크 분기**(스킴/도메인 화이트리스트)
- **페이지 로딩 수명주기**(`onPageStarted/Finished`)로 로딩 UI 제어
- **HTTP/네트워크 에러 처리**(대체 화면, 재시도)
- **SSL 에러 차단**(보안 우선)
- **리소스 가로채기**(헤더 주입, 간단 캐싱, 오프라인 페이지)
- **리디렉션/새 창 처리**(동일 WebView 내 유지)

> 반면 **프로그레스/파일 업로드/JS 다이얼로그/전체화면 영상/권한**은 `WebChromeClient`에 맡기세요.

---

## 2 리다이렉션

모바일 웹은 내부 리디렉션이 많습니다. 기본값으로 동일 WebView에서 처리하면 UX가 안정적입니다.
```kotlin
override fun shouldOverrideUrlLoading(
    view: WebView,
    request: WebResourceRequest
): Boolean {
    // request.isRedirect 등 조건이 있어도 false로 두어 내부에서 진행
    return false
}
```
- 만약 target="_blank"나 새 창 요구가 잦다면, 그대로 현재 WebView로 로드하거나 별도의 탭 컨트롤을 직접 구현합니다.

---

## 3 큰/헤더 주입(고급) – shouldInterceptRequest

성능/보안 이슈가 있으니 정말 필요한 요청만 가로채세요.
- HTTPS/캐시/쿠키 상호작용에 주의
- 대량 가로채기는 렌더링 저하의 원인

```kotlin
// 개념 예시 – OkHttp로 프록시 후 WebResourceResponse 반환
private fun proxyWithOkHttp(req: WebResourceRequest): WebResourceResponse? {
    val client = OkHttpClient()
    val newReq = Request.Builder()
        .url(req.url.toString())
        .apply {
            // 필요 시 헤더 주입
            header("Authorization", "Bearer ${token()}")
            req.requestHeaders.forEach { (k, v) -> header(k, v) }
        }.build()

    val resp = client.newCall(newReq).execute()
    val mime = resp.header("Content-Type")?.substringBefore(";") ?: "text/plain"
    val stream = resp.body?.byteStream() ?: return null
    return WebResourceResponse(mime, "utf-8", resp.code, resp.message, emptyMap(), stream)
}
```

---

## 4 보안 베스트 프랙티스

- 신뢰 도메인 화이트리스트 + 강제 HTTPS
- onReceivedSslError는 cancel: 사용자가 모르는 사이 우회 금지
- 파일 접근/JS 브릿지 최소화
  + settings.allowFileAccess = false(필요 시에만 true)
  + @JavascriptInterface는 필요한 메서드만 노출
- Safe Browsing & Mixed Content 차단
  + ```kotlin
    WebSettingsCompat.setSafeBrowsingEnabled(webView.settings, true)
    webView.settings.mixedContentMode = WebSettings.MIXED_CONTENT_NEVER_ALLOW
    ``` 
    
---

## 5. 결론

- 네비·로딩·에러·SSL·리소스 = WebViewClient
- 프로그레스/파일 업로드/JS 다이얼로그/미디어 권한 = WebChromeClient
- 실무 핵심은 화이트리스트 기반 라우팅과 보안 우선 에러 처리, 과도한 shouldInterceptRequest 남용 금지입니다.