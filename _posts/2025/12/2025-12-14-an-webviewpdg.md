---
title: (Android/안드로이드) WebView PDF 문서로 저장하기
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) WebView PDF 문서로 저장하기 - 스크롤이 긴 웹 문서를 한 장으로 저장
---

## ✨ 개요

WebView를 쓰다 보면 아래 요구사항이 자주 나옵니다.

- 현재 WebView 화면을 이미지로 저장(공유/리포트)
- 영수증/결제 내역 같은 전체 페이지를 캡처(PDF/이미지)
- 스크롤이 긴 웹 문서를 한 장으로 저장

이럴때 가장 간편하게 해결 가능한 Print(PDF) 저장 방법입니다

---

## 1.  코드

“전체 문서 보존” 목적이라면 이미지보다 PDF가 훨씬 안정적이고 품질도 좋습니다.

```kotlin
fun saveWebViewAsPdf(activity: Activity, webView: WebView, jobName: String = "WebViewDoc") {
    val printManager = activity.getSystemService(Context.PRINT_SERVICE) as PrintManager
    val printAdapter = webView.createPrintDocumentAdapter(jobName)

    val printAttributes = PrintAttributes.Builder()
        .setMediaSize(PrintAttributes.MediaSize.ISO_A4)
        .setMinMargins(PrintAttributes.Margins.NO_MARGINS)
        .build()

    val result = printManager.print(jobName, printAdapter, printAttributes)
    Toast.makeText(requireContext(), "result=$result", Toast.LENGTH_SHORT).show()
}
```

---

## 2. 캡처와 PDF 사용시점

| 목표                   | 추천 방식                                  |
| -------------------- | -------------------------------------- |
| 지금 보이는 화면만 저장/공유     | `webView.draw()` 캡처                    |
| 페이지 전체를 “이미지”로 꼭 저장  | 타일(분할) 캡처 후 여러 장 저장 권장                 |
| 영수증/문서 보관 목적(전체 페이지) | **PDF 출력(PrintDocumentAdapter)** 강력 추천 |
| 빠르게 전체를 시도해보고 싶음     | `capturePicture()`(권장 X, 참고용)          |

---

## 3. 캡처 시 자주 터지는 문제와 해결

- 흰 화면(빈 Bitmap) 캡처
  - 페이지 로딩 완료 전 캡처하면 발생 가능
  - onPageFinished 이후 또는 적절한 딜레이 후 캡처
- “WebView method must be called on the same thread”
  - 백그라운드에서 캡처/scrollTo/loadUrl 호출
  - 항상 webView.post {} 또는 Dispatchers.Main에서 실행
- OOM(OutOfMemory)
  - 전체 페이지를 한 장 Bitmap으로 만들 때 흔함
  - 타일 저장 / 다운스케일 / PDF로 전환