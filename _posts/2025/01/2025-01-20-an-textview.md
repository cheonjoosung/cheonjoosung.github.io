---
title: Android TextView 취소선 긋는 방법 with 확장함수 - Android How To TextView StrikeThrough
tags: [Android]
style: fill
color: dark
description: Android TextView 취소선 긋는 방법 with 확장함수 - Android How To TextView StrikeThrough
---

## 개요

Android에서 TextView에 취소선을 추가하거나 제거할 때, 확장 함수를 활용하면 코드의 가독성과 재사용성을 크게 높일 수 있습니다.

확장 함수는 기존 클래스(TextView)의 코드를 수정하지 않고도 새로운 기능을 추가할 수 있어 유지보수성과 생산성을 동시에 향상시킵니다.

---

## 1. TextView 확장 함수 만들기
- or Paint.STRIKE_THRU_TEXT_FLAG: 기존 플래그에 취소선 플래그를 추가합니다.
- or Paint.STRIKE_THRU_TEXT_FLAG.inv(): 기존 플래그에 취소선 플래그를 제거합니다.

```kotlin
// TextView 취소선 추가 확장 함수
private fun TextView.addStrikeThrough() {
    paintFlags = this.paintFlags or Paint.STRIKE_THRU_TEXT_FLAG
}

// TextView 취소선 제거 확장 함수
private fun TextView.removeStrikeThrough() {
    paintFlags = paintFlags and Paint.STRIKE_THRU_TEXT_FLAG.inv()
}
```

---

## 2. 확장함수 호출하여 텍스트 뷰 취소선 긋기
- 간편하게 TextView.addStrikeThrough() 를 호출하여 취소선을 그을 수 있고
- TextView.removeStrikeThrough() 를 호출하여 원상태로 복구할 수 있습니다

```kotlin
var isChecked = false

binding.button.setOnClickListener {
    if (isChecked) {
        binding.titleTextView.removeStrikeThrough()
        binding.contentTextView.removeStrikeThrough()
    } else {
        binding.titleTextView.addStrikeThrough()
        binding.contentTextView.addStrikeThrough()
    }

    isChecked = !isChecked
}
```

---

## 3.  확장 함수의 장점

```kotlin
// 일반함수 사용 시
fun addStrikeThrough(textView: TextView) {
    textView.paintFlags = textView.paintFlags or Paint.STRIKE_THRU_TEXT_FLAG
}

addStrikeThrough(binding.titleTextView)

// 확장함수 사용시
binding.titleTextView.addStrikeThrough()
```
- 확장 함수를 사용하면 호출부가 직관적이고 클래스의 기본 기능처럼 보이도록 작성할 수 있습니다.
- 반복적으로 사용하는 취소선 설정 또는 제거 작업을 하나의 함수로 추상화하여 중복 코드를 제거합니다.
