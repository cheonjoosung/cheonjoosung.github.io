---
title: (Android/UI) Android 애니메이션 — ObjectAnimator, ValueAnimator
tags: [ Android, Animation ]
style: fill
color: dark
description: ValueAnimator와 ObjectAnimator의 동작 원리와 Interpolator, AnimatorSet 조합, ViewPropertyAnimator, 스프링 애니메이션, 생명주기 처리를 정리합니다.
---

---

## 1. 속성 애니메이션이란?

값을 **시간에 따라 변화시키고, 그 값을 객체의 속성에 적용**하는 방식입니다.
(구형 View Animation과 달리 실제 속성이 바뀌므로 터치 영역도 함께 이동합니다.)

```
ValueAnimator    : 값만 계산해 알려줌 (적용은 직접)
ObjectAnimator   : 값을 계산하고 객체의 속성(setter)에 자동 적용
```

---

## 2. ValueAnimator — 값 변화 직접 처리

```kotlin
ValueAnimator.ofFloat(0f, 1f).apply {
    duration = 600
    interpolator = DecelerateInterpolator()
    addUpdateListener { animator ->
        val value = animator.animatedValue as Float
        gaugeView.progress = value                     // 값을 원하는 곳에 직접 적용
    }
    start()
}

// 숫자 카운트업
ValueAnimator.ofInt(0, 12_500).apply {
    duration = 1000
    addUpdateListener { textView.text = "%,d".format(it.animatedValue as Int) }
    start()
}

// 색상 전환
ValueAnimator.ofArgb(Color.RED, Color.BLUE).apply {
    duration = 500
    addUpdateListener { view.setBackgroundColor(it.animatedValue as Int) }
    start()
}
```

---

## 3. ObjectAnimator — 속성에 바로 적용

`setXxx` 메서드가 있는 속성 이름을 문자열로 지정합니다.

```kotlin
ObjectAnimator.ofFloat(view, "translationX", 0f, 300f).apply {
    duration = 400
    start()
}

// View의 기본 속성
// translationX/Y, rotation, scaleX/Y, alpha 등
ObjectAnimator.ofFloat(view, View.ALPHA, 1f, 0f).start()          // 문자열 대신 상수 (오타 방지)
ObjectAnimator.ofFloat(view, View.ROTATION, 0f, 360f).start()
```

> 커스텀 속성을 애니메이션하려면 대상 객체에 `setProgress(Float)` 같은 **setter가 있어야** 합니다. Kotlin의 `var progress`는 setter가 생성되므로 `"progress"`로 사용할 수 있습니다.

---

## 4. ViewPropertyAnimator — 가장 간단한 방법

View의 흔한 변환은 `animate()`로 체이닝합니다. 여러 속성이 **하나의 애니메이터**로 최적화됩니다.

```kotlin
view.animate()
    .translationY(-200f)
    .alpha(0.5f)
    .scaleX(1.2f).scaleY(1.2f)
    .setDuration(300)
    .setInterpolator(OvershootInterpolator())
    .withEndAction { view.isVisible = false }         // 종료 시 실행
    .start()
```

단순 이동/페이드/확대는 `animate()`가 가장 간결하고 효율적입니다.

---

## 5. Interpolator — 속도 곡선

| Interpolator | 느낌 |
|-------------|------|
| `LinearInterpolator` | 일정한 속도 |
| `AccelerateInterpolator` | 점점 빨라짐 |
| `DecelerateInterpolator` | 점점 느려짐 (자연스러운 정지) |
| `AccelerateDecelerateInterpolator` | 천천히 → 빠르게 → 천천히 (기본값) |
| `OvershootInterpolator` | 목표를 살짝 넘었다 돌아옴 |
| `BounceInterpolator` | 튕김 |
| `FastOutSlowInInterpolator` | Material 표준 |

화면에 **들어올 때는 Decelerate**, **나갈 때는 Accelerate**를 쓰면 자연스럽습니다.

---

## 6. AnimatorSet — 여러 애니메이션 조합

```kotlin
val fadeIn = ObjectAnimator.ofFloat(card, View.ALPHA, 0f, 1f)
val slideUp = ObjectAnimator.ofFloat(card, View.TRANSLATION_Y, 100f, 0f)
val scale = ObjectAnimator.ofFloat(icon, View.SCALE_X, 0.8f, 1f)

AnimatorSet().apply {
    playTogether(fadeIn, slideUp)             // 동시에
    play(scale).after(300)                    // 300ms 뒤에 이어서
    duration = 400
    start()
}

// 순차 실행
AnimatorSet().apply { playSequentially(anim1, anim2, anim3); start() }
```

---

## 7. 반복과 리스너

```kotlin
ObjectAnimator.ofFloat(heart, View.SCALE_X, 1f, 1.3f).apply {
    duration = 500
    repeatCount = ValueAnimator.INFINITE
    repeatMode = ValueAnimator.REVERSE         // 갔다가 되돌아오기
    start()
}

animator.addListener(object : AnimatorListenerAdapter() {
    override fun onAnimationEnd(animation: Animator) { /* 종료 */ }
    override fun onAnimationCancel(animation: Animator) { /* 취소 */ }
})
```

---

## 8. 스프링 애니메이션 (물리 기반)

```kotlin
// androidx.dynamicanimation:dynamicanimation-ktx
SpringAnimation(view, DynamicAnimation.TRANSLATION_Y, 0f).apply {
    spring.stiffness = SpringForce.STIFFNESS_LOW       // 강성
    spring.dampingRatio = SpringForce.DAMPING_RATIO_MEDIUM_BOUNCY
    start()
}
```

시간 대신 **물리 법칙**으로 움직여 드래그 후 되돌아오는 동작처럼 사용자 입력에 자연스럽게 이어집니다.

---

## 9. 주의사항 — 생명주기

```kotlin
override fun onDestroyView() {
    animator?.cancel()                // 무한 반복/긴 애니메이션은 반드시 정리
    super.onDestroyView()
}
```

- 반복(`INFINITE`) 애니메이션을 정리하지 않으면 **메모리 누수와 배터리 소모**가 발생합니다
- 사용자 설정(접근성의 애니메이션 끄기)을 고려하세요 (`ValueAnimator.areAnimatorsEnabled()`)
- 레이아웃 크기를 바꾸는 애니메이션(`width/height`)은 매 프레임 `requestLayout`이 호출되어 느립니다. `translation/scale/alpha`처럼 **레이아웃을 건드리지 않는 속성**을 우선하세요

---

## 10. 정리

- `ValueAnimator`는 값 계산, `ObjectAnimator`는 값 계산 + 속성 적용
- 단순한 View 변환은 `view.animate()`가 가장 간결
- `Interpolator`로 속도 곡선, `AnimatorSet`으로 동시/순차 조합
- 물리적인 움직임은 `SpringAnimation`
- 반복 애니메이션은 해제하고, 레이아웃을 유발하지 않는 속성을 선택
