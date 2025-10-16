---
title: Android WebChromeClient 완벽 가이드
tags: [Android]
style: fill
color: dark
description: Android WebChromeClient 완벽 가이드 – 파일 업로드, 진행률, JS 다이얼로그, 전체화면 영상까지 (Kotlin)
---

## ✨ 개요

`WebViewClient`는 **페이지 탐색 흐름과 로딩 수명주기**를 다루는 핵심 콜백 집합입니다.  
링크 열기 경로(내부/외부), 에러 화면, SSL 처리, 리디렉션, 리소스 가로채기까지 모두 `WebViewClient` 영역이죠.  
이 글에서는 **실무에서 바로 쓰는 기본 세팅 + 콜백별 베스트 프랙티스**를 정리합니다.
---

## 1 언제 `WebViewClient`를 쓰나?

WebChromeClient 완벽 가이드 – 파일 업로드, 진행률, JS 다이얼로그, 전체화면 영상까지 (Kotlin)

`WebChromeClient`는 WebView에서 브라우저 UI 성격의 기능을 처리하는 클라이언트입니다.  
**진행률(Progress), 페이지 제목/아이콘, 파일 업로드(input type="file"), JS 다이얼로그(alert/confirm/prompt), 권한(Camera/Mic/Geolocation), 콘솔 로그, 전체화면 동영상** 등을 담당합니다.

> 비교: 네비게이션/에러/SSL/리소스는 `WebViewClient`, 브라우저 UI는 `WebChromeClient`.

---

## 2 기본 세팅 템플릿

```kotlin
@SuppressLint("SetJavaScriptEnabled")
fun WebView.setupChrome(
    onProgress: (Int) -> Unit = {},
    onTitle: (String?) -> Unit = {},
    pickFiles: (mime: String, allowMultiple: Boolean, cb: (Array<Uri>) -> Unit) -> Unit,
    enterFullscreen: (view: View, cb: WebChromeClient.CustomViewCallback) -> Unit,
    exitFullscreen: () -> Unit
) = apply {
        settings.javaScriptEnabled = true
        settings.domStorageEnabled = true

        webChromeClient = object : WebChromeClient() {

            // 1) 진행률 0..100
            override fun onProgressChanged(view: WebView?, newProgress: Int) {
                onProgress(newProgress)
            }

            // 2) 페이지 제목/아이콘
            override fun onReceivedTitle(view: WebView?, title: String?) {
                onTitle(title)
            }

            // 3) 파일 업로드 (input type="file")
            override fun onShowFileChooser(
                webView: WebView,
                filePathCallback: ValueCallback<Array<Uri>>,
                fileChooserParams: FileChooserParams
            ): Boolean {
                val accept = fileChooserParams.acceptTypes.firstOrNull()?.ifBlank { "*/*" } ?: "*/*"
                val multiple = fileChooserParams.mode == FileChooserParams.MODE_OPEN_MULTIPLE
                // 앱 쪽(Activity Result API) 파일 선택기로 위임
                pickFiles(accept, multiple) { uris ->
                    filePathCallback.onReceiveValue(uris)
                }
                return true
            }

            // 4) JS 다이얼로그 (alert/confirm/prompt)
            override fun onJsAlert(view: WebView?, url: String?, message: String?, result: JsResult): Boolean {
                val ctx = view?.context ?: return false
                AlertDialog.Builder(ctx)
                    .setMessage(message)
                    .setPositiveButton(android.R.string.ok) { _, _ -> result.confirm() }
                    .setOnCancelListener { result.cancel() }
                    .show()
                return true
            }
            override fun onJsConfirm(view: WebView?, url: String?, message: String?, result: JsResult): Boolean {
                val ctx = view?.context ?: return false
                AlertDialog.Builder(ctx)
                    .setMessage(message)
                    .setPositiveButton(android.R.string.ok) { _, _ -> result.confirm() }
                    .setNegativeButton(android.R.string.cancel) { _, _ -> result.cancel() }
                    .show()
                return true
            }
            override fun onJsPrompt(
                view: WebView?, url: String?, message: String?, defaultValue: String?, result: JsPromptResult
            ): Boolean {
                val ctx = view?.context ?: return false
                val input = EditText(ctx).apply { setText(defaultValue ?: "") }
                AlertDialog.Builder(ctx)
                    .setMessage(message)
                    .setView(input)
                    .setPositiveButton(android.R.string.ok) { _, _ -> result.confirm(input.text.toString()) }
                    .setNegativeButton(android.R.string.cancel) { _, _ -> result.cancel() }
                    .show()
                return true
            }

            // 5) 콘솔 로그 (개발 편의)
            override fun onConsoleMessage(consoleMessage: ConsoleMessage): Boolean {
                Log.d("WV-CONSOLE", "[${consoleMessage.messageLevel()}] ${consoleMessage.message()} @${consoleMessage.sourceId()}:${consoleMessage.lineNumber()}")
                return super.onConsoleMessage(consoleMessage)
            }

            // 6) 전체화면 영상(HTML5 video, <iframe> 등)
            override fun onShowCustomView(view: View, callback: CustomViewCallback) {
                enterFullscreen(view, callback)
            }
            override fun onHideCustomView() {
                exitFullscreen()
            }

            // 7) 카메라/마이크 권한 (WebRTC 등)
            override fun onPermissionRequest(request: PermissionRequest) {
                // 요청한 리소스
                val wants = request.resources // ex) VIDEO_CAPTURE, AUDIO_CAPTURE
                // 1) Android 런타임 권한 확인/요청 (CAMERA/RECORD_AUDIO)
                // 2) 승인 후에만 request.grant(wants)
                // 3) 거부 시 request.deny()
            }

            // 8) 위치 권한(지오로케이션)
            override fun onGeolocationPermissionsShowPrompt(origin: String?, callback: GeolocationPermissions.Callback) {
                // 1) ACCESS_FINE_LOCATION 런타임 권한 체크/요청
                // 2) 허용 시 callback.invoke(origin, true, false), 거부 시 false
            }
        }
    }
```
- 만약 target="_blank"나 새 창 요구가 잦다면, 그대로 현재 WebView로 로드하거나 별도의 탭 컨트롤을 직접 구현합니다.

