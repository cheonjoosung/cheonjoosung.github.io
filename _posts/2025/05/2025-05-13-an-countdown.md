---
title: 안드로이드 CountDownTimer로 타이머 UI 구현하기 -  Android CountDownTimer
tags: [Android]
style: fill
color: dark
description: 안드로이드 CountDownTimer로 타이머 UI 구현하기 -  Android CountDownTimer Example
---

## ✨ 개요

`CountDownTimer`는 Android에서 **일정 시간 간격으로 작업을 수행**하거나, **남은 시간을 표시하는 타이머 UI**를 만들 때 매우 유용합니다.  
- 로그인 제한, 퀴즈 타이머, 재전송 제한 등에 자주 쓰입니다.

이번 포스트에서는 `CountDownTimer` 사용법과 함께 실전 예제를 소개합니다.

---

## 1. ✅ 주요 기능

- 초 단위의 시간을 입력 받음
- 타이머 시작/취소 기능 가능

---

## 2. ✅ CountDownTimer 코드

### 2.1 레이아웃 코드

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
          android:id="@+id/timerEditText"
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"
          android:layout_marginTop="16dp"
          android:hint="시간을 입력해주세요"
          android:inputType="numberDecimal"
          android:textSize="24sp"
          android:textStyle="bold"
          app:layout_constraintEnd_toEndOf="parent"
          app:layout_constraintStart_toStartOf="parent"
          app:layout_constraintTop_toTopOf="parent" />

  <Button
          android:id="@+id/startTimerButton"
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"
          android:layout_marginTop="16dp"
          android:text="스타트 버튼"
          android:textSize="20sp"
          app:layout_constraintEnd_toStartOf="@id/cancelTimerButton"
          app:layout_constraintStart_toStartOf="parent"
          app:layout_constraintTop_toBottomOf="@id/timerEditText" />

  <Button
          android:id="@+id/cancelTimerButton"
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"
          android:layout_marginTop="16dp"
          android:text="취소 버튼"
          android:textSize="20sp"
          app:layout_constraintEnd_toEndOf="parent"
          app:layout_constraintStart_toEndOf="@id/startTimerButton"
          app:layout_constraintTop_toBottomOf="@id/timerEditText" />


  <TextView
          android:id="@+id/timerTextView"
          android:layout_width="0dp"
          android:layout_height="wrap_content"
          android:gravity="center"
          android:textSize="30sp"
          android:textStyle="bold"
          app:layout_constraintBottom_toBottomOf="parent"
          app:layout_constraintEnd_toEndOf="parent"
          app:layout_constraintStart_toStartOf="parent"
          app:layout_constraintTop_toTopOf="parent" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

### 2.2 CountDownTimer 코드

```kotlin
import android.os.Bundle
import android.os.CountDownTimer
import android.util.Log
import androidx.appcompat.app.AppCompatActivity
import com.example.sample.databinding.ActivityMainBinding

class MainActivity : AppCompatActivity() {

  private lateinit var binding: ActivityMainBinding

  private val TAG: String = MainActivity::class.java.simpleName

  private var timer: CountDownTimer? = null

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    binding = ActivityMainBinding.inflate(layoutInflater)
    setContentView(binding.root)

    with(binding) {
      startTimerButton.setOnClickListener {
        val time = binding.timerEditText.text.toString() ?: ""
        Log.e(TAG, "Time=$time")
        if (time.isEmpty()) startTimer(60_000) // 60초
        else startTimer(time.toInt() * 1000L) // 60초
      }

      cancelTimerButton.setOnClickListener {
        cancelTimer()
      }
    }
  }

  private fun startTimer(time: Long) {
    timer?.cancel()

    timer = object : CountDownTimer(time, 1000) {
      override fun onTick(millisUntilFinished: Long) {
        val seconds = millisUntilFinished / 1000
        binding.timerTextView.text = "남은시간 $seconds 초"
      }

      override fun onFinish() {
        binding.timerTextView.text = "⏰ 종료!"
      }
    }.start()
  }

  private fun cancelTimer() {
    timer?.cancel()
    binding.timerTextView.text = "⏰ 취소!"
  }

}
```
- 타이머를 시작 전 중복 제거를 위해 cancel() 호출


--- 

## 3.💡 활용사례

- 🔐 인증 코드 60초 제한
- 📱 문자 재전송 제한 버튼
- 🧠 퀴즈 타이머
- 🧘‍♀️ 명상 타이머, 운동 앱

---

## 4.🧠 **결론**

- CountDownTimer는 Android에서 간단하고 강력한 타이머 구현 도구입니다.
- 복잡한 스레드 구현 없이도 정확한 시간 간격으로 UI 갱신이 가능하므로 다양한 앱에서 유용하게 활용됩니다.