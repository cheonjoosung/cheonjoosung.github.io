---
title: (Android/안드로이드) 앱에서 플레이스토어 설치 페이지로 이동하는 방법
tags: [ Android,Kotlin ]
style: fill
color: dark
description: (Android/안드로이드) 앱에서 플레이스토어 설치 페이지로 이동하는 방법 – market:// vs https://play.google.com
---

## 개요

앱에서 아래와 같은 상황은 매우 흔합니다.
- 다른 앱 설치 유도
- 자사 앱 업데이트 안내
- 리뷰/평점 요청
- 앱 미설치 상태에서 스토어로 이동
- 이때 가장 표준적인 방법은 Intent로 Google Play Store 앱을 직접 여는 것입니다.

요약
- 기본: `market://details?id=패키지명`
- 예외 대응: Play Store 앱이 없으면 `https://play.google.com`로 fallback
- `ActivityNotFoundException` 반드시 처리
- 패키지명은 `BuildConfig.APPLICATION_ID` 활용 권장

---

## 1. 가장 기본적인 방법 (Play Store 앱으로 이동)

```kotlin
fun openPlayStore(context: Context, packageName: String) {
    val intent = Intent(
        Intent.ACTION_VIEW,
        Uri.parse("market://details?id=$packageName")
    )
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
    context.startActivity(intent)
}

openPlayStore(this, "com.example.myapp")
```

---

## 2. 실무에서 반드시 필요한 예외 처리 (권장)

- 일부 기기에서는 다음 경우가 발생할 수 있습니다.
  - Google Play Store 앱이 없음
  - 제조사 커스텀 OS
  - 에뮬레이터
  - 중국 단말
  - 이때는 웹 Play Store로 fallback 해야 합니다.

```kotlin
fun openPlayStoreSafe(context: Context, packageName: String) {
    try {
        val intent = Intent(
            Intent.ACTION_VIEW,
            Uri.parse("market://details?id=$packageName")
        )
        intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        context.startActivity(intent)
    } catch (e: ActivityNotFoundException) {
        // Play Store 앱이 없는 경우 웹으로 이동
        val webIntent = Intent(
            Intent.ACTION_VIEW,
            Uri.parse("https://play.google.com/store/apps/details?id=$packageName")
        )
        webIntent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
        context.startActivity(webIntent)
    }
}
```

---

## 3. 내 앱(자기 자신) 스토어 페이지로 이동하기

업데이트/리뷰 유도 시 가장 흔한 케이스

```kotlin
openPlayStoreSafe(
    context = this,
    packageName = BuildConfig.APPLICATION_ID
)
```

> BuildConfig.APPLICATION_ID 사용을 강력 추천 → build variant/패키지명 변경에도 안전

---

## 4. 플레이스토에서 "리뷰 페이지"로 바로 이동하기

```kotlin
fun openPlayStoreReview(context: Context) {
    val uri = Uri.parse(
        "market://details?id=${BuildConfig.APPLICATION_ID}&showAllReviews=true"
    )
    val intent = Intent(Intent.ACTION_VIEW, uri)
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
    context.startActivity(intent)
}
```

> 단, Google 정책/UI 변경으로 동작이 바뀔 수 있으므로 In-App Review API 사용이 더 권장되는 경우도 많음

---

## 5. `market://` vs `https://` 차이

| 구분    | market://     | [https://play.google.com](https://play.google.com) |
| ----- | ------------- | -------------------------------------------------- |
| 열리는 앱 | Play Store 앱  | 브라우저                                               |
| UX    | 빠르고 자연스러움     | 상대적으로 불편                                           |
| 의존성   | Play Store 필요 | 없음                                                 |
| 실무 사용 | **우선 사용**     | fallback                                           |

---

## 6. “설치 유도” UX에서 자주 쓰는 패턴

- 버튼 클릭 → Play Store 이동
- 다이얼로그 → 확인 시 이동
- 강제 업데이트 → 스토어 이동 후 앱 종료

```kotlin
AlertDialog.Builder(context)
    .setTitle("업데이트 필요")
    .setMessage("최신 버전을 설치해주세요.")
    .setPositiveButton("업데이트") { _, _ ->
        openPlayStoreSafe(context, BuildConfig.APPLICATION_ID)
    }
    .setCancelable(false)
    .show()
```