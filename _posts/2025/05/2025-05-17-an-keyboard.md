---
title: Android 키보드 자동 숨기기 및 보이기 처리 방법 (입력창 외 터치 시 키보드 내리기)
tags: [Android]
style: fill
color: dark
description: 입력창(EditText) 외 터치 시 소프트 키보드 숨기기 및 보이기 처리 방법을 소개합니다.
---

## ✨ 개요

안드로이드 앱에서 **EditText 외 영역 터치 시 키보드를 자동으로 숨기고**, 필요할 때는 **프로그래밍적으로 키보드를 보이게** 하고 싶을 때가 많습니다.

이 글에서는 `dispatchTouchEvent()`를 활용해 키보드를 숨기는 방법과, `InputMethodManager`를 이용한 키보드 표시 방법을 함께 다룹니다.

---

## 1. ✅ 전체 예제 코드

### 1.1 레이아웃 파일

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
                                                   xmlns:app="http://schemas.android.com/apk/res-auto"
                                                   xmlns:tools="http://schemas.android.com/tools"
                                                   android:id="@+id/main"
                                                   android:layout_width="match_parent"
                                                   android:layout_height="match_parent"
                                                   tools:context=".MainActivity">

  <EditText
          android:id="@+id/editText"
          android:layout_width="0dp"
          android:layout_height="wrap_content"
          android:hint="입력해주세요."
          app:layout_constraintEnd_toEndOf="parent"
          app:layout_constraintStart_toStartOf="parent"
          app:layout_constraintTop_toTopOf="parent" />

  <Button
          android:id="@+id/showKeyboardButton"
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"
          android:layout_marginTop="8dp"
          android:text="키보드 보이기"
          app:layout_constraintEnd_toEndOf="parent"
          app:layout_constraintStart_toStartOf="parent"
          app:layout_constraintTop_toBottomOf="@id/editText" />


</androidx.constraintlayout.widget.ConstraintLayout>
```
- EditText & Button

### 1.2  입력 창 외 터치 코드

```kotlin
import android.widget.EditText
import android.view.MotionEvent
import android.graphics.Rect

override fun dispatchTouchEvent(ev: MotionEvent): Boolean {
  if (ev.action == MotionEvent.ACTION_DOWN) {
    val v = currentFocus
    if (v is EditText) {
      val outRect = Rect()
      v.getGlobalVisibleRect(outRect)
      if (!outRect.contains(ev.rawX.toInt(), ev.rawY.toInt())) {
        v.clearFocus()
        hideKeyboard(v)
      }
    }
  }
  return super.dispatchTouchEvent(ev)
}
```

### 1.3 키보드 숨기기/보이기 코드

```kotlin
import android.content.Context
import android.view.View
import android.view.inputmethod.InputMethodManager

private fun hideKeyboard(view: View) {
  val imm = getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
  imm.hideSoftInputFromWindow(view.windowToken, 0)
}

private fun showKeyboard(view: View) {
  if (view.requestFocus()) {
    val imm = getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
    imm.showSoftInput(view, InputMethodManager.SHOW_IMPLICIT)
  }
}
```

---

## 2.🧠 **결론 & 보안 팁**

- 사용자 편의성을 높이기 위해 입력창 외 영역 터치 시 키보드 자동 숨김 처리는 필수적인 UX 개선 포인트입니다.
- 복잡한 코드 없이 dispatchTouchEvent만 오버라이드하면 대부분의 앱에서 간단히 구현할 수 있습니다.

> ✨ 보완 팁: Dialog, BottomSheetDialogFragment 등에서도 비슷한 방식으로 구현 가능하지만, 각기 다른 컨텍스트에서 window 객체 접근 방식은 조정이 필요할 수 있습니다.