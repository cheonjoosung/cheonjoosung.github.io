---
title: Android View의 visibility 속성 제어 방법 (VISIBLE, GONE, INVISIBLE)
tags: [Android]
style: fill
color: dark
description: Android에서 View의 visibility 속성을 제어하는 방법과 VISIBLE, GONE, INVISIBLE의 차이점 및 활용 예시를 소개합니다.
---

## ✨ 개요

Android 앱을 개발하다 보면 UI 요소를 보이거나 숨기는 일이 매우 자주 발생합니다.  
이때 사용하는 것이 바로 **`View.visibility` 속성**입니다.

- `VISIBLE`: 보임
- `INVISIBLE`: 공간은 유지, 보이지 않음
- `GONE`: 보이지 않으며 공간도 사라짐

이 포스트에서는 각 속성의 차이와 실제 사용 방법을 소개합니다.

---

## 1. ✅ visibility 속성 종류 비교

| 속성        | 설명                                     | 공간 차지 여부 |
|-------------|------------------------------------------|----------------|
| `VISIBLE`   | View가 **화면에 표시됨**                  | 차지함         |
| `INVISIBLE` | View가 **보이지 않지만 레이아웃 공간 차지** | 차지함         |
| `GONE`      | View가 **보이지 않고 공간도 제거됨**        | 안 차지함      |

---

## 2. ✅ Visibility 코드

### 2.1 **코틀린 코드**

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    binding = ActivityMainBinding.inflate(layoutInflater)
    setContentView(binding.root)

    with(binding) {
        visibleButton.setOnClickListener {
            button.isVisible = true  //button.visibility = View.VISIBLE
            checkVisibility(button)
        }

        invisibleButton.setOnClickListener {
            button.isInvisible = true  //button.visibility = View.INVISIBLE
            checkVisibility(button)
        }

        goneButton.setOnClickListener {
            button.isGone = true  //button.visibility = View.GONE
            checkVisibility(button)
        }
    }

}

private fun checkVisibility(view: View) {
    when (view.visibility) {
        View.VISIBLE -> Log.e(TAG, "VISIBLE")
        View.INVISIBLE -> Log.e(TAG, "INVISIBLE")
        View.GONE -> Log.e(TAG, "GONE")
        else -> Log.e(TAG, "NONE")
    }
}
```
- visibility 를 이용한 VIEW.GONE/VISIBLE/INVISIBLE 설정 가능
- isGone, isVisible, isInvisible 를 이용한 true/false 설정 가능

### 2.2 **확장 함수**

```kotlin
// 확장함수 사용
fun View.visible() { visibility = View.VISIBLE }
fun View.invisible() { visibility = View.INVISIBLE }
fun View.gone() { visibility = View.GONE }

with(binding) {
    visibleButton.setOnClickListener {
        button.visible()
    }

    invisibleButton.setOnClickListener {
        button.invisible()
    }

    goneButton.setOnClickListener {
        button.gone()
    }
}
```
- 확장함수를 사용하면 유동적으로 뷰 가시성을 범용적 사용이 가능 

---

## 3. 🔍 주의사항

- INVISIBLE은 공간을 유지하므로 레이아웃 밀림을 방지하고 싶을 때 사용
- GONE은 공간도 제거하므로, 레이아웃이 유동적으로 재배치됨
- ViewStub이나 ConstraintLayout의 goneMargin 등과 함께 사용할 수 있음

---

## 4.🧠 **결론**

- View.visibility 속성은 UI를 제어하는 핵심 도구입니다.
- 상황에 맞게 VISIBLE, INVISIBLE, GONE을 잘 활용하면 사용자 경험을 더욱 향상시킬 수 있습니다.
- 로딩 상태, 에러 안내, 빈 데이터 처리 등 실무에서 다양한 상황에 유용하게 활용됩니다.