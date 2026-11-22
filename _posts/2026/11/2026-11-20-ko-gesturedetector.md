---
title: (Android/UI) GestureDetector — 스와이프, 롱프레스, 더블탭 감지
tags: [ Android, Gesture ]
style: fill
color: dark
description: GestureDetector로 탭, 더블탭, 롱프레스, 플링(스와이프)을 감지하는 방법과 ScaleGestureDetector 핀치 줌, onTouchEvent 연동, Compose 제스처를 정리합니다.
---

---

## 1. 터치 이벤트의 한계

`onTouchEvent`는 `DOWN`, `MOVE`, `UP` 같은 **원시 이벤트**만 줍니다.
"더블탭인지, 롱프레스인지, 빠른 스와이프(fling)인지"는 시간과 거리를 직접 계산해야 하는데, `GestureDetector`가 이를 대신 해줍니다.

---

## 2. 기본 사용법

```kotlin
class GestureView(context: Context, attrs: AttributeSet? = null) : View(context, attrs) {

    private val detector = GestureDetector(context, object : GestureDetector.SimpleOnGestureListener() {

        // 반드시 true를 반환해야 이후 제스처(롱프레스, 플링 등)가 전달됨
        override fun onDown(e: MotionEvent) = true

        override fun onSingleTapConfirmed(e: MotionEvent): Boolean {
            // 더블탭이 아님이 확정된 단일 탭
            performClick()
            return true
        }

        override fun onDoubleTap(e: MotionEvent): Boolean {
            // 더블탭 (예: 좋아요, 확대)
            return true
        }

        override fun onLongPress(e: MotionEvent) {
            // 롱프레스 (예: 컨텍스트 메뉴)
        }

        override fun onFling(
            e1: MotionEvent?, e2: MotionEvent, velocityX: Float, velocityY: Float
        ): Boolean {
            // 빠른 스와이프 — 속도로 방향 판별
            return true
        }

        override fun onScroll(
            e1: MotionEvent?, e2: MotionEvent, distanceX: Float, distanceY: Float
        ): Boolean {
            // 손가락을 끌 때 (이동 거리)
            return true
        }
    })

    @SuppressLint("ClickableViewAccessibility")
    override fun onTouchEvent(event: MotionEvent): Boolean =
        detector.onTouchEvent(event) || super.onTouchEvent(event)
}
```

---

## 3. 콜백 정리

| 콜백 | 시점 |
|------|------|
| `onDown` | 손가락이 닿음 (**true 반환 필수**) |
| `onSingleTapUp` | 탭 후 손을 뗌 (더블탭일 수도 있음) |
| `onSingleTapConfirmed` | **더블탭이 아니라고 확정**된 단일 탭 (약간 지연됨) |
| `onDoubleTap` | 두 번째 탭이 눌릴 때 |
| `onLongPress` | 길게 누름 |
| `onScroll` | 손가락을 끌며 이동 |
| `onFling` | 빠르게 쓸고 뗌 (속도 포함) |

> `onSingleTapUp`은 더블탭의 첫 탭에서도 호출됩니다. 더블탭과 구분해야 한다면 **`onSingleTapConfirmed`** 를 사용하세요. 대신 더블탭 판정 시간(약 300ms)만큼 반응이 늦어집니다.

---

## 4. 스와이프 방향 판별 — onFling

```kotlin
override fun onFling(e1: MotionEvent?, e2: MotionEvent, vx: Float, vy: Float): Boolean {
    if (e1 == null) return false
    val dx = e2.x - e1.x
    val dy = e2.y - e1.y
    val minDistance = 100f                       // 최소 이동 거리
    val minVelocity = 100f                       // 최소 속도

    if (abs(dx) > abs(dy)) {                     // 가로 방향이 더 큼
        if (abs(dx) > minDistance && abs(vx) > minVelocity) {
            if (dx > 0) onSwipeRight() else onSwipeLeft()
            return true
        }
    } else {
        if (abs(dy) > minDistance && abs(vy) > minVelocity) {
            if (dy > 0) onSwipeDown() else onSwipeUp()
            return true
        }
    }
    return false
}
```

