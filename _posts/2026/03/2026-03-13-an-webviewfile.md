---
title: (Android/안드로이드) WebView에서 파일 업로드 구현 방법 - input type="file" 처리 가이드
tags: [ Android,Kotlin ]
style: fill
color: dark
description: (Android/안드로이드) WebView에서 input type=file 업로드를 구현하는 방법을 정리합니다. WebChromeClient onShowFileChooser 처리, 갤러리/카메라 선택, Activity Result API 적용, Android 13 권한 이슈까지 실무 기준으로 설명합니다.
---

## 개요

- WebView를 사용하다 보면 웹 페이지 안의 아래 같은 태그를 처리해야 할 때가 많습니다.
  - `<input type="file" />`
- 하지만 Android WebView는 이 태그를 자동으로 완벽하게 처리해주지 않습니다.
- 실제로는 WebChromeClient의 onShowFileChooser()를 구현해서 직접 파일 선택기를 열어줘야 합니다.
  - input type="file" 동작 원리
  - WebChromeClient.onShowFileChooser() 구현

---

## 1. WebView 파일 업로드는 어떻게 동작하나?

- ```html
  <input type="file" accept="image/*" />
  ```
- ```kotlin
  override fun onShowFileChooser(
    webView: WebView?,
    filePathCallback: ValueCallback<Array<Uri>>?,
    fileChooserParams: FileChooserParams?
  ): Boolean
  ```
  - input type="file" 액션이 발생하면, WebView는 내부적으로 파일 선택 UI가 필요하다고 판단합니다.
  - 이때 Android 쪽으로 콜백이 들어옵니다.
  - ```text
    HTML input[type=file]
        ↓
    WebView
        ↓
    WebChromeClient.onShowFileChooser()
        ↓
    Android 파일 선택 Intent 실행
        ↓
    선택 결과(Uri) 반환
        ↓
    웹 페이지 업로드 진행
    ```

---

## 2. 기본 구현 구조

- 핵심은 2가지입니다.
  - WebChromeClient에서 filePathCallback 보관
  - 파일 선택 결과를 다시 filePathCallback.onReceiveValue()로 전달

---

### 3. WebView 기본 세팅

- ```kotlin
  class WebViewActivity : AppCompatActivity() {

    private lateinit var webView: WebView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_webview)

        webView = findViewById(R.id.webView)

        webView.settings.apply {
            javaScriptEnabled = true
            domStorageEnabled = true
            allowFileAccess = true
            allowContentAccess = true
        }

        webView.webViewClient = WebViewClient()
        webView.webChromeClient = CustomWebChromeClient()

        webView.loadUrl("https://your-site.com")
    }
  }
  ```

---

## 4. 가장 기본적인 파일 업로드 구현

### 4.1 콜백 변수 선언

```kotlin
private var filePathCallback: ValueCallback<Array<Uri>>? = null
```

### 4.2 Activity Result API 등록

```kotlin
private val fileChooserLauncher =
    registerForActivityResult(ActivityResultContracts.StartActivityForResult()) { result ->
        val callback = filePathCallback ?: return@registerForActivityResult

        val uris = WebChromeClient.FileChooserParams.parseResult(
            result.resultCode,
            result.data
        )

        callback.onReceiveValue(uris)
        filePathCallback = null
    }
```

### 4.3 WebChromeClient 구현

```kotlin
inner class CustomWebChromeClient : WebChromeClient() {

    override fun onShowFileChooser(
        webView: WebView?,
        filePathCallback: ValueCallback<Array<Uri>>?,
        fileChooserParams: FileChooserParams?
    ): Boolean {
        this@WebViewActivity.filePathCallback?.onReceiveValue(null)
        this@WebViewActivity.filePathCallback = filePathCallback

        val intent = fileChooserParams?.createIntent()
        return try {
            fileChooserLauncher.launch(intent)
            true
        } catch (e: Exception) {
            this@WebViewActivity.filePathCallback = null
            false
        }
    }
}
```

---

## 5. Android 13 이상 권한 이슈

- 예전에는 READ_EXTERNAL_STORAGE를 많이 사용했지만,
- 지금은 `ACTION_GET_CONTENT` 기반이면 대부분 별도 저장소 권한 없이도 선택이 가능합니다.
  - 갤러리/파일 선택: ACTION_GET_CONTENT, OpenDocument 사용 시 보통 권한 불필요
  - 카메라 촬영: 카메라 앱 호출 자체는 가능하지만, 앱 내부에서 직접 카메라 권한을 쓸 경우는 별도 처리 필요
- 정리
  - 단순 파일 선택이면 저장소 권한을 먼저 의심하지 않아도 됨
  - 직접 파일 경로 접근 방식보다 Uri 중심으로 처리해야 안전

---

## 6. 자주 터지는 실수

- 이전 콜백 정리 안함
  - ```kotlin
    // 권장하지 않음
    filePathCallback = newCallback

    // 권장
    this.filePathCallback?.onReceiveValue(null)
    this.filePathCallback = newCallback
    ```
  - 이전에 살아 있던 callback을 정리하지 않으면 웹 쪽 상태가 꼬일 수 있습니다.
- 취소 시 callback 미호출
  - 사용자가 선택창을 닫았는데 onReceiveValue(null)를 안 부르면 다음 업로드가 안 열리는 경우가 있습니다.
- camera intent 결과를 data?.data만 보는 실수
  - 카메라 촬영은 EXTRA_OUTPUT 방식이면 data가 null일 수 있습니다.
- file path를 직접 뽑으려는 시도
  - 요즘은 실제 파일 경로를 못 얻는 경우가 많습니다.
  - Uri 기반 업로드를 전제로 가는 것이 맞습니다.
- Fragment에서 launcher 등록 위치 문제
  - Fragment라면 registerForActivityResult()는 적절한 생명주기 시점에 선언해야 합니다.
  - 보통 프로퍼티 초기화 시점에 등록하는 패턴이 안전합니다.
- Fragment 버전 예시에서 특히 주의할 점
  - WebView를 Fragment에서 쓴다면:
    - onDestroyView()에서 WebView 정리
    - filePathCallback 누수 방지
    - Activity 참조 오래 잡지 않기
    - ```kotlin
      override fun onDestroyView() {
        filePathCallback?.onReceiveValue(null)
        filePathCallback = null
        cameraImageUri = null

        webView.webChromeClient = null
        webView.webViewClient = null
        webView.destroy()

        super.onDestroyView()
      }
      ```
      
---

## 7. 실무에서 추천하는 구조

```text
WebViewActivity / WebViewFragment
        ↓
WebFileChooserManager
        ↓
- chooser intent 생성
- camera uri 생성
- 결과 파싱
- callback 전달
```

- 이렇게 분리하면:
  - WebChromeClient가 가벼워지고
  - 테스트가 쉬워지고
  - Activity/Fragment 재사용성이 좋아집니다.