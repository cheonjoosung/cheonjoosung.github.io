---
title: 안드로이드 패턴 patternLockView 쉽게 구현하는 방법 - Android PatternLockView 
tags: [Android]
style: fill
color: dark
description: Android에서 패턴 락 UI를 라이브러리 없이 직접 구현하는 방법을 소개합니다. 안드로이드 패턴 patternLockView 쉽게 구현하는 방법 - Android PatternLockView
---

## ✨ 개요

Android에서는 기본적으로 PIN, 패턴, 생체인증 등을 제공하지만, 자체 인증 화면을 만들거나 보조 잠금 화면을 구성할 때 **패턴 락(Pattern Lock)** UI가 유용하게 쓰입니다.

이번 포스팅에서는 **라이브러리 없이 순수 Custom View로 Pattern Lock UI를 구현하는 방법**을 소개합니다.

---

## 1. ✅ 주요 기능

| 항목               | 설명                                   |
|------------------|--------------------------------------|
| `Custom View`    | View를 상속 받아 직접 그리기(Canvas)           |
| `onDraw()`       | 패턴 원 및 선을 그림                         |
| `onTouchEvent()` | 사용자 입력(터치 좌표)을 처리                    |
| `패턴 배열 반환`       | 이어진 선 목록 및 해쉬 값 |

---

## 2. ✅ 패턴 PatternLockView 코드

### 2.1 패턴 PatternLockView 코드

```kotlin
import android.content.Context
import android.graphics.*
import android.util.AttributeSet
import android.view.MotionEvent
import android.view.View
import java.security.MessageDigest
import kotlin.math.hypot

class PatternLockView(context: Context, attrs: AttributeSet?) : View(context, attrs) {

  interface PatternListener {
    fun onPatternDetected(pattern: List<Int>, hash: String)
  }

  var patternListener: PatternListener? = null
  var showLines: Boolean = true

  private val gridSize = 3
  private val points = Array(gridSize) { Array(gridSize) { PointF() } }

  // 선택된 점 인덱스 및 좌표
  private val selectedIndices = mutableListOf<Int>()
  private val selectedPoints = mutableListOf<PointF>()
  private val selectedIndexSet = mutableSetOf<Int>() // 최적화를 위한 Set (contains 빠름)

  // 점 및 선 색상 설정
  private val defaultPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
    color = Color.GRAY
    style = Paint.Style.FILL
  }

  // 선택된 점 색상
  private val selectedPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
    color = Color.BLUE
    style = Paint.Style.FILL
  }

  private val errorPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
    color = Color.RED
    style = Paint.Style.FILL
  }

  // 이어진 선 색상
  private val linePaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
    color = Color.BLUE
    strokeWidth = 10f
    style = Paint.Style.STROKE
  }

  private val errorLinePaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
    color = Color.RED
    strokeWidth = 10f
    style = Paint.Style.STROKE
  }

  private var currentTouchX = -1f
  private var currentTouchY = -1f
  private var isError = false

  // 뷰 크기 변경 시 각 점 좌표 계산
  override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
    val offsetX = w / (gridSize + 1)
    val offsetY = h / (gridSize + 1)
    for (i in 0 until gridSize) {
      for (j in 0 until gridSize) {
        points[i][j] = PointF((j + 1) * offsetX.toFloat(), (i + 1) * offsetY.toFloat())
      }
    }
  }

  // 점과 선을 그림
  override fun onDraw(canvas: Canvas) {
    super.onDraw(canvas)

    // 점 그리기
    for (i in 0 until gridSize) {
      for (j in 0 until gridSize) {
        val index = i * gridSize + j
        val p = points[i][j]
        val paint = when {
          isError && selectedIndexSet.contains(index) && showLines -> errorPaint
          selectedIndexSet.contains(index) && showLines -> selectedPaint
          else -> defaultPaint
        }
        canvas.drawCircle(p.x, p.y, 30f, paint)
      }
    }

    // 선 그리기
    if (showLines && selectedPoints.isNotEmpty()) {
      val paint = if (isError) errorLinePaint else linePaint
      for (i in 0 until selectedPoints.size - 1) {
        canvas.drawLine(
          selectedPoints[i].x,
          selectedPoints[i].y,
          selectedPoints[i + 1].x,
          selectedPoints[i + 1].y,
          paint
        )
      }

      // 현재 터치 중이면 마지막 점에서 손가락까지 선 그리기
      if (currentTouchX >= 0 && currentTouchY >= 0) {
        val last = selectedPoints.last()
        canvas.drawLine(last.x, last.y, currentTouchX, currentTouchY, paint)
      }
    }
  }

  // 터치 이벤트 처리
  override fun onTouchEvent(event: MotionEvent): Boolean {
    when (event.action) {
      MotionEvent.ACTION_DOWN, MotionEvent.ACTION_MOVE -> {
        currentTouchX = event.x
        currentTouchY = event.y
        val (p, index) = findNearestPoint(event.x, event.y) ?: return true
        if (selectedIndexSet.add(index)) {
          selectedIndices.add(index)
          selectedPoints.add(p)
        }
        invalidate()
      }

      MotionEvent.ACTION_UP -> {
        currentTouchX = -1f
        currentTouchY = -1f
        if (selectedIndices.isNotEmpty()) {
          val hash = hashPattern(selectedIndices)
          patternListener?.onPatternDetected(selectedIndices.toList(), hash)
        }
        resetPattern()
      }
    }
    return true
  }

  // 터치 위치가 가까운 점을 찾음
  private fun findNearestPoint(x: Float, y: Float): Pair<PointF, Int>? {
    for (i in 0 until gridSize) {
      for (j in 0 until gridSize) {
        val p = points[i][j]
        if (hypot(x - p.x, y - p.y) < 50f) {
          val index = i * gridSize + j
          return p to index
        }
      }
    }
    return null
  }

  // 잘못된 패턴 입력 시 애니메이션과 함께 초기화
  fun showErrorAndReset() {
    isError = true
    invalidate()

    this.animate().translationX(30f).setDuration(50).withEndAction {
      this.animate().translationX(-30f).setDuration(50).withEndAction {
        this.animate().translationX(0f).setDuration(50).start()
      }.start()
    }.start()

    postDelayed({
      isError = false
      resetPattern()
    }, 1000)
  }

  // 패턴 리셋
  private fun resetPattern() {
    selectedIndices.clear()
    selectedPoints.clear()
    selectedIndexSet.clear()
    invalidate()
  }

  // 패턴을 해시 문자열로 변환 (SHA-256)
  private fun hashPattern(pattern: List<Int>): String {
    val message = pattern.joinToString("")
    val digest = MessageDigest.getInstance("SHA-256")
    val hashBytes = digest.digest(message.toByteArray())
    return hashBytes.joinToString("") { "%02x".format(it) }
  }
}
```
- onSizeChanged()
  + View의 크기가 결정될 때마다 호출되어 원 9개의 위치 좌표를 계산합니다.
  + offsetX, offsetY를 사용해 균등하게 배치합니다.