임계값은 `ViewConfiguration`의 시스템 값을 활용하면 기기에 맞습니다.

```kotlin
val config = ViewConfiguration.get(context)
val minVelocity = config.scaledMinimumFlingVelocity
val touchSlop = config.scaledTouchSlop           // 이동으로 인정하는 최소 거리
```

---

## 5. 이미 있는 뷰에 적용 (View.setOnTouchListener)

```kotlin
val detector = GestureDetector(context, object : GestureDetector.SimpleOnGestureListener() {
    override fun onDown(e: MotionEvent) = true
    override fun onDoubleTap(e: MotionEvent): Boolean {
        likeAnimation(); return true
    }
})

imageView.setOnTouchListener { v, event ->
    detector.onTouchEvent(event)
    if (event.action == MotionEvent.ACTION_UP) v.performClick()   // 접근성 경고 방지
    true
}
```

---

## 6. 핀치 줌 — ScaleGestureDetector

```kotlin
private var scale = 1f
private val scaleDetector = ScaleGestureDetector(context,
    object : ScaleGestureDetector.SimpleOnScaleGestureListener() {
        override fun onScale(detector: ScaleGestureDetector): Boolean {
            scale = (scale * detector.scaleFactor).coerceIn(0.5f, 5f)    // 배율 제한
            invalidate()
            return true
        }
    })

override fun onTouchEvent(event: MotionEvent): Boolean {
    scaleDetector.onTouchEvent(event)
    gestureDetector.onTouchEvent(event)               // 두 Detector를 함께 사용
    return true
}
```

---

## 7. 부모 스크롤과의 충돌

`ScrollView`/`ViewPager`/`RecyclerView` 안에서 가로 스와이프를 처리하면 **부모가 이벤트를 가로챕니다.**

```kotlin
override fun onDown(e: MotionEvent): Boolean {
    parent.requestDisallowInterceptTouchEvent(true)   // 부모가 가로채지 못하게 요청
    return true
}
// ACTION_UP/CANCEL에서 다시 false로 되돌리기
```

---

## 8. Jetpack Compose 제스처

```kotlin
Box(
    Modifier.pointerInput(Unit) {
        detectTapGestures(
            onTap = { /* 탭 */ },
            onDoubleTap = { /* 더블탭 */ },
            onLongPress = { /* 롱프레스 */ }
        )
    }
)

// 드래그
Modifier.pointerInput(Unit) {
    detectDragGestures { change, dragAmount ->
        change.consume()
        offset += dragAmount
    }
}

// 핀치/팬/회전
Modifier.pointerInput(Unit) {
    detectTransformGestures { _, pan, zoom, rotation -> /* ... */ }
}

// 단순 클릭 계열은 Modifier.combinedClickable
Modifier.combinedClickable(onClick = {}, onDoubleClick = {}, onLongClick = {})
```

---

## 9. 접근성과 주의사항

- 제스처만으로 제공되는 기능은 **TalkBack 사용자가 쓸 수 없습니다.** 버튼 등 **대체 수단**을 함께 제공하세요
- `onTouchEvent`를 재정의하면 `performClick()` 경고가 나올 수 있으니 클릭 동작은 `performClick()`을 호출하세요
- 롱프레스/더블탭 시간(`ViewConfiguration.getLongPressTimeout()`)은 사용자 설정에 따라 다릅니다
- 제스처가 많으면 충돌하므로 **한 화면에서 사용하는 제스처 종류를 줄이세요**

---

## 10. 정리

- `GestureDetector`가 탭/더블탭/롱프레스/스크롤/플링을 감지, **`onDown`은 true 반환 필수**
- 더블탭과 구분하려면 `onSingleTapConfirmed`, 스와이프는 `onFling`의 거리와 속도로 판단
- 핀치 줌은 `ScaleGestureDetector`, 부모와의 충돌은 `requestDisallowInterceptTouchEvent`
- Compose는 `detectTapGestures`, `detectDragGestures`, `combinedClickable`
- 제스처 전용 기능에는 **접근성 대체 수단**을 제공
