---
title: 안드로이드 텍스트뷰 programmatically - 스타일 변경, 밑줄, 취소선, 배경 색, 텍스트 크기 등 동적 제어
tags: [Android]
style: fill
color: dark
description: Android TextView Programmatically - 스타일 변경, 밑줄, 취소선, 배경 복원, 텍스트 크기 동적 조절 등 실무에서 필요한 내용 중심.
---

## ✨ 개요

`TextView`는 Android에서 가장 많이 사용되는 UI 컴포넌트 중 하나입니다.  
하지만 단순한 텍스트 출력 외에도 다양한 **스타일 적용**, **동적 제어**, **이벤트 처리**까지 폭넓은 기능을 가지고 있습니다.

이 포스트에서는 **텍스트 크기 조절, 밑줄/취소선/볼드 처리, 배경색 변경 및 복원** 등 실무에 꼭 필요한 `TextView` 활용법을 정리합니다.

---

## 1. ✅ TextView Programmatically 기능

- 취소선, 밑줄, 볼드, 기울임, 색상 변경, 배경색 변경, 글자 크기 변경

---

## 2. ✅ TextView Programmatically 코드

### 2.1 레이아웃 파일

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
                                                   xmlns:app="http://schemas.android.com/apk/res-auto"
                                                   xmlns:tools="http://schemas.android.com/tools"
                                                   android:id="@+id/main"
                                                   android:layout_width="match_parent"
                                                   android:layout_height="match_parent"
                                                   tools:context=".MainActivity">

    <TextView
            android:id="@+id/textView"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_marginTop="16dp"
            android:text="이것은 텍스트입니다"
            android:textSize="24sp"
            app:layout_constraintEnd_toEndOf="parent"
            app:layout_constraintStart_toStartOf="parent"
            app:layout_constraintTop_toTopOf="parent" />

    <LinearLayout
            android:layout_width="match_parent"
            android:layout_height="0dp"
            android:gravity="center_horizontal"
            android:orientation="vertical"
            app:layout_constraintBottom_toBottomOf="parent"
            app:layout_constraintTop_toBottomOf="@id/textView">

        <Button
                android:id="@+id/cancelLine"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginTop="16dp"
                android:text="취소선"
                android:textSize="20sp" />

        <Button
                android:id="@+id/underline"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginTop="8dp"
                android:text="밑줄"
                android:textSize="20sp" />

        <Button
                android:id="@+id/bold"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginTop="8dp"
                android:text="볼드"
                android:textSize="20sp" />

        <Button
                android:id="@+id/tilt"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginTop="8dp"
                android:text="기울임"
                android:textSize="20sp" />

        <Button
                android:id="@+id/color"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginTop="8dp"
                android:text="색상 변경"
                android:textSize="20sp" />

        <Button
                android:id="@+id/background"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginTop="8dp"
                android:text="배경색 변경"
                android:textSize="20sp" />

        <Button
                android:id="@+id/sizeUp"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginTop="8dp"
                android:text="글짜 크기 업"
                android:textSize="20sp" />

        <Button
                android:id="@+id/sizeDown"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginTop="8dp"
                android:text="글짜 크기 다운"
                android:textSize="20sp" />

        <Button
                android:id="@+id/reset"
                android:layout_width="wrap_content"
                android:layout_height="wrap_content"
                android:layout_marginTop="8dp"
                android:text="초기화"
                android:textSize="20sp" />

    </LinearLayout>

</androidx.constraintlayout.widget.ConstraintLayout>
```
- 버튼에 따라 텍스트 스타일 변경

### 2.2  커스텀 확장 함수로 깔끔하게 관리하기

```kotlin
import android.graphics.Paint
import android.widget.TextView

fun TextView.setStrikeThrough(enable: Boolean) {
    paintFlags = if (enable)
        paintFlags or Paint.STRIKE_THRU_TEXT_FLAG
    else
        paintFlags and Paint.STRIKE_THRU_TEXT_FLAG.inv()
}

fun TextView.setUnderline(enable: Boolean) {
    paintFlags = if (enable)
        paintFlags or Paint.UNDERLINE_TEXT_FLAG
    else
        paintFlags and Paint.UNDERLINE_TEXT_FLAG.inv()
}
```

### 2.3 안드로이드 설치된 앱 목록 조회 코드

```kotlin
import android.graphics.Color
import android.graphics.Paint
import android.graphics.Typeface
import android.os.Bundle
import android.util.TypedValue
import androidx.appcompat.app.AppCompatActivity
import com.example.sample.databinding.ActivityMainBinding

class MainActivity : AppCompatActivity() {

    private lateinit var binding: ActivityMainBinding

    private val TAG: String = MainActivity::class.java.simpleName

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        val originSize= binding.textView.textSize / binding.textView.resources.displayMetrics.scaledDensity

        with(binding) {
            cancelLine.setOnClickListener {
                textView.paintFlags = textView.paintFlags or Paint.STRIKE_THRU_TEXT_FLAG
            }

            underline.setOnClickListener {
                textView.paintFlags = textView.paintFlags or Paint.UNDERLINE_TEXT_FLAG
            }

            bold.setOnClickListener {
                textView.setTypeface(null, Typeface.BOLD) // NORMAL, BOLD, ITALIC, BOLD_ITALIC
            }

            tilt.setOnClickListener {
                textView.setTypeface(null, Typeface.ITALIC) // NORMAL, BOLD, ITALIC, BOLD_ITALIC
            }

            color.setOnClickListener {
                textView.setTextColor(Color.RED)
            }

            background.setOnClickListener {
                textView.setBackgroundColor(Color.BLUE)
            }

            sizeUp.setOnClickListener {
                val currentSpSize = textView.textSize / textView.resources.displayMetrics.scaledDensity
                textView.setTextSize(TypedValue.COMPLEX_UNIT_SP, currentSpSize + 1)
            }

            sizeDown.setOnClickListener {
                val currentSpSize = textView.textSize / textView.resources.displayMetrics.scaledDensity
                textView.setTextSize(TypedValue.COMPLEX_UNIT_SP, currentSpSize - 1)
            }

            reset.setOnClickListener {
                textView.paintFlags = 0
                textView.setTypeface(Typeface.DEFAULT)
                textView.setTextColor(Color.WHITE)
                textView.setBackgroundColor(Color.TRANSPARENT)
                textView.textSize = originSize
            }
        }
    }
}
```

---

## 3.🧠 **결론**

- TextView는 단순히 문자열을 출력하는 뷰를 넘어서, 사용자와의 상호작용을 강화하는 매우 강력한 UI 도구입니다.
    - 스타일링 (취소선, 밑줄, 볼드)
    - 동적 크기 조절
    - 배경색 처리
- 이런 기능을 잘 활용하면 동적으로 필요한 부분을 변경하는 UI/UX 구현이 가능합니다