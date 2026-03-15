---
title: (Android/안드로이드) WebView Mixed Content / HTTPS 오류 해결 방법 총정리
tags: [ Android,Kotlin ]
style: fill
color: dark
description: (Android/안드로이드) WebView에서 발생하는 Mixed Content, ERR_CLEARTEXT_NOT_PERMITTED, SSL 오류 등의 HTTPS 관련 이슈 원인과 해결 방법 정리
---

## 개요

- WebView에서 웹 페이지를 로드할 때 다음과 같은 오류를 자주 만나게 됩니다.
  - Mixed Content 오류
  - ERR_CLEARTEXT_NOT_PERMITTED
  - `SSL Error`
  - HTTPS 페이지에서 HTTP 리소스 차단
- 특히 Android 9(API 28) 이후부터 보안 정책이 강화되면서 이러한 문제가 더 자주 발생합니다.

---

## 1. Mixed Content란 무엇인가?

- ```text
  HTTPS 페이지
    ↓
  HTTP 리소스 요청
  ```
- ```html
  <!--  https://example.com/index.html 페이지 에서 --> 
  <script src="http://cdn.example.com/script.js"></script>
  <img src="http://image.example.com/a.png">
  ```
  + 이 경우 HTTPS 페이지에서 HTTP 리소스를 호출하게 됩니다.
  + 브라우저와 WebView는 이를 보안 위험으로 판단하고 차단합니다.

---

## 2. WebView에서 Mixed Content 기본 정책

- | Android 버전   | 정책                  |
  | ------------ | ------------------- |
  | Android 5~8  | Mixed Content 일부 허용 |
  | Android 9 이상 | 기본 차단               |
- (HTTPS 페이지 → HTTP 리소스) 요청은 기본적으로 막힘

---

### 3. Mixed Content 허용 방법

- ```kotlin
  webView.settings.apply {
    javaScriptEnabled = true
    mixedContentMode = WebSettings.MIXED_CONTENT_ALWAYS_ALLOW
  }
  ```
- MixedContentMode 옵션
  + | 값                                | 설명    |
    | -------------------------------- | ----- |
    | MIXED_CONTENT_NEVER_ALLOW        | 완전 차단 |
    | MIXED_CONTENT_COMPATIBILITY_MODE | 일부 허용 |
    | MIXED_CONTENT_ALWAYS_ALLOW       | 모두 허용 |
- 가능하면 웹 리소스를 HTTPS로 변경하는 것이 가장 좋은 해결 방법입니다.

---

## 4. Android 9 이상에서 HTTP 요청 차단 (Cleartext 정책)

- Android 9(API 28)부터는 HTTP 통신이 기본적으로 차단됩니다.
- 오류메시지: ERR_CLEARTEXT_NOT_PERMITTED
- 'http://example.com' 이 URL을 WebView에서 로드하면 차단됩니다.

---

## 5. HTTP 허용 방법 (Network Security Config)

### 📌network_security_config.xml 생성

```xml
<?xml version="1.0" encoding="utf-8"?>
<network-security-config>
    <base-config cleartextTrafficPermitted="true" />
</network-security-config>
```

### 📌AndroidManifest 설정

```xml
<application
        android:networkSecurityConfig="@xml/network_security_config"
        android:usesCleartextTraffic="true">
</application>
```

---

## 6. 특정 도메인만 HTTP 허용하기

보안을 위해 트겆ㅇ 도메인만 허용하는 것이 좋습니다.

```xml
<network-security-config>

    <domain-config cleartextTrafficPermitted="true">
        <domain includeSubdomains="true">
            example.com
        </domain>
    </domain-config>

</network-security-config>
```

---

## 7. SSL 인증서 오류 처리

- WebView에서 SSL 오류가 발생하면 다음 메서드가 호출됩니다.
- ```kotlin
  override fun onReceivedSslError(
    view: WebView?,
    handler: SslErrorHandler?,
    error: SslError?
  ) {
    handler?.cancel()
  }
  ```

---

## 8. HTTPS 리다이렉트 문제

- 웹 서버가 HTTP → HTTPS 리다이렉트를 잘못 설정하면 다음 문제가 발생합니다.
  - 무한 리다이렉트
  - 페이지 로딩 실패
- 확인 방법
  - bash: curl -I http://example.com
  - 응답: 301 → https://example.com