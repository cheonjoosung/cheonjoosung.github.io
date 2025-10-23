---
title: Android/안드로이드 View & Button 확장함수 모음
tags: [Kotlin]
style: fill
color: dark
description: Android/안드로이드 View & Button 확장함수 모음 – 디바운스/가시성/로딩/키보드/인셋 한 번에
---

## ✨ 개요

- 대상: `View`, `Button`(MaterialButton 포함)
- 목적: 중복 클릭 방지, 상태(활성/비활성)와 알파 동기화, 로딩 스피너, 인셋/라운드, 키보드 제어
- 의존: AndroidX KTX, (선택) `androidx.swiperefreshlayout:swiperefreshlayout` (로딩 스피너)

---

## 1 `ViewButton`

```kotlin
@file:Suppress("unused")
package com.example.ext

import android.content.Context
        import android.graphics.Rect
        import android.graphics.drawable.GradientDrawable
        import android.os.SystemClock
        import android.view.View
        import android.view.ViewGroup
        import android.view.inputmethod.InputMethodManager
        import android.widget.Button
        import androidx.annotation.ColorInt
        import androidx.annotation.Px
        import androidx.core.view.ViewCompat
        import androidx.core.view.WindowInsetsCompat
        import androidx.core.view.updateLayoutParams
        import androidx.core.view.updatePadding
        import androidx.swiperefreshlayout.widget.CircularProgressDrawable
        import com.google.android.material.button.MaterialButton

/* ----------------------------- 가시성/상태 ----------------------------- */

/** VISIBLE/GONE 토글 */
var View.isVisibleEx: Boolean
get() = visibility == View.VISIBLE
set(v) { visibility = if (v) View.VISIBLE else View.GONE }

fun View.show() { isVisibleEx = true }
fun View.gone() { isVisibleEx = false }
fun View.invisible() { visibility = View.INVISIBLE }

/** enabled 변경과 알파를 함께 (회색 처리) */
fun View.enableWithAlpha(enabled: Boolean, disabledAlpha: Float = 0.5f) {
    isEnabled = enabled
    alpha = if (enabled) 1f else disabledAlpha
}

/* ----------------------------- 클릭 디바운스/스로틀 ----------------------------- */

/** 중복 탭 방지(디바운스) – intervalMs 내 추가 탭 무시 */
fun View.setOnSafeClick(intervalMs: Long = 500L, block: (View) -> Unit) {
    var last = 0L
    setOnClickListener {
        val now = SystemClock.uptimeMillis()
        if (now - last >= intervalMs) { last = now; block(it) }
    }
}

/** 스로틀(첫 탭만 통과, 윈도우 내 추가 탭 무시) */
fun View.setOnThrottleClick(windowMs: Long = 800L, block: (View) -> Unit) {
    var allowAt = 0L
    setOnClickListener {
        val now = SystemClock.uptimeMillis()
        if (now >= allowAt) {
            allowAt = now + windowMs
            block(it)
        }
    }
}

/* ----------------------------- 키보드 ----------------------------- */

fun View.showKeyboard() {
    requestFocus()
    val imm = context.getSystemService(InputMethodManager::class.java)
    imm?.showSoftInput(this, 0)
}

fun View.hideKeyboard() {
    val imm = context.getSystemService(InputMethodManager::class.java)
    imm?.hideSoftInputFromWindow(windowToken, 0)
}

/* ----------------------------- 인셋/레이아웃 ----------------------------- */

/** 시스템 바 인셋을 패딩에 반영 */
fun View.applySystemBarInsets(
    left: Boolean = false, top: Boolean = true, right: Boolean = false, bottom: Boolean = true
) {
    ViewCompat.setOnApplyWindowInsetsListener(this) { v, insets ->
        val bars = insets.getInsets(WindowInsetsCompat.Type.systemBars())
        v.updatePadding(
            left   = if (left) bars.left else v.paddingLeft,
            top    = if (top) bars.top else v.paddingTop,
            right  = if (right) bars.right else v.paddingRight,
            bottom = if (bottom) bars.bottom else v.paddingBottom
        )
        WindowInsetsCompat.CONSUMED
    }
}

/** 마진 수정 헬퍼 */
fun View.updateMargins(@Px left: Int? = null, @Px top: Int? = null, @Px right: Int? = null, @Px bottom: Int? = null) =
    updateLayoutParams<ViewGroup.MarginLayoutParams> {
        setMargins(left ?: leftMargin, top ?: topMargin, right ?: rightMargin, bottom ?: bottomMargin)
    }

/** 라운드 배경 즉석 생성 */
fun View.roundBackground(@Px radius: Int, @ColorInt color: Int) {
    background = GradientDrawable().apply {
        cornerRadius = radius.toFloat()
        setColor(color)
    }
}

/** 화면에 실제로 보이는지 (겹침/크기 반영) */
fun View.isVisibleOnScreen(): Boolean {
    if (!isShown) return false
    val rect = Rect()
    return getGlobalVisibleRect(rect) && rect.height() > 0 && rect.width() > 0
}

/* ----------------------------- 버튼: 로딩/아이콘/눌림효과 ----------------------------- */

/**
 * Button/MaterialButton 공용 로딩 토글 (텍스트 옆에 인디케이터)
 * - CircularProgressDrawable 사용 (권장 의존: androidx.swiperefreshlayout)
 * - 토글 시 기존 텍스트/아이콘 유지
 */
private const val TAG_PROGRESS_DRAWABLE = 0x7F010001
private const val TAG_SAVED_TEXT = 0x7F010002

fun Button.setLoading(loading: Boolean, @ColorInt spinnerColor: Int? = null) {
    if (loading) {
        if (getTag(TAG_PROGRESS_DRAWABLE) != null) return
        setTag(TAG_SAVED_TEXT, text)
        enableWithAlpha(false)

        val spinner = CircularProgressDrawable(context).apply {
            strokeWidth = 3f
            centerRadius = 9f
            spinnerColor?.let { setColorSchemeColors(it) }
            start()
        }
        // TextView 계열: compound drawable로 표시 (start 위치)
        spinner.setBounds(0, 0, spinner.intrinsicWidth, spinner.intrinsicHeight)
        setCompoundDrawablesRelative(spinner, null, null, null)
        compoundDrawablePadding = (compoundDrawablePadding.takeIf { it > 0 } ?: 16)
        setTag(TAG_PROGRESS_DRAWABLE, spinner)
    } else {
        (getTag(TAG_PROGRESS_DRAWABLE) as? CircularProgressDrawable)?.stop()
        setCompoundDrawablesRelative(null, null, null, null)
        enableWithAlpha(true)
        (getTag(TAG_SAVED_TEXT) as? CharSequence)?.let { text = it }
        setTag(TAG_PROGRESS_DRAWABLE, null)
        setTag(TAG_SAVED_TEXT, null)
    }
}

/** MaterialButton: 아이콘 오른쪽/왼쪽 여백 간단 조절 */
fun MaterialButton.iconPaddingHorizontal(@Px padding: Int) {
    iconPadding = padding
    insetLeft = insetLeft // no-op but keeps clarity
    insetRight = insetRight
}

/** 눌림(scale) 애니메이션 – 짧고 기분 좋은 피드백 */
fun View.enablePressedScale(enabled: Boolean = true, scale: Float = 0.98f) {
    if (!enabled) {
        setOnTouchListener(null)
        scaleX = 1f; scaleY = 1f
        return
    }
    setOnTouchListener { v, ev ->
        when (ev.actionMasked) {
            android.view.MotionEvent.ACTION_DOWN -> { v.animate().scaleX(scale).scaleY(scale).setDuration(80).start() }
            android.view.MotionEvent.ACTION_UP,
            android.view.MotionEvent.ACTION_CANCEL -> { v.animate().scaleX(1f).scaleY(1f).setDuration(80).start() }
        }
        false
    }
}
```
- setLoading은 버튼 레이아웃을 건드리지 않고 컴파운드 드로어블로 스피너를 붙이는 방식이라 깔끔합니다.
- ProgressBar 부모에 붙였다 떼는 방식보다 안정적이며 회전/리사이즈에도 안전합니다.

