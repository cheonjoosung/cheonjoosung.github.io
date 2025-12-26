---
title: (Android/안드로이드) WebView method must be called on the same thread
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) WebView method must be called on the same thread
---

## ✨ 개요

WebView는 생성된 스레드(대부분 main/UI thread)에서만 접근해야 하는데, 다른 스레드에서 WebView 메서드를 호출했기 때문에 발생합니다.

---

## 1.  요약

- WebView는 thread-affine 객체: “생성된 스레드”에 묶임(대개 main thread)
- IO/Default 스레드에서 loadUrl, evaluateJavascript, goBack, reload 등을 호출하면 크래시
- 해결: 항상 main thread에서 WebView 호출
  - webView.post { ... } (가장 간단)
  - Handler(Looper.getMainLooper()).post { ... }
  - lifecycleScope.launch(Dispatchers.Main) { ... }

---

## 2. 에러 메시지의 의미

`WebView method must be called on the same thread`는 다음을 의미합니다.
- WebView는 내부에 렌더링 엔진/메시지 큐/상태를 갖고 있고 이 상태는 특정 스레드(=WebView를 만든 스레드)에서만 안전하게 접근 가능
- 따라서 다른 스레드에서 WebView 메서드를 호출하면 동시성/상태 오염을 막기 위해 런타임에서 강제 크래시
- 즉, “같은 스레드”는 보통 main thread이지만, 정확히는 WebView를 생성한 thread입니다.

---

## 3. 해결 방법

### 3-1 post 사용

```kotlin
webView.post {
    webView.loadUrl("https://example.com")
}
```
- 현재 스레드가 무엇이든 상관없이, WebView가 붙어있는 UI 스레드 큐로 보냄
- 코드가 짧고 실수 확률이 낮음
- WebView가 이미 detach/파괴된 상태면 실행이 무시되거나 NPE 위험이 있음 → Fragment에서는 onDestroyView() 이후 호출되지 않도록 관리

### 3.2 Handler(Looper.getMainLooper()).post { }

```kotlin
val mainHandler = Handler(Looper.getMainLooper())
mainHandler.post {
    webView.evaluateJavascript("alert('hi')", null)
}
```
- WebView가 없어도 main thread로 안전하게 전환 가능
- “UI 스레드로 보내는 통로”로 범용적으로 사용 가능

### 3.3 Coroutine에서 Main으로 전환

```kotlin
// 백그라운드 scope 라면
lifecycleScope.launch(Dispatchers.IO) {
    val data = repo.load()
    withContext(Dispatchers.Main) {
        webView.loadUrl("javascript:render(${data})")
    }
}

// main으로 실행
lifecycleScope.launch(Dispatchers.Main) {
    webView.reload()
}
```