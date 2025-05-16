---
title: Android RecyclerView 스크롤 최상단/최하단 감지 방법
tags: [Android]
style: fill
color: dark
description: Android RecyclerView에서 스크롤이 최상단 또는 최하단에 도달했는지 감지하는 방법을 예제와 함께 소개합니다.
---

## ✨ 개요

`RecyclerView`는 대용량 리스트를 효율적으로 표현할 수 있는 대표적인 컴포넌트입니다.  
사용자의 스크롤이 **최상단 또는 최하단에 도달했는지 감지**하면 아래와 같은 기능에 응용할 수 있습니다:

- 🔁 새로고침 기능 (`pull to refresh`)
- ⬇️ 무한 스크롤 (infinite scroll)
- 🎯 사용자 경험 향상 (예: 스크롤 안내, 로딩 표시 등)

---

## 1. ✅ 기본 원리

`RecyclerView`는 `addOnScrollListener()`를 사용하여 스크롤 이벤트를 감지할 수 있습니다.  
`LinearLayoutManager`와 함께 사용하여 **스크롤 위치와 아이템 개수**를 기준으로 최하단/최상단 도달 여부를 확인합니다.

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
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import com.js.sample.databinding.ActivityMainBinding

class MainActivity : AppCompatActivity() {

  private lateinit var binding: ActivityMainBinding

  private val TAG: String = "MainActivity"

  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    binding = ActivityMainBinding.inflate(layoutInflater)
    setContentView(binding.root)

    // 샘플 데이터
    val items = (1..50).map { "Item $it" }
    val adapter = SimpleStringAdapter(items)
    val layoutManager = LinearLayoutManager(this)

    binding.recyclerView.layoutManager = layoutManager
    binding.recyclerView.adapter = adapter

    binding.recyclerView.addOnScrollListener(object : RecyclerView.OnScrollListener() {
      override fun onScrolled(rv: RecyclerView, dx: Int, dy: Int) {
        super.onScrolled(rv, dx, dy)

        val totalItemCount = layoutManager.itemCount
        val lastVisibleItem = layoutManager.findLastCompletelyVisibleItemPosition()
        val firstVisibleItem = layoutManager.findFirstCompletelyVisibleItemPosition()

        if (firstVisibleItem == 0) {
          Toast.makeText(this@MainActivity, "🔝 최상단입니다", Toast.LENGTH_SHORT).show()
        }

        if (lastVisibleItem == totalItemCount - 1) {
          Toast.makeText(this@MainActivity, "🔚 최하단입니다", Toast.LENGTH_SHORT).show()
        }
      }
    })
  }
}
```

---

## 3.💡 **팁 & 참고사항**

- findFirstCompletelyVisibleItemPosition() - 첫 번째 완전 노출된 항목 
- findLastCompletelyVisibleItemPosition() - 마지막 완전 노출된 항목 
- RecyclerView.SCROLL_STATE_IDLE - 스크롤이 멈춘 시점 감지 가능

❗ 최하단 감지 시점을 이용해 "무한 스크롤" 구현에도 사용할 수 있습니다.

---

## 4.🧠 **결론**

- RecyclerView의 스크롤 위치를 활용하면 유저 인터랙션의 흐름을 제어하고, 다양한 UX 개선을 구현할 수 있습니다. 
- 최상단/최하단 감지를 통해 새로고침, 다음 페이지 요청, 안내 메시지 출력 등을 적용해보세요! 😊