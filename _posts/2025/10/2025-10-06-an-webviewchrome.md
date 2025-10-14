---
title: Android WebViewClient vs WebChromeClient 차이
tags: [Android]
style: fill
color: dark
description: Android WebViewClient vs WebChromeClient 차이와 올바른 사용 가이드 (Kotlin 예제)
---

## ✨ 개요

안드로이드에서 WebView를 제대로 쓰려면 **`WebViewClient`**와 **`WebChromeClient`**의 역할을 정확히 구분해야 합니다.  
둘을 섞어 쓰면 “링크 전환/에러/SSL/내부 네비게이션”과 “프로그레스/파일 업로드/JS 다이얼로그/퍼미션/전체화면 영상”을 깔끔하게 나눌 수 있습니다.


---

## 1 한 줄 정의

| 구분 | WebViewClient | WebChromeClient |
|---|---|---|
| 목적 | **네비게이션 & 로딩 수명주기** 제어 | **브라우저 UI & 고급 기능** 제어 |
| 대표 콜백 | `shouldOverrideUrlLoading`, `onPageStarted/Finished`, `onReceivedError`, `onReceivedSslError`, `shouldInterceptRequest` | `onProgressChanged`, `onShowFileChooser`, `onJsAlert/Confirm/Prompt`, `onConsoleMessage`, `onPermissionRequest`, `onReceivedTitle/Icon`, `onShowCustomView/onHideCustomView` |
| 언제 쓰나 | 내부/외부 링크 분기, 에러화면, SSL 대응, 헤더 주입/리소스 가로채기 | 진행바, 파일 업로드, JS 팝업, 카메라/마이크/지오로케이션 권한, 전체화면 동영상 |

> **핵심**: *페이지 이동/로드는 `WebViewClient`, 브라우저스러운 UI는 `WebChromeClient`.*

---

## 2 올바른 사용” 체크리스트

### WebViewClient 쪽
- 링크 분기는 shouldOverrideUrlLoading(request 기반, API 24+) 사용
  + 전화/메일/딥링크/외부 도메인은 외부 앱으로 넘기고 true 반환
  + 내부 도메인은 false → WebView가 로딩
- 에러 처리는 request.isForMainFrame 기준으로 메인 프레임만 대체 화면 표시
- SSL 에러는 proceed 금지: 보안상 차단하고 사용자 안내
- 리소스 가로채기(shouldInterceptRequest)는 최소화: 토큰 헤더 주입·오프라인 캐시 등 꼭 필요한 경우만. 성능/보안 주의

### WebChromeClient 쪽
- 진행률/제목/아이콘 → 툴바/ProgressBar와 연동
- 파일 업로드 → onShowFileChooser에서 Activity Result API로 처리, READ_MEDIA_*/카메라 권한 분기
- JS 다이얼로그 → 네이티브 다이얼로그로 감싸고 confirm()/cancel() 호출 잊지 않기
- 미디어 권한(onPermissionRequest) → 실제 런타임 권한과 동기화 후 grant()/deny()
- 전체화면 영상 → onShowCustomView/onHideCustomView에서 컨테이너로 이동 + Immersive 모드 토글

---

## 3 보안/안정성 팁

- 신뢰 도메인만 로드: 화이트리스트 + 강제 HTTPS 권장
- JavaScript는 필요한 화면에서만 true, 브릿지는 최소 범위만 @JavascriptInterface
- Safe Browsing / Mixed Content
- 데이터 정리: 로그아웃/개인정보 메뉴에서 CookieManager, 캐시, WebStorage 정리

```kotlin
WebView.setWebContentsDebuggingEnabled(BuildConfig.DEBUG)
WebSettingsCompat.setSafeBrowsingEnabled(webView.settings, true)
webView.settings.mixedContentMode = WebSettings.MIXED_CONTENT_NEVER_ALLOW
```

---

## 4 최소예제

```kotlin
class MyActivity : AppCompatActivity() {
    private lateinit var webView: WebView
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        webView = WebView(this)
        setContentView(webView)
        webView.setupForApp { progress -> /* progress bar */ }
        webView.loadUrl("https://www.my-domain.com")
    }
}
```

---

## 5 결론

- 페이지 이동/에러/SSL/네비 제어는 WebViewClient.
- 프로그레스/파일 업로드/JS 팝업/권한/전체화면은 WebChromeClient.
- 두 클라이언트를 역할대로 분리하면 디버깅이 쉬워지고, 보안·UX 품질이 크게 올라갑니다.