---
title: (Android/안드로이드) WebView JavaScript Bridge 설계 패턴 - AppBridge 구조 설계 가이드
tags: [ Android,Kotlin ]
style: fill
color: dark
description: (Android/안드로이드) WebView addJavascriptInterface 기반 JavaScript Bridge를 안전하고 확장 가능하게 설계하는 AppBridge 아키텍처 패턴 정리 (보안, 콜백, Reflection 구조 포함)
---

## 개요

- WebView에서 JS ↔ Android 통신을 구현할 때 가장 흔히 사용하는 방식은: `webView.addJavascriptInterface(bridge, "AppBridge")`
- 하지만 단순히 메서드를 몇 개 추가하는 방식은 다음과 같은 문제를 만듭니다:
  - 기능이 늘어날수록 클래스가 비대해짐
  - 보안 취약점 발생 가능
  - 콜백 구조 난잡
  - 모듈화 불가능
  - 유지보수 어려움

---

## 1. 구조

- ```text
  JS → window.AppBridge.method()
        ↓
  @JavascriptInterface
        ↓
  Android 처리
        ↓
  evaluateJavascript() 로 콜백
        ↓
  JS callback
  ```

---

## 2. 단순 구현 방식 (권장 ❌)

```kotlin
class SimpleBridge {

    @JavascriptInterface
    fun showToast(message: String) {
        Toast.makeText(context, message, Toast.LENGTH_SHORT).show()
    }

    @JavascriptInterface
    fun openCamera() {
        // ...
    }
}
```

- 문제점
  - 기능 증가 = 메서드 폭증
  - 클래스 비대화
  - 테스트 어려움
  - 모듈 분리 불가

---

### 3. AppBridge 설계 목표

- ✔ 확장 가능
- ✔ 모듈 분리 가능
- ✔ 보안 통제 가능
- ✔ 콜백 구조 통일
- ✔ Reflection 기반 자동 라우팅 가능

---

## 4. AppBridge 권장 아키텍처

### 📌 JS 호출 형식 통일

- JS에서 이렇게 호출하도록 강제:

```html
window.AppBridge.postMessage(JSON.stringify({
    module: "Image",
    method: "showImage",
    params: {
    url: "https://..."
},
    callbackId: "cb_1"
}))
```

### 📌 Android 공통 Bridge Entry

```kotlin
class AppBridge(
    private val dispatcher: BridgeDispatcher
) {

    @JavascriptInterface
    fun postMessage(json: String) {
        dispatcher.dispatch(json)
    }
}
```

### 📌 Dispatcher 설계

```kotlin
class BridgeDispatcher(
    private val modules: Map<String, BridgeModule>,
    private val webView: WebView
) {

    fun dispatch(json: String) {
        val message = Gson().fromJson(json, BridgeMessage::class.java)

        val module = modules[message.module] ?: return
        val result = module.invoke(message.method, message.params)

        sendCallback(message.callbackId, result)
    }

    private fun sendCallback(callbackId: String?, result: Any?) {
        if (callbackId == null) return

        val js = """
            window.__appBridgeCallback &&
            window.__appBridgeCallback("$callbackId", ${Gson().toJson(result)});
        """.trimIndent()

        webView.post {
            webView.evaluateJavascript(js, null)
        }
    }
}
```

### 📌 BridgeMessage 모델

```kotlin
data class BridgeMessage(
    val module: String,
    val method: String,
    val params: JsonObject?,
    val callbackId: String?
)
```

---

## 5. 모듈화 패턴

### 📌공통 인터페이스

```kotlin
interface BridgeModule {
    fun invoke(method: String, params: JsonObject?): Any?
}
```

### 📌ImageModule 예시

```kotlin
class ImageModule : BridgeModule {

    override fun invoke(method: String, params: JsonObject?): Any? {
        return when (method) {
            "showImage" -> {
                val url = params?.get("url")?.asString
                showImage(url)
                mapOf("status" to "ok")
            }
            else -> null
        }
    }

    private fun showImage(url: String?) {
        // 이미지 뷰어 실행
    }
}
```

### 등록

```kotlin
val dispatcher = BridgeDispatcher(
    modules = mapOf(
        "Image" to ImageModule(),
        "App" to AppModule()
    ),
    webView = webView
)

webView.addJavascriptInterface(AppBridge(dispatcher), "AppBridge")
```

---

## 6. Reflection 기반 자동 라우팅 (고급)

method 이름과 함수 이름을 자동 매핑 가능:

```kotlin
class ReflectionModule : BridgeModule {

    override fun invoke(method: String, params: JsonObject?): Any? {
        val function = this::class.members
            .firstOrNull { it.name == method } ?: return null

        return function.call(this, params)
    }

    fun showImage(params: JsonObject?) {
        // ...
    }
}
```

---

## 7. 보안 주의사항 (매우 중요)

- @TargetApi(Build.VERSION_CODES.JELLY_BEAN_MR1)
  - addJavascriptInterface는 17 이상에서만 안전
- 외부 URL 로드 금지
  - ```kotlin
    override fun shouldOverrideUrlLoading(...): Boolean {
        return !url.startsWith("https://trusted-domain.com")
    }
    ```
- module whitelist 적용
  - `if (!allowedModules.contains(message.module)) return`
- 민감 정보 직접 전달 금지
  - 토큰, 사용자 ID, 암호화 키

---

## 8. AppBridge 설계 장점

| 항목    | 단순 방식 | AppBridge 구조 |
| ----- | ----- | ------------ |
| 확장성   | 낮음    | 높음           |
| 모듈화   | 불가    | 가능           |
| 보안 통제 | 어려움   | 가능           |
| 유지보수  | 어려움   | 용이           |
| 콜백 통일 | 없음    | 있음           |