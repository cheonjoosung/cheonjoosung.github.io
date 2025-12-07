---
title: (Android/안드로이드) RecyclerView 에서 TextView 더보기/접기 구현하기 
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) RecyclerView 에서 TextView 더보기/접기 구현하기 (maxLines & post) 2줄 이상일 때 자동으로 확장 버튼 표시하기
---

## ✨ 개요

- RecyclerView에서 텍스트가 길 경우 ‘더보기/접기’를 구현하는 기능은 리스트 화면(UI/UX) 최적화에서 매우 자주 등장한다.
- 하지만 TextView의 lineCount는 setText() 이후 레이아웃이 그려져야만 계산 가능하기 때문에 ListAdapter 안에서 2줄 이상인지 판별하는 작업이 까다롭다.
- 이 포스팅에서는 아래의 내용을 포함
  + 2줄 이상일 때만 “더보기” 버튼 표시
  + 누르면 TextView 확장/축소
  + ListAdapter에서도 안정적으로 동작하도록 구성

---

## 1. 구현 핵심 포인트 요약

- TextView는 setText() 직후에는 lineCount 측정 불가 → post{} 안에서 측정
- 최초 한 번만 측정하려면 item에 canExpand 플래그를 둠
  + post 의 값들이 자주 호출될수록 리소스 낭비가 심하고 스크롤 할 때 뷰가 그려지면서 부자연스러운 현상 발생 
- 접힘 상태: maxLines = 1
- 펼침 상태: 원하는 만큼(예: maxLines = 2 or Int.MAX_VALUE)
- 버튼 클릭 시 item의 상태(isExpanded)를 갱신하고 UI를 업데이트

---

## 2. Item 모델 & 레이아웃

### 2-1 Item 모델

```kotlin
data class Item(
    val id: String,
    val text: String,
    var canExpand: Boolean? = null, // 처음 한 번만 계산하기 위한 플래그
    var isExpanded: Boolean = false // 현재 펼침 여부
)
```
- canExpand == null → 아직 lineCount를 측정하지 않은 상태
- 이후 lineCount가 2줄 이상이면 true

### 2-2 Item Layout

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
                                                   xmlns:app="http://schemas.android.com/apk/res-auto"
                                                   android:layout_width="match_parent"
                                                   android:layout_height="wrap_content"
                                                   android:background="@drawable/border"
                                                   android:layout_margin="16dp">

    <TextView
            android:id="@+id/tvContent"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:textSize="14dp"
            android:textColor="@color/white"
            app:layout_constraintTop_toTopOf="parent" />

    <Button
            android:id="@+id/btnMore"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            app:layout_constraintTop_toBottomOf="@id/tvContent"
            android:text="더보기" />

</androidx.constraintlayout.widget.ConstraintLayout>
```

---

## 3. ViewHolder & ListAdapter 에서 더보기/접기 구현

```kotlin
inner class ItemViewHolder(val binding: ItemBinding) : RecyclerView.ViewHolder(binding.root) {
    fun bind(item: Item) {
        binding.tvContent.text = item.text
        binding.tvContent.maxLines = 2

        // 기본은 접힌 상태(1줄)
        binding.tvContent.post {
            if (item.canExpand == null) { // 최초 1회만 호출되도록 설정
                val lineCount = binding.tvContent.layout?.lineCount ?: 0
                binding.btnMore.isVisible = (lineCount >= 2)
                item.canExpand = (lineCount >= 2)
                binding.tvContent.maxLines = if (item.isExpanded) 2 else 1
            }
        }

        if (item.canExpand != null) {
            binding.tvContent.maxLines = if (item.isExpanded) 2 else 1
            binding.btnMore.isVisible = item.canExpand == true
            binding.btnMore.text = if (item.isExpanded) "접기" else "더보기"
        }

        binding.btnMore.setOnClickListener {
            val pos = adapterPosition
            if (pos == RecyclerView.NO_POSITION) return@setOnClickListener

            if (item.isExpanded) {
                item.isExpanded = false
                binding.tvContent.maxLines = 1
                binding.btnMore.text = "더보기"
            } else {
                item.isExpanded = true
                binding.tvContent.maxLines = 2
                binding.btnMore.text = "접기"
            }
        }
    }
}
```

---

## 4. 왜 post{} 안에서 lineCount를 계산해야 할까?

- TextView는 실제로 화면에 배치(layout)되기 전까지 정확한 줄 수가 계산되지 않는다.
- 따라서 아래와 같은 코드는 항상 0 또는 잘못된 값을 반환한다:
- 즉, 레이아웃이 그려진 뒤 실행되는 콜백에서 계산해야 한다.

```kotlin
// 잘못된 방식
binding.tvContent.text = item.text
val lines = binding.tvContent.lineCount // ❌ 잘못된 값

// 올바른 방식
binding.tvContent.post {
    val lines = binding.tvContent.layout?.lineCount ?: 0
}
```

---

## 5. canExpand 플래그가 필요한 이유

- RecyclerView는 ViewHolder를 재사용한다.
- 따라서 스크롤할 때 매번 lineCount를 계산하면:
  + 깜빡임(flickering) 발생
  + 더보기 버튼이 순간적으로 나타났다 사라지는 버그
  + 성능 저하(lineCount는 레이아웃 연산이기 때문)
- 이를 방지하기 위해 해당 item의 텍스트가 2줄 이상인지 최초 1회만 계산하고 Item 객체에 저장하는 방식이 가장 안정적이다.

---

## 6. 더보기/접기 토글 UX 팁

- 펼침 상태에서 maxLines = Int.MAX_VALUE 해도 됨
- 스크롤 이동이 자연스럽지 않다면 NestedScrollView 안에 RecyclerView를 넣지 말 것

---

## 7. 정리

| 기능                | 설명                    |
| ----------------- | --------------------- |
| 2줄 이상 감지          | lineCount로 정확히 판단     |
| 깜빡임 없음            | canExpand 저장으로 재측정 방지 |
| ViewHolder 재활용 OK | stable한 값만 재바인딩       |
| UX 좋은 확장/축소       | 버튼 토글 처리              |
