---
title: (Android/UI) Android Transition API — 화면 전환 애니메이션
tags: [ Android, Animation ]
style: fill
color: dark
description: Transition 프레임워크의 Scene과 TransitionManager로 뷰 상태 변화를 자동 애니메이션하는 방법과 내장 Transition, 커스텀 Transition, Activity 전환을 정리합니다.
---

---

## 1. Transition 프레임워크란?

뷰의 **시작 상태와 끝 상태만 정의**하면, 그 사이의 애니메이션을 프레임워크가 자동으로 만들어 줍니다.

```
ObjectAnimator: 어떤 속성을 어떻게 바꿀지 직접 코딩
Transition:     "레이아웃이 바뀐다"만 알려주면 자동으로 계산해 애니메이션
```

---

## 2. TransitionManager.beginDelayedTransition

가장 많이 쓰는 방법입니다. **변경 직전에 호출**하고, 그 뒤에 뷰를 바꾸면 변화가 애니메이션됩니다.

```kotlin
binding.toggle.setOnClickListener {
    // 1. 변경 전에 호출 (ViewGroup 지정)
    TransitionManager.beginDelayedTransition(binding.container)

    // 2. 뷰 상태 변경 (표시/숨김, 크기, 위치 등)
    binding.detail.isVisible = !binding.detail.isVisible
}
```

내부적으로 변경 전/후의 속성(가시성, 위치, 크기)을 비교해 **자동으로 보간**합니다.

```kotlin
// 크기 변경도 자동 애니메이션
TransitionManager.beginDelayedTransition(container)
binding.card.updateLayoutParams { height = if (expanded) 600 else 200 }
```

---

## 3. 내장 Transition

| Transition | 효과 |
|-----------|------|
| `Fade` | 나타남/사라짐 (alpha) |
| `Slide(Gravity.BOTTOM)` | 가장자리에서 슬라이드 |
| `Explode` | 중심에서 퍼져나가듯 이동 |
| `ChangeBounds` | 위치/크기 변화 |
| `ChangeTransform` | 스케일/회전 변화 |
| `ChangeImageTransform` | ImageView 크기/scaleType 변화 |
| `AutoTransition` | Fade + ChangeBounds 조합 (기본값) |

```kotlin
val transition = Slide(Gravity.END).apply {
    duration = 300
    interpolator = FastOutSlowInInterpolator()
    addTarget(binding.panel)                  // 특정 뷰에만 적용
}
TransitionManager.beginDelayedTransition(binding.container, transition)
binding.panel.isVisible = true
```

---

## 4. TransitionSet — 조합

```kotlin
val set = TransitionSet().apply {
    ordering = TransitionSet.ORDERING_TOGETHER      // 또는 ORDERING_SEQUENTIAL
    addTransition(Fade())
    addTransition(ChangeBounds())
    duration = 250
}
TransitionManager.beginDelayedTransition(container, set)
```

---

## 5. Scene — 두 레이아웃 사이 전환

미리 정의한 두 레이아웃(Scene) 사이를 애니메이션으로 전환합니다.

```kotlin
val sceneA = Scene.getSceneForLayout(root, R.layout.scene_a, this)
val sceneB = Scene.getSceneForLayout(root, R.layout.scene_b, this)

TransitionManager.go(sceneB, ChangeBounds())      // A → B
```

두 Scene에서 **같은 `id`를 가진 뷰**는 같은 요소로 간주되어 위치/크기가 부드럽게 이동합니다.

---

## 6. ConstraintLayout과 함께

`ConstraintSet`을 적용할 때 Transition을 쓰면 제약 변화가 자연스럽게 이어집니다.

```kotlin
val expanded = ConstraintSet().apply { clone(this@Activity, R.layout.activity_main_expanded) }

TransitionManager.beginDelayedTransition(binding.root, ChangeBounds().apply { duration = 300 })
expanded.applyTo(binding.root)
```

더 정교한 제어(드래그에 따른 진행)가 필요하면 **MotionLayout**을 사용하세요.

---

## 7. Activity/Fragment 전환

```kotlin
// Activity 전환 효과 (Theme 또는 코드)
window.enterTransition = Fade()
window.exitTransition = Slide(Gravity.START)

startActivity(intent, ActivityOptions.makeSceneTransitionAnimation(this).toBundle())

// 전환 시 기본 애니메이션 (간단)
startActivity(intent)
overridePendingTransition(R.anim.slide_in_right, R.anim.slide_out_left)   // API 34+는 overrideActivityTransition 권장
```

```kotlin
// Fragment
fragment.enterTransition = MaterialFadeThrough()      // Material Components Transition
fragment.exitTransition = MaterialFadeThrough()
```

Material Components의 `MaterialFadeThrough`, `MaterialSharedAxis`, `MaterialContainerTransform`은 Material 가이드라인에 맞는 전환을 제공합니다.

---

## 8. 커스텀 Transition

```kotlin
class ColorTransition : Transition() {

    override fun captureStartValues(tv: TransitionValues) = capture(tv)
    override fun captureEndValues(tv: TransitionValues) = capture(tv)

    private fun capture(tv: TransitionValues) {
        val bg = (tv.view.background as? ColorDrawable)?.color ?: return
        tv.values["custom:color"] = bg          // 상태를 기록
    }

    override fun createAnimator(root: ViewGroup, start: TransitionValues?, end: TransitionValues?): Animator? {
        val from = start?.values?.get("custom:color") as? Int ?: return null
        val to = end?.values?.get("custom:color") as? Int ?: return null
        if (from == to) return null
        return ValueAnimator.ofArgb(from, to).apply {
            addUpdateListener { end.view.setBackgroundColor(it.animatedValue as Int) }
        }
    }
}
```

시작/끝 값을 **캡처**하고, 둘이 다르면 **Animator를 반환**하는 구조입니다.

---

## 9. 주의사항

- `beginDelayedTransition`은 **상태를 바꾸기 전에** 호출해야 합니다 (뒤에 호출하면 효과 없음)
- 변경이 많은 큰 레이아웃에 적용하면 **측정/레이아웃 비용**이 커집니다
- `RecyclerView` 아이템 변경에는 Transition 대신 `ItemAnimator`를 사용하세요
- 전환 중 뷰가 겹치거나 사라지는 문제는 `addTarget`/`excludeTarget`으로 대상을 제한하세요

---

## 10. 정리

- Transition은 **시작/끝 상태 차이를 자동으로 애니메이션**
- `TransitionManager.beginDelayedTransition(container)` 후 뷰를 변경하는 것이 기본 사용법
- `Fade`, `Slide`, `ChangeBounds`, `AutoTransition` 등 내장 효과, `TransitionSet`으로 조합
- 두 레이아웃 전환은 `Scene`, 제약 변화는 `ConstraintSet`과 함께
- 화면 전환은 Material의 `MaterialFadeThrough`/`SharedAxis` 활용