---

## 2 사용 예

```kotlin
class SampleFragment : Fragment(R.layout.f_sample) {

    private lateinit var btnSubmit: Button
    private lateinit var root: View

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        btnSubmit = view.findViewById(R.id.btnSubmit)
        root = view.findViewById(R.id.root)

        // 1) 디바운스 + 눌림 스케일
        btnSubmit.setOnSafeClick { submit() }
        btnSubmit.enablePressedScale(true)

        // 2) 시스템 인셋 반영(상/하)
        root.applySystemBarInsets(top = true, bottom = true)

        // 3) 상태 + 알파 동기화
        btnSubmit.enableWithAlpha(false)
    }

    private fun submit() {
        // 입력 검증 후…
        btnSubmit.setLoading(true)
        btnSubmit.postDelayed({
            btnSubmit.setLoading(false)
            btnSubmit.snack("완료!") // 필요하면 View.snack 확장 추가해 쓰세요
        }, 1500)
    }
}
```

---

## 3 체크리스트 (실무 팁)

- 중복 클릭 방지: `setOnSafeClick()`를 버튼 기본으로 붙이세요.
- 버튼 상태 UI 통일: `enableWithAlpha()`로 비활성일 때 회색/반투명 처리.
- 로딩 중 클릭 막기: `setLoading(true)`가 isEnabled=false를 함께 적용.
- 인셋 안전 영역: `applySystemBarInsets()`로 Edge-to-Edge에서 겹침 방지.
- 키보드 UX: 포커스 줄 때 `showKeyboard()`, 보내고 `hideKeyboard()`.

---

## 스낵바 & dp 

```kotlin
// 간단 스낵바
fun View.snack(msg: CharSequence, duration: Int = com.google.android.material.snackbar.Snackbar.LENGTH_SHORT) =
    com.google.android.material.snackbar.Snackbar.make(this, msg, duration).show()

// dp → px
val Int.dp: Int get() = (this * resources.displayMetrics.density).toInt()
```