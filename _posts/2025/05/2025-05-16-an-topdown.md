---
title: Android RecyclerView 최상단/최하단으로 스크롤 이동하는 방법
tags: [Android]
style: fill
color: dark
description: Android RecyclerView 에서 사용자가 버튼 클릭 시 최상단 또는 최하단으로 스크롤 이동하는 방법을 실습 코드와 함께 설명합니다.
---

## ✨ 개요

`RecyclerView`는 스크롤 가능한 리스트 UI를 제공하지만, **특정 위치로 이동**해야 하는 경우도 많습니다.

- 🔝 맨 위로 이동 (예: 새로고침 버튼)
- 🔚 맨 아래로 이동 (예: 채팅 앱에서 최신 메시지로 이동)

이 포스팅에서는 **최상단/최하단으로 부드럽게 스크롤 이동하는 방법**을 소개합니다.

---

## 1. ✅ 기본 원리

- `scrollToPosition(position: Int)`  
  → 즉시 해당 위치로 이동 (애니메이션 없음)

- `smoothScrollToPosition(position: Int)`  
  → 애니메이션과 함께 자연스럽게 스크롤 이동

## 2. ✅ 전체 예제 코드

### 2.1 레이아웃 파일

```xml
<androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:padding="16dp" />
```
- 버튼에 따라 텍스트 스타일 변경

### 2.2  SimpleStringAdapter 코드

```kotlin
import android.view.ViewGroup
import android.widget.TextView
import androidx.recyclerview.widget.RecyclerView

class SimpleStringAdapter(private val items: List<String>) :
  RecyclerView.Adapter<SimpleStringAdapter.ViewHolder>() {

  inner class ViewHolder(private val view: TextView) : RecyclerView.ViewHolder(view) {
    fun bind(item: String) {
      view.text = item
    }
  }

  override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ViewHolder {
    val textView = TextView(parent.context).apply {
      setPadding(24, 32, 24, 32)
      textSize = 18f
    }
    return ViewHolder(textView)
  }

  override fun onBindViewHolder(holder: ViewHolder, position: Int) {
    holder.bind(items[position])
  }

  override fun getItemCount(): Int = items.size
}

```

### 2.3 MainActivity 코드

```kotlin
import android.os.Bundle
import androidx.appcompat.app.AppCompatActivity
import androidx.recyclerview.widget.LinearLayoutManager
import com.js.sample.databinding.ActivityMainBinding

class MainActivity : AppCompatActivity() {

    private lateinit var binding: ActivityMainBinding

    private val TAG: String = "MainActivity"

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        // 샘플 데이터
        val items = (1..100).map { "Item $it" }
        val adapter = SimpleStringAdapter(items)
        val layoutManager = LinearLayoutManager(this)

        with(binding) {

            recyclerView.layoutManager = layoutManager
            recyclerView.adapter = adapter

            topButton.setOnClickListener {
                recyclerView.smoothScrollToPosition(0)
            }

            downButton.setOnClickListener {
                recyclerView.smoothScrollToPosition(items.size - 1)
            }
        }

    }
}
```

---

## 3.💡 **팁 & 참고사항**

- 최상단 즉시 이동 - scrollToPosition(0) 순간 이동, 애니메이션 없음
- 최상단 부드럽게 이동 - smoothScrollToPosition(0) 애니메이션 포함
- 최하단으로 이동 - scrollToPosition(items.size - 1) 또는 smoothScrollToPosition() 리스트의 마지막 인덱스로 이동

> 📌 참고: 데이터 수가 매우 많을 경우 scrollToPosition()이 더 적합할 수 있습니다.

---

## 4.🧠 **결론**

- RecyclerView에서 위치 이동은 scrollToPosition() 또는 smoothScrollToPosition()으로 간단하게 구현 가능합니다.
- 버튼이나 특정 이벤트에 따라 사용자 편의성을 높일 수 있는 UX 요소로 적극 활용해보세요!