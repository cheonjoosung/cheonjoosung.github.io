---
title: Android/안드로이드 WebView에 localStorage 사용하는 방법
tags: [Kotlin]
style: fill
color: dark
description: Android/안드로이드 WebView에 localStorage 사용하는 방법 – 설정, 브릿지, 퍼시스턴스, 초기화까지
---

## ✨ 개요

`localStorage`는 **도메인(오리진) 단위로 영구 저장**되는 클라이언트 저장소입니다.  
안드로이드 WebView에서도 **설정만 맞추면** 브라우저처럼 동작하며, 로그인 상태 유지, 온보딩 플래그, UI 캐시 등에 유용합니다.

---

## 1. 핵심 요약

- WebView에서 `localStorage`를 쓰려면 **`domStorageEnabled = true`**.
- 저장 범위는 **오리진(프로토콜+호스트+포트)** 기준. 앱을 재시작해도 **삭제 전까지 유지**.
- 앱 내 여러 WebView라도 같은 오리진이면 **같은 저장소**를 공유.
- **초기화/삭제**는 `WebStorage.getInstance()`로 제어(쿠키와 별개!).
- `file://`은 OS/버전별로 동작이 엇갈릴 수 → **https 또는 `WebViewAssetLoader`(content://)** 권장.

---

## 2 필수 설정

```kotlin
@SuppressLint("SetJavaScriptEnabled")
fun WebView.enableDomStorage() = apply {
        settings.javaScriptEnabled = true
        settings.domStorageEnabled = true      // ★ localStorage/sessionStorage 사용
    }
```

---

## 3 JS에서 사용하는 방법

```java
// 저장
localStorage.setItem("token", "abc123");
// 읽기
const token = localStorage.getItem("token"); // "abc123"
// 삭제
localStorage.removeItem("token");
// 전체 초기화
localStorage.clear();
```

---

## 4 네이티브 ↔ JS 브릿지 연동 

#### 4-1 네이티브에서 값 쓰기

```kotlin
webView.evaluateJavascript(
    "localStorage.setItem('login','true'); undefined;"
) { /* callback: null (undefined) */ }
```

#### 4-2 네이티브에서 값 읽기

```kotlin
webView.evaluateJavascript(
    "localStorage.getItem('login');"
) { json -> 
    // json은 JSON 문자열 형태("true" 또는 null)
    val value = json?.trim('"')
}
```

---

## 5. 퍼시스턴스(저장 지속성) 이해

- 오리진 단위로 저장: https://m.example.com과 https://example.com은 다른 저장소.
- 앱을 종료/재시작해도 남아 있음(앱 데이터 삭제, WebStorage 삭제, OS가 캐시 정리하지 않는 한).
- 같은 앱 내 여러 WebView 인스턴스라도 동일 오리진이면 공유.
- sessionStorage는 탭/인스턴스 세션 한정으로 앱 종료 혹은 WebView 파괴 시 사라짐.

---

## 6. 초기화 & 정리(로그아웃/프라이버시)

```kotlin
val ws = WebStorage.getInstance()

// 전체 WebStorage 삭제(모든 오리진)
ws.deleteAllData()

// 특정 오리진만
ws.deleteOrigin("https://m.example.com")

// 현재 저장소 용량/오리진 목록 조회 후 조건부 삭제
ws.getOrigins { origins ->
    origins.forEach { origin ->
        // origin.origin, origin.quota, origin.usage
    }
}
```
- CookieManager의 removeAllCookies()는 쿠키만 지웁니다.
- localStorage는 WebStorage에서 별도로 관리합니다.

---

## 7. 실무 팁: 안정성·보안·운영

#### 7-1. 오리진, 스킴 주의

- file:// 컨텐츠는 안드로이드 버전에 따라 localStorage 지속성이 달라질 수 있습니다.
- 앱 번들 HTML을 쓸 때는 **WebViewAssetLoader**로 https://appassets.androidplatform.net/... 처럼 가짜 안전 오리진을 제공하세요.
  + 이렇게 하면 localStorage가 오리진 규칙대로 안정적으로 유지됩니다.

#### 7-2. 혼합콘텐츠/세이프 브라우징(권장)

```kotlin
WebSettingsCompat.setSafeBrowsingEnabled(webView.settings, true)
webView.settings.mixedContentMode = WebSettings.MIXED_CONTENT_NEVER_ALLOW
```

#### 7-3. 민감정보 저장 금지

- localStorage는 암호화되지 않음 + JS에서 접근 가능.
- 토큰/개인정보는 만료가 짧거나 서버 세션 중심으로 설계하고, 필요 시 네이티브 보안 저장소(Keystore, EncryptedSharedPreferences) 사용.