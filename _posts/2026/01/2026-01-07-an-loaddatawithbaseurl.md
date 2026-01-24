---
title: (Android/안드로이드) WebView에 HTML 템플릿 적용하기 (assets + loadDataWithBaseURL)
tags: [ Android,Kotlin ]
style: fill
color: dark
description: (Android/안드로이드) WebView에 HTML 템플릿 적용하기 – assets + loadDataWithBaseURL로 style/body 주입
---

## 개요

- 구현 내용
    - (1) 웹에서 \<style>...\</style> “태그 포함”으로 온다 → 그대로 \<head> 안에 삽입
    - (2) body는 \<div>...\</div> 같은 inner HTML만 온다 → \<body> 내부에 삽입
    - (3) assets의 test.html 템플릿을 읽어서 치환 후 loadDataWithBaseURL()로 로드

---

## 1. assets/test.html 템플릿 (토큰 방식 권장)

- app/src/main/assets/test.html

```html
<!doctype html>
<html lang="ko">
<head>
    <meta charset="utf-8"/>
    <meta name="viewport" content="width=device-width, initial-scale=1"/>
    <!-- {{STYLE_TAG}} -->
</head>
<body>
<!-- {{BODY_INNER}} -->
</body>
</html>
```

- {{STYLE_TAG}} : \<style>...\</style> 통째로 삽입
- {{BODY_INNER}} : \<div>...\</div> 같은 바디 내부 콘텐츠 삽입

---

## 2. 코틀린 템플릿 치환 + webView 로드

### 2.1 템플릿 렌더러

```kotlin
object HtmlTemplateRenderer {

    fun render(
        templateHtml: String,
        styleTagFromWeb: String?,   // "<style>...</style>" 통째로 온다고 가정
        bodyInnerFromWeb: String?   // "<div>...</div>" 만 온다고 가정
    ): String {
        val styleTag = styleTagFromWeb.orEmpty()
        val bodyInner = bodyInnerFromWeb.orEmpty()

        // (선택) 아주 최소한의 방어: styleTag에 <style>가 없으면 삽입 안 함
        val safeStyleTag = if (styleTag.trim().startsWith("<style", ignoreCase = true)) {
            styleTag
        } else {
            "" // 전제와 다르면 무시(또는 에러 처리)
        }

        return templateHtml
            .replace("{{STYLE_TAG}}", safeStyleTag)
            .replace("{{BODY_INNER}}", bodyInner)
    }
}
```

### 2.2 assets 읽기 & webView 표

```kotlin
fun loadWebHtmlIntoWebView(
    context: Context,
    webView: WebView,
    styleTagFromWeb: String,     // "<style>...</style>"
    bodyInnerFromWeb: String     // "<div>...</div>"
) {
    val template = AndroidUtils.readFromAssets(context, "test.html")

    val finalHtml = HtmlTemplateRenderer.render(
        templateHtml = template,
        styleTagFromWeb = styleTagFromWeb,
        bodyInnerFromWeb = bodyInnerFromWeb
    )시
    webView.settings.javaScriptEnabled = true // 필요 시만
    webView.settings.domStorageEnabled = true // 필요 시만

    // assets 내 리소스/상대경로가 있으면 이 baseUrl이 안정적
    val baseUrl = "file:///android_asset/"

    webView.loadDataWithBaseURL(
        baseUrl,
        finalHtml,
        "text/html",
        "utf-8",
        null
    )
}
```

---

## 3. 왜 loadDataWithBaseURL()을 쓰는가? (포스팅에 넣기 좋은 설명)

- loadData()도 HTML을 띄울 수는 있지만, 실무에서는 loadDataWithBaseURL()을 권장하는 케이스가 많습니다.
  - 템플릿에 이미지/CSS/폰트 등 상대경로 리소스가 있을 수 있음
  - 오리진(base URL)을 명시하면 리소스 로딩/보안정책이 더 예측 가능
  - file:///android_asset/를 주면 assets 기반 리소스 참조가 쉬움

---

## 4. 주의사항 (전제가 맞아도 실무에서 중요한 포인트)

- \<style>이 “태그 포함”으로 오면 중첩 주의
  - 이번 전제에서는 \<head>에 그대로 넣으므로 안전합니다.
  - (템플릿에 \<style>...\</style> 고정 블록을 만들고 그 안에 또 \<style>을 넣으면 깨짐)
- body가 \<div>만 온다면 그대로 삽입 OK
  - 단, 외부 입력이므로 HTML에 \<script>가 섞일 가능성이 있다면 보안 정책을 검토해야 합니다.
  - 특히 addJavascriptInterface를 쓰는 앱이라면 화이트리스트/필터링을 고려하세요.