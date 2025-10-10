---
title: Android 최상단에서만 SwipeRefresh 동작시키기 – RecyclerView/NestedScrollView/WebView 한 번에 정리
tags: [Android]
style: fill
color: dark
description: Android 최상단에서만 SwipeRefresh 동작시키기 – RecyclerView/NestedScrollView/WebView 한 번에 정리
---

## ✨ 개요

당겨서 새로고침(Swipe to Refresh)은 자연스러운 UX지만, **스크롤 중에도 오작동**하면 좋지 않습니다.  
이 글에서는 **“스크롤이 최상단일 때만 새로고침”**이 동작하도록 만드는 실전 패턴을 뷰 타입별로 정리합니다.

> 핵심: **SwipeRefreshLayout의 canChildScrollUp**을 올바르게 판단시키면 됨  
> 최신 API: `setOnChildScrollUpCallback { … }`  
> 레거시: `SwipeRefreshLayout` 상속 → `canChildScrollUp()` 오버라이ㄷㅡ

---

## 1. 최신 권장: setOnChildScrollUpCallback (AndroidX SwipeRefreshLayout 1.1+)

`SwipeRefreshLayout`이 **자식이 위로 더 스크롤 가능한지**를 직접 물어봅니다.  
콜백에서 **최상단 여부**를 판단해 `true/false`를 반환하세요.

#### 1-1. RecyclerView (가장 흔한 케이스)
```kotlin
binding.swipeRefresh.setOnChildScrollUpCallback { _, child ->
    val rv = child as? RecyclerView ?: return@setOnChildScrollUpCallback child.canScrollVertically(-1)
    // 첫 아이템이 0이고, 맨 위로 더 스크롤할 수 없으면 = 최상단
    rv.canScrollVertically(-1) // true면 '아직 위로 더 스크롤 가능' → 새로고침 막음
}
```
- canScrollVertically(-1) == false → 이미 최상단 → 새로고침 허용
- true → 아직 위로 스크롤 여지 있음 → 새로고침 동작 차단

#### 1-2. NestedScrollView

```kotlin
binding.swipeRefresh.setOnChildScrollUpCallback { _, child ->
    val nsv = child as? NestedScrollView ?: return@setOnChildScrollUpCallback child.canScrollVertically(-1)
    nsv.canScrollVertically(-1) // 최상단이면 false → 새로고침 허용
}
```

#### 1-3. WebView

```kotlin
binding.swipeRefresh.setOnChildScrollUpCallback { _, child ->
    val web = child as? WebView ?: return@setOnChildScrollUpCallback child.canScrollVertically(-1)
    web.canScrollVertically(-1)
}
```
- 팁: child가 정확히 최상단 스크롤 뷰가 아닐 수 있으면, findViewById로 스크롤 가능한 실제 자식을 찾아 검사하세요.

---

## 2. 레거시 호환: 커스텀 SwipeRefreshLayout

```kotlin
class TopOnlySwipeRefresh @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null
) : SwipeRefreshLayout(context, attrs) {

    var scrollableChild: View? = null

    override fun canChildScrollUp(): Boolean {
        val v = scrollableChild ?: getChildAt(0)
        return v?.canScrollVertically(-1) == true
    }
}

val swipe = findViewById<TopOnlySwipeRefresh>(R.id.swipe)
swipe.scrollableChild = binding.recyclerView // 또는 NestedScrollView/WebView
```
- `setOnChildScrollUpCallback`이 없다면 상속해서 `canChildScrollUp()`을 오버라이드합니다.

---

## 3. AppBarLayout/CollapsingToolbar와 함께 쓰는 경우

툴바가 **아직 접히는 중(스크롤 shrink)**이라면 새로고침이 끼어들 수 있습니다.
AppBarLayout의 오프셋을 같이 체크해 완전히 펼쳐졌을 때만 허용하세요.

```kotlin
binding.swipeRefresh.setOnChildScrollUpCallback { _, child ->
    val atTop = !child.canScrollVertically(-1)
    val allow = appBarExpanded && atTop
    !allow.not() // allow가 true면 canChildScrollUp == false가 되어 새로고침 허용
}
```

---

## 4. 결론

- 빠르게 스크롤 중 오작동: canScrollVertically(-1) 체크는 프레임 단위로 즉시 반영됨.
- 혹시 미세 떨림이 있다면 NestedScrollingEnabled/overScrollMode 조합을 확인.
- 여러 스크롤 자식: 실제 스크롤 주체(RecyclerView 등)를 명시적으로 찾아 검사.
- 헤더/배너가 있는 리스트: 첫 아이템이 헤더라면, 헤더의 높이가 완전히 보였는지도 함께 판단 필요.
- 웹뷰 탄성 스크롤: 일부 기기에서 최상단 부근 탄성 효과로 `canScrollVertically(-1)`이 간헐적으로 `true`
  + 상단 여백을 없애거나 `scrollY == 0` 보조 체크를 추가.