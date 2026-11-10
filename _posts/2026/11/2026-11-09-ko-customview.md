---
title: (Android/UI) CustomView 만들기 — onDraw, onMeasure, onLayout
tags: [ Android, CustomView ]
style: fill
color: dark
description: 안드로이드 커스텀 뷰의 생성자, onMeasure, onLayout, onDraw 동작 순서와 attrs 속성 정의, 터치 처리, 상태 저장 방법을 예제로 정리합니다.
---

---

## 1. 커스텀 뷰가 필요한 경우

기본 위젯 조합으로 표현하기 어려운 UI(원형 프로그레스, 차트, 서명 패드 등)는 `View`를 직접 상속해 그립니다.

| 방식 | 설명 | 예 |
|------|------|-----|
| 기존 View 확장 | `AppCompatButton` 등을 상속해 일부만 변경 | 둥근 버튼 |
| 복합 뷰 (Compound) | `ViewGroup`에 기존 뷰들을 묶음 | 별점 + 숫자 입력 |
| `View` 직접 구현 | `onDraw`로 직접 그림 | 차트, 게이지 |

---

## 2. 뷰의 그리기 과정

```
생성자 → onAttachedToWindow
   ↓
onMeasure(크기 결정)  →  onLayout(위치 결정, ViewGroup)  →  onDraw(그리기)
   ↑                                                           ↑
requestLayout()                                           invalidate()
```

- 크기나 위치가 바뀌면 `requestLayout()` → measure/layout/draw 전체 재실행
- 모양만 바뀌면 `invalidate()` → draw만 재실행 (훨씬 저렴)

---

## 3. 기본 골격 — 원형 게이지

```kotlin
class CircleGaugeView @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : View(context, attrs, defStyleAttr) {

    // Paint는 onDraw 밖에서 미리 생성 (onDraw는 초당 수십 번 호출됨)
    private val trackPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = 24f
        color = Color.LTGRAY
    }
    private val progressPaint = Paint(Paint.ANTI_ALIAS_FLAG).apply {
        style = Paint.Style.STROKE
        strokeWidth = 24f
        strokeCap = Paint.Cap.ROUND
        color = Color.parseColor("#3F51B5")
    }
    private val arcBounds = RectF()

    var progress: Float = 0f           // 0f..1f
        set(value) {
            field = value.coerceIn(0f, 1f)
            invalidate()               // 값이 바뀌면 다시 그리기
        }

    override fun onSizeChanged(w: Int, h: Int, oldw: Int, oldh: Int) {
        // 크기가 확정되면 그릴 영역을 한 번만 계산
        val inset = trackPaint.strokeWidth / 2 + paddingLeft
        arcBounds.set(inset, inset, w - inset, h - inset)
    }

    override fun onDraw(canvas: Canvas) {
        canvas.drawArc(arcBounds, 0f, 360f, false, trackPaint)                     // 배경 원
        canvas.drawArc(arcBounds, -90f, 360f * progress, false, progressPaint)     // 진행 호 (12시 방향 시작)
    }
}
```

> `onDraw` 안에서 `Paint`, `Path`, `RectF` 같은 객체를 **새로 생성하지 마세요.** 매 프레임 GC가 발생해 끊김의 원인이 됩니다.

---

## 4. onMeasure — 크기 결정

`wrap_content`일 때 원하는 크기를 직접 알려줘야 합니다. 기본 `View`는 `wrap_content`도 `match_parent`처럼 동작합니다.

```kotlin
override fun onMeasure(widthMeasureSpec: Int, heightMeasureSpec: Int) {
    val desired = (120 * resources.displayMetrics.density).toInt()   // 기본 크기 120dp

    val w = resolveSize(desired + paddingLeft + paddingRight, widthMeasureSpec)
    val h = resolveSize(desired + paddingTop + paddingBottom, heightMeasureSpec)

    val size = minOf(w, h)               // 정사각형 유지
    setMeasuredDimension(size, size)     // 반드시 호출해야 함
}
```