---

## 3 보안/권한 체크리스트

- HTTPS + 신뢰 도메인만 로드(화이트리스트 권장)
- JS 다이얼로그 남용 방지: 앱 내 민감 플로우와 충돌하지 않게 설계
- 파일 업로드/권한: 실제 Android 런타임 권한 흐름과 동기화하여 grant()/deny()
- Mixed Content 차단 & Safe Browsing
- 캡처/녹화 방지 화면이 필요하면 Activity에 FLAG_SECURE 옵션 제공

```kotlin
WebSettingsCompat.setSafeBrowsingEnabled(webView.settings, true)
webView.settings.mixedContentMode = WebSettings.MIXED_CONTENT_NEVER_ALLOW

```

---

## 4 생명주기 & 메모리 주의

- 전체화면 진입 후 Activity 전환/종료 시 반드시 onHideCustomView() 호출로 복구
- Fragment에서는 onDestroyView()에서 WebView를 removeView → destroy()
- 파일 선택 콜백 보관 시 회전/재생성에 유의(상태 저장 또는 콜백 클리어)
    
---

## 5. 결론

- WebChromeClient는 브라우저 UI 기능 전반(진행률/제목/파일 업로드/JS 다이얼로그/권한/전체화면)을 담당.
- 파일 업로드는 Activity Result API로 깔끔하게, 전체화면 영상은 오버레이 컨테이너 + 시스템 바 토글 패턴으로 처리.
- 보안(권한, HTTPS, Safe Browsing)을 기본값으로 두고, WebViewClient와 역할 분리하면 유지보수가 쉬워집니다.