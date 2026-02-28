---
title: (Android/안드로이드) WebView 터치 이벤트가 먹통이 되는 원인과 해결 방법 총정리
tags: [ Android,Kotlin ]
style: fill
color: dark
description: (Android/안드로이드) WebView에서 스크롤/클릭이 먹통이 되는 원인을 setOnTouchListener, 부모 intercept, NestedScroll, 단말 설정 이슈까지 실무 사례 중심으로 정리
---

## 개요

- WebView를 사용하다 보면 다음과 같은 현상을 겪는 경우가 있습니다.
  - 스크롤이 안 된다
  - 클릭이 안 먹는다
  - 특정 단말에서만 터치가 무시된다
  - 마치 `setOnTouchListener { _, _ -> true }`를 설정한 것처럼 동작한다

---

## 1. 구조

- ```text
  Activity
     ↓
  ViewGroup.dispatchTouchEvent()
     ↓
  onInterceptTouchEvent()
     ↓
  Child.dispatchTouchEvent()
     ↓
  onTouchEvent()
  ```
- 👉 부모가 intercept하거나, onTouchListener에서 소비하면 자식은 이벤트를 못 받습니다.

---

## 2. 가장 흔한 원인 ① setOnTouchListener 반환값

```kotlin
// ❌ 잘못된 코드
webView.setOnTouchListener { _, _ ->
    true
}

// ✅ 올바른 방식
webView.setOnTouchListener { _, _ ->
    false
}
```

- true 반환 = 이벤트 소비
- WebView 내부 onTouchEvent 호출 안 됨
- 결과: 스크롤/클릭 모두 먹통

---

### 3. 원인 ② 부모 View가 intercept 하는 경우

```text
SwipeRefreshLayout
  └── NestedScrollView
        └── WebView
```

- 부모가 onInterceptTouchEvent에서 가로/세로 스크롤을 가로채면 WebView는 이벤트를 못 받습니다.

```kotlin
webView.setOnTouchListener { v, event ->
    v.parent.requestDisallowInterceptTouchEvent(true)
    false
}
```

---

## 4. 원인 ③ NestedScroll 충돌

- WebView는 내부적으로 자체 스크롤을 가집니다.
- 다음 구조에서 충돌이 잘 발생합니다:

```text
RecyclerView
   └── WebView
   
ViewPager2
   └── Fragment
        └── WebView
```

```kotlin
webView.isNestedScrollingEnabled = false
recyclerView.isNestedScrollingEnabled = false
```

---

## 5. 원인 ④ clickable / focusable 설정 문제

- XML에서 다음이 설정되어 있으면 문제가 발생할 수 있습니다.
  - android:clickable="false"
  - android:focusable="false"

```kotlin
webView.isFocusable = true
webView.isFocusableInTouchMode = true
```

---

## 6. 원인 ⑤ hardware acceleration 문제 (특정 단말)

- 저사양 단말 또는 특정 GPU에서 터치 인식은 되지만 화면 갱신이 안 되어 먹통처럼 보이는 경우

```kotlin
// android:hardwareAccelerated="true"
webView.setLayerType(View.LAYER_TYPE_HARDWARE, null)
```

---

## 7. 원인 ⑥ JS에서 preventDefault()

- 웹 코드 : event.preventDefault(); 설정시
    - 터치 이벤트가 막힐 수 있음
    - 스크롤/클릭 무시
    - 👉 앱 문제가 아니라 웹 코드 문제일 수도 있음

---

## 8. 원인 ⑦ Transparent View overlay

```text
FrameLayout
   ├── WebView
   └── View (alpha=0) <- 이 View가 터치를 먹고 있을 수 있습니다.
```

- 디버깅 팁

```kotlin
view.setOnTouchListener { _, _ ->
    Log.d("Touch", "overlay touched")
    false
}
```