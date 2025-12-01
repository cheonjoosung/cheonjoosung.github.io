---
title: (Android/안드로이드) WebView 스크롤 감지하기 (Scroll up & Scroll down) 
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) WebView 스크롤 감지하기 (Scroll up & Scroll down)
---

## ✨ 개요

WebView 내부 스크롤 이벤트를 활용해 하단 탭 숨기기 / 노출,
상단 툴바 애니메이션 처리 등을 구현하려면
스크롤 방향(Up or Down)을 정확하게 감지해야 한다.

이 글에서는 WebView의 Up/Down 스크롤 방향 감지 방법을 코드 중심으로 정리한다.

---

## 1. Scroll up/down 코드 & 중복호출 막기

### 1-1 스크롤 코드

```kotlin
webView.setOnScrollChangeListener { _, _, scrollY, _, oldScrollY ->
val dy = scrollY - oldScrollY

    if (dy > 0) {
        // 내려가는 중 → 숨김
        requestBottomVisibility(false)
    } else if (dy < 0) {
        // 올라가는 중 → 보임
        // 단, 아래 끝에서 튕긴 경우는 무시
        if (webView.canScrollVertically(1)) {
            requestBottomVisibility(true)
        }
    }
}
```

### 1-2 중복호출 Runnable 구현

```kotlin
private val handler = Handler(Looper.getMainLooper())
private var pendingRunnable: Runnable? = null

private fun requestBottomVisibility(visible: Boolean) {
    // 이미 목표 상태가 같으면 할 필요 없음
    if (desiredVisible == visible) return
    desiredVisible = visible

    // 기존 예약된 애니메이션 제거 → 중복 호출 방지
    pendingRunnable?.let { handler.removeCallbacks(it) }

    // 새로 예약
    pendingRunnable = Runnable {
        if (isVisible == desiredVisible) return@Runnable  // 애니메이션 중복 방지

        if (desiredVisible) {
            bottomTab.animate()
                .translationY(0f)
                .setDuration(200)
                .start()
        } else {
            bottomTab.animate()
                .translationY(bottomTab.height.toFloat())
                .setDuration(200)
                .start()
        }

        isVisible = desiredVisible
    }

    // 바로 실행해도 되고, 0~20ms 정도 딜레이를 줘도 안정적
    handler.post(pendingRunnable!!)
}
```

---

## 2. 정리

✔ show/hide 중복 호출 없음
- 스크롤할 때 수십 번 이벤트가 들어와도 애니메이션은 단 1번만 실행됨.

✔ 부드러운 UI
- 스크롤 중에는 “목표 상태(desiredVisible)”만 갱신하고 UI는 Runnable이 안정적으로 처리.

✔ 바닥에서 튕김에 의한 show 방지
- canScrollVertically(1) 체크로 해결.

✔ 현재 상태(isVisible)와 목표(desiredVisible)를 분리
- 튐/깜빡임 문제 완전 제거.