- override fun onDraw(canvas: Canvas)
  + 모든 원을 반복하며 선택 상태에 따라 색상을 달리하여 원을 그림.
  + 사용자가 선택한 경로를 선으로 연결합니다.
  + 현재 터치 중일 경우 마지막 점에서 현재 터치 지점까지 선을 연장합니다.
- onTouchEvent()
  + 터치 이벤트(DOWN, MOVE, UP)를 받아 처리합니다.
  + ACTION_DOWN/MOVE: 사용자의 손가락 위치가 점 내부에 들어오면 선택 처리 
  + ACTION_UP: 입력 완료 시 해시값을 구해 콜백 전달 후 초기화
- findNearestPoint()
  + 사용자가 터치한 지점에서 가장 가까운 원을 찾아 반환합니다. 
  + 원의 반지름 거리 안에 있는 경우 인식.
- showErrorAndReset()
  + 잘못된 패턴일 경우 붉은색으로 점/선을 표시하며, 약간의 진동 애니메이션 후 자동 초기화됩니다.
  + 보안성 향상과 사용자 경험 개선을 위한 기능입니다.
- hashPattern()
  + 사용자의 패턴 입력 리스트(Int 배열)를 SHA-256 해시로 변환합니다.
  + 서버 전송, 저장 시 패턴을 평문으로 저장하지 않기 위한 보안 기능입니다


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
          android:id="@+id/resultTextView"
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"
          android:layout_marginTop="16dp"
          android:text="패턴을 그려주세요!!"
          android:textSize="24sp"
          android:textStyle="bold"
          app:layout_constraintEnd_toEndOf="parent"
          app:layout_constraintStart_toStartOf="parent"
          app:layout_constraintTop_toTopOf="parent" />

  <com.example.sample.PatternLockView
          android:id="@+id/patternLockView"
          android:layout_width="0dp"
          android:layout_height="500dp"
          android:layout_margin="24dp"
          app:layout_constraintEnd_toEndOf="parent"
          app:layout_constraintStart_toStartOf="parent"
          app:layout_constraintTop_toBottomOf="@id/resultTextView" />

  <CheckBox
          android:id="@+id/checkBox"
          android:layout_width="wrap_content"
          android:layout_height="wrap_content"
          android:checked="false"
          android:text="패턴 숨기기"
          app:layout_constraintEnd_toEndOf="@id/patternLockView"
          app:layout_constraintStart_toStartOf="@id/patternLockView"
          app:layout_constraintTop_toBottomOf="@id/patternLockView" />

</androidx.constraintlayout.widget.ConstraintLayout>
```
- View 를 PatternLockView 로 설정
- CheckBox 패턴 숨기기 여부

### 2.3 호출 방법

```kotlin
import android.os.Bundle
import android.util.Log
import androidx.appcompat.app.AppCompatActivity
import com.example.sample.databinding.ActivityMainBinding

class MainActivity : AppCompatActivity() {

    private lateinit var binding: ActivityMainBinding

    private val TAG: String = MainActivity::class.java.simpleName

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        binding.patternLockView.patternListener = object : PatternLockView.PatternListener {
            override fun onPatternDetected(pattern: List<Int>, hash: String) {
                Log.e(TAG, "patternNumber=${pattern.joinToString(",")}  hash=$hash")
            }
        }

        binding.checkBox.setOnCheckedChangeListener { _, isChecked ->
            binding.patternLockView.showLines = !isChecked
            binding.patternLockView.invalidate()
        }

    }

}
```
- 패턴 입력 값의 결과를 선택된 점의 목록과 해쉬값으로 받을 수 있기에 패턴 등록/변경 시 활용 가능 함
- 체크박스(패턴 숨기기 여부)를 통해 패턴 보안 강화 가능

--- 

## 3.🧠 결론 & 보안 유의사항

- 결론
  + Pattern Lock UI는 직접 구현해보면 Canvas, View, Touch 처리에 대한 이해를 높일 수 있습니다.
  + 라이브러리에 의존하지 않고 앱에 맞는 UX를 자유롭게 구성할 수 있는 장점이 있습니다.
  + 더 고도화된 UX가 필요하다면, 애니메이션, 진동, 보안 알고리즘 등을 함께 고려해보세요 😊