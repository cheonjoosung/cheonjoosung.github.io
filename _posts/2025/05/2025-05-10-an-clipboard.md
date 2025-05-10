---
title: Android ClipboardManager 클립보드 복사하여 붙여넣기 + 계좌번호 감지 이체 처리
tags: [Android]
style: fill
color: dark
description: Android ClipboardManager 클립보드 복사하여 붙여넣기 + 계좌번호 감지 이체 처리
---

## ✨ 개요

Android에서는 ClipboardManager를 통해 사용자가 복사한 텍스트를 감지하고 앱 내부에서 활용할 수 있습니다.

특히 다른 앱에서 복사한 계좌번호, 인증번호, 쿠폰 코드와 같은 텍스트를 감지해 사용자에게 스낵바 안내나 즉시 실행 버튼을 제공하는 방식은 실무에서 매우 유용하게 쓰입니다.

이 글에서는 클립보드 텍스트를 감지하고 조건에 따라 동작하는 방법을 실전 예제와 함께 소개합니다.

---

## 1. ✅ 주요 기능

- 텍스트 복사 및 붙여넣기 처리
- 클립보드 감지 후 계좌번호 형식이면 스낵바로 안내 및 "이체하기" 버튼 표시
- 계좌번호 외 다른 실무 사용 예도 사용 가능

---

## 2. ✅ 클립보드 처리 코드 예제

### 2.1 계좌번호 감지 및 Snackbar 예제

```kotlin
import android.content.ClipData
import android.content.ClipboardManager
import android.content.Context
import android.os.Bundle
import android.util.Log
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import com.google.android.material.snackbar.Snackbar
import com.js.sample.databinding.ActivityMainBinding

class MainActivity : AppCompatActivity() {

  private lateinit var binding: ActivityMainBinding
  private val TAG: String = "MainActivity"

  // 📌 클립보드 매니저를 lazy 로딩
  private val clipboard by lazy {
    getSystemService(Context.CLIPBOARD_SERVICE) as ClipboardManager
  }

  // 📌 클립보드 접근은 앱이 포커스를 얻었을 때 시도 (Android 10+ 보안 정책 대응)
  override fun onWindowFocusChanged(hasFocus: Boolean) {
    super.onWindowFocusChanged(hasFocus)
    if (hasFocus) {
      val clip = clipboard.primaryClip
      if (clip != null && clip.itemCount > 0) {
        val text = clip.getItemAt(0).coerceToText(this)?.toString() ?: ""
        Log.e(TAG, "복사된 텍스트=$text")
        transferMoney(text) // → 계좌번호 패턴일 경우 스낵바 띄움
      }
    }
  }

  // 📌 계좌번호 패턴 정의 (숫자 2자리 이상-2자리 이상-2자리 이상)
  private val accountRegex = Regex("""\d{2,}-\d{2,}-\d{2,}""")

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    binding = ActivityMainBinding.inflate(layoutInflater)
    setContentView(binding.root)

    // 📌 복사 버튼 클릭 시 입력값을 클립보드에 저장
    binding.copyButton.setOnClickListener {
      val text = binding.accountNumberEditText.text.toString() ?: ""
      if (text.isEmpty()) return@setOnClickListener
      copyToClipboard(this, "label", text)
    }
  }

  // 📌 복사된 텍스트가 계좌번호 형태일 경우 UI로 표시하고 스낵바 띄움
  private fun transferMoney(text: String) {
    binding.clipboardResultTextView.text = text

    if (accountRegex.matches(text)) {
      Snackbar.make(binding.root, "이체 페이지로 이동하시겠습니까?", Snackbar.LENGTH_LONG)
        .setAction("이체하기") {
          Toast.makeText(
            applicationContext,
            "이체 페이지로 이동합니다 계좌번호=$text",
            Toast.LENGTH_SHORT
          ).show()
        }
        .show()
    }
  }

  // 📌 클립보드에 텍스트를 저장하는 함수
  private fun copyToClipboard(context: Context, label: String, text: String) {
    val clipboard = context.getSystemService(Context.CLIPBOARD_SERVICE) as ClipboardManager
    val clip = ClipData.newPlainText(label, text)
    clipboard.setPrimaryClip(clip)
    Toast.makeText(context, "클립보드에 복사됨", Toast.LENGTH_SHORT).show()
  }
}
```
- 계좌번호 패턴은 2자리이상-2자리이상-2자리이상 으로 설정
- onWindowFocusChanged()는 Android 10+의 보안 정책에 따라 앱이 포커스를 갖지 않으면 클립보드 접근이 차단되는 문제를 회피하기 위해 사용합니다.
- 계좌번호를 인식하는 정규식을 유연하게 설정해 다양한 포맷을 감지할 수 있습니다.
- 클립보드를 통해 감지된 값에 **행동 유도 UI(Snackbar)**를 연결해 사용자 경험을 개선할 수 있습니다.

### 2.2 레이아웃 코드

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
        android:id="@+id/accountNumberTextView"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="24dp"
        android:text="12-345678-90"
        android:textSize="24dp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

    <Button
        android:id="@+id/copyButton"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginTop="16dp"
        android:text="계좌번호 복사하기"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toBottomOf="@id/accountNumberTextView" />

    <TextView
        android:id="@+id/clipboardResultTextView"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:gravity="center"
        android:hint="클립보드에 있는 계좌번호가 표시 됩니다"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

--- 

## 3.🧠 결론 & 보안 유의사항

- 결론
  + Android에서 클립보드 활용은 사용자 경험을 향상시킬 수 있는 강력한 도구입니다.
  + 특정 텍스트 패턴(계좌번호, 쿠폰, 인증번호 등)을 감지하여 행동 유도(UI)로 연결하면 실제 서비스 완성도도 향상됩니다.
  + Snackbar + Action, ClipboardManager를 조합하면 깔끔하고 자연스러운 UX를 만들 수 있습니다.

- 보안 유의사항
  + Android 10(API 29) 이상부터는 백그라운드 앱에서는 클립보드 접근 불가합니다.
    * 즉, 포그라운드 앱 상태에서만 감지 가능 (보안 강화 정책)
  + 사용자에게 민감 정보 사용 안내 또는 설정에서 끌 수 있는 옵션 제공 권장