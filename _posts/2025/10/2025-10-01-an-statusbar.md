---
title: Android 상태바/네비게이션바 숨기기(Immersive Mode)
tags: [Android]
style: fill
color: dark
description: Android 상태바/네비게이션바 숨기기(Immersive Mode) – WindowInsetsController / SYSTEM_UI_FLAG 한 줄 토글
---

## ✨ 개요


전체화면 사진/영상, 게임, 뷰어 앱에서 상태바(Status Bar)와 내비게이션 바(Navigation Bar)를 숨기는 가장 간단하고 안전한 방법을 정리했습니다.  
신규(안드 11+)는 `WindowInsetsController`, 구형은 `SYSTEM_UI_FLAG_*`로 한 줄 토글이 가능합니다.

> 핵심 요약
> - **신규 API (Android 11+, API 30+)**: `WindowInsetsController`
> - **레거시 (API 19~29)**: `View.SYSTEM_UI_FLAG_*` (Immersive/Sticky)
> - **Edge-to-Edge**: `WindowCompat.setDecorFitsSystemWindows(window, false)`

---

## 1. 최신 권장 방식: WindowInsetsController (API 30+)

```kotlin
fun Activity.toggleImmersive(isImmersive: Boolean) {
    WindowCompat.setDecorFitsSystemWindows(window, !isImmersive)
    val controller = WindowCompat.getInsetsController(window, window.decorView) ?: return
    controller.systemBarsBehavior =
        WindowInsetsController.BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE

    if (isImmersive) {
        controller.hide(WindowInsets.Type.systemBars())
    } else {
        controller.show(WindowInsets.Type.systemBars())
    }
}

// 전체화면 ON
toggleImmersive(true)
// 전체화면 OFF
toggleImmersive(false)
```
- `decorFitsSystemWindows(false)`로 컨텐츠를 시스템 바 아래까지 확장
- `BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE`로 스와이프로 일시 표시
- `systemBars()` = Status + Navigation


---

## 2. 포커스 변화 시 자동 복구(게임/비디오 앱에 유용)

```kotlin
override fun onWindowFocusChanged(hasFocus: Boolean) {
    super.onWindowFocusChanged(hasFocus)
    if (hasFocus) toggleImmersive(true)
}
```

---

## 3. 구형 기기 호환: SYSTEM_UI_FLAG_* (API 19~29)

```kotlin
@Suppress("DEPRECATION")
fun View.toggleImmersiveLegacy(isImmersive: Boolean) {
    if (isImmersive) {
        systemUiVisibility =
            View.SYSTEM_UI_FLAG_LAYOUT_STABLE or
                    View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION or
                    View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN or
                    View.SYSTEM_UI_FLAG_HIDE_NAVIGATION or
                    View.SYSTEM_UI_FLAG_FULLSCREEN or
                    View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY
    } else {
        systemUiVisibility = View.SYSTEM_UI_FLAG_VISIBLE
    }
}

window.decorView.toggleImmersiveLegacy(true)
```
- IMMERSIVE_STICKY를 쓰면 사용자가 화면 가장자리에서 스와이프해도 바가 잠깐 나타났다 사라집니다.

---

## 4. Edge-to-Edge 레이아웃 정리

```kotlin
ViewCompat.setOnApplyWindowInsetsListener(rootView) { v, insets ->
    val bars = insets.getInsets(WindowInsetsCompat.Type.systemBars())
    v.setPadding(bars.left, bars.top, bars.right, bars.bottom)
    WindowInsetsCompat.CONSUMED
}
```
- 전체화면에서 툴바/버튼이 바 아래로 겹치지 않게 인셋을 적용해야 합니다.
- 전체화면 유지 + 안전영역 보장(노치, 제스처 영역 포함)

---

## 5. 결론

- 입력창(IME)와 충돌: 키보드가 필요하면 WindowInsets.Type.ime()만 show/hide 제어.
- Dialog/Activity 전환 시 해제됨: 복원 로직을 onResume()/onWindowFocusChanged()에서 재적용.
- 제스처 내비게이션(버튼 無) 기기: BEHAVIOR_SHOW_TRANSIENT_BARS_BY_SWIPE로 UX 확보.
- 비디오 플레이어: 재생 화면 진입 시 ON, 컨트롤 노출/일시정지 시 OFF 패턴 권장.
- 테마 간섭: decorFitsSystemWindows(false) 사용 시 레이아웃 패딩으로 인셋 적용 필수.