| MeasureSpec 모드 | 의미 |
|-----------------|------|
| `EXACTLY` | `match_parent`/고정 dp — 그 크기로 |
| `AT_MOST` | `wrap_content` — 최대 크기 이내에서 원하는 크기 |
| `UNSPECIFIED` | 제한 없음 (ScrollView 내부 등) |

`resolveSize()`가 모드에 맞게 값을 결정해 주므로 직접 분기할 필요가 적습니다.

---

## 5. onLayout — ViewGroup에서만

단일 `View`는 `onLayout`이 필요 없습니다. 자식을 배치하는 `ViewGroup`을 만들 때 구현합니다.

```kotlin
class SimpleStackLayout(context: Context, attrs: AttributeSet?) : ViewGroup(context, attrs) {

    override fun onMeasure(widthSpec: Int, heightSpec: Int) {
        var totalHeight = 0
        var maxWidth = 0
        for (i in 0 until childCount) {
            val child = getChildAt(i)
            measureChild(child, widthSpec, heightSpec)       // 자식 측정
            totalHeight += child.measuredHeight
            maxWidth = maxOf(maxWidth, child.measuredWidth)
        }
        setMeasuredDimension(
            resolveSize(maxWidth, widthSpec),
            resolveSize(totalHeight, heightSpec)
        )
    }

    override fun onLayout(changed: Boolean, l: Int, t: Int, r: Int, b: Int) {
        var top = 0
        for (i in 0 until childCount) {
            val child = getChildAt(i)
            child.layout(0, top, child.measuredWidth, top + child.measuredHeight)   // 위치 확정
            top += child.measuredHeight
        }
    }
}
```

---

## 6. XML 속성 (attrs) 정의

```xml
<!-- res/values/attrs.xml -->
<declare-styleable name="CircleGaugeView">
    <attr name="trackColor" format="color" />
    <attr name="progressColor" format="color" />
    <attr name="strokeWidth" format="dimension" />
</declare-styleable>
```

```kotlin
init {
    context.obtainStyledAttributes(attrs, R.styleable.CircleGaugeView, defStyleAttr, 0).use { a ->
        trackPaint.color = a.getColor(R.styleable.CircleGaugeView_trackColor, Color.LTGRAY)
        progressPaint.color = a.getColor(R.styleable.CircleGaugeView_progressColor, Color.BLUE)
        val width = a.getDimension(R.styleable.CircleGaugeView_strokeWidth, 24f)
        trackPaint.strokeWidth = width
        progressPaint.strokeWidth = width
    }
}
```

```xml
<com.example.CircleGaugeView
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    app:progressColor="#FF5722"
    app:strokeWidth="12dp" />
```

> `TypedArray`는 사용 후 `recycle()`(또는 Kotlin `use`)해야 합니다.

---

## 7. 상태 저장 — 화면 회전 대응

```kotlin
override fun onSaveInstanceState(): Parcelable =
    Bundle().apply {
        putParcelable("super", super.onSaveInstanceState())
        putFloat("progress", progress)
    }

override fun onRestoreInstanceState(state: Parcelable?) {
    if (state is Bundle) {
        progress = state.getFloat("progress")
        super.onRestoreInstanceState(state.getParcelable("super"))
    } else super.onRestoreInstanceState(state)
}
```

뷰에 **`android:id`가 있어야** 저장/복원이 동작합니다.

---

## 8. 정리

- 흐름: 생성자 → `onMeasure` → `onLayout` → `onDraw`
- 크기 변경은 `requestLayout()`, 모양 변경은 `invalidate()`
- `onDraw`에서는 객체를 생성하지 말고 `Paint`/`RectF`를 미리 준비
- `wrap_content`를 지원하려면 `onMeasure`에서 `setMeasuredDimension` 호출
- `attrs.xml`로 XML 속성을 노출하고, `onSaveInstanceState`로 회전 시 상태 보존
