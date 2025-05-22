---
title: Android RecyclerView Header/Footer 추가하는 가장 쉬운 방법 (ViewType 활용)
tags: [Android]
style: fill
color: dark
description: Android RecyclerView 에서 Header와 Footer를 추가하는 방법을 예제와 함께 정리합니다. ViewType을 활용해 다양한 UI를 구성하는 팁도 포함합니다.
---

## ✨ 개요

Android 앱에서 리스트 상단에 공지, 타이틀, 필터 등을 넣고 싶거나, 하단에 로딩 인디케이터 또는 "더보기" 버튼 등을 붙이고 싶을 때는 **RecyclerView에 Header/Footer 추가** 기능이 필요합니다.

이번 포스팅에서는 **ViewType을 사용해 RecyclerView에 Header/Footer를 추가하는 방법**을 소개합니다.


---

## 1. ✅ ViewType 구조 이해

RecyclerView는 `getItemViewType(position: Int)`를 통해 각 아이템에 **다른 ViewHolder를 매핑할 수 있는 구조**를 제공합니다.  
이 구조를 활용하면 쉽게 Header, Footer를 붙일 수 있습니다.

---

## 2. ✅ 예제

### 2.1 **layout 코드**

- item_header.xml

```xml
<TextView
        android:id="@+id/headerTextView"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:padding="16dp"
        android:gravity="center"
        android:text="📢 헤더: 공지사항 클릭!"
        android:textStyle="bold"
        android:textSize="18sp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
```

- item_footer.xml

```xml
<Button
        android:id="@+id/footerButton"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_margin="16dp"
        android:text="더보기"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
```

- item_my.xml

```xml
<TextView
        android:id="@+id/textView"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:padding="16dp"
        android:textSize="18sp"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintTop_toTopOf="parent" />
```

### 2.2 **Adapter 코드**

```kotlin
data class Item(
    val name: String
)

class MyAdapter(
    private val items: List<Item>,
    val itemClick: ((Int, Item) -> Unit),
    val onHeaderClick: (() -> Unit)? = null,
    val onFooterClick: (() -> Unit)? = null,
) : RecyclerView.Adapter<RecyclerView.ViewHolder>() {

    companion object {
        private const val VIEW_TYPE_HEADER = 0
        private const val VIEW_TYPE_ITEM = 1
        private const val VIEW_TYPE_FOOTER = 2
    }

    override fun getItemCount(): Int = items.size + 2 // +1 header, +1 footer

    override fun getItemViewType(position: Int): Int {
        return when (position) {
            0 -> VIEW_TYPE_HEADER
            itemCount - 1 -> VIEW_TYPE_FOOTER
            else -> VIEW_TYPE_ITEM
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
        val inflater = LayoutInflater.from(parent.context)

        return when (viewType) {
            VIEW_TYPE_HEADER -> {
                val binding =
                    ItemHeaderBinding.inflate(inflater, parent, false)
                HeaderViewHolder(binding)
            }

            VIEW_TYPE_FOOTER -> {
                val binding =
                    ItemFooterBinding.inflate(inflater, parent, false)
                FooterViewHolder(binding)
            }

            else -> {
                val binding =
                    ItemMyBinding.inflate(inflater, parent, false)
                ItemViewHolder(binding)
            }
        }
    }

    inner class HeaderViewHolder(private val binding: ItemHeaderBinding) :
        RecyclerView.ViewHolder(binding.root) {
        fun bind() {
            binding.headerTextView.setOnClickListener {
                onHeaderClick?.invoke()
            }
        }
    }

    inner class FooterViewHolder(private val binding: ItemFooterBinding) :
        RecyclerView.ViewHolder(binding.root) {
        fun bind() {
            binding.footerButton.setOnClickListener {
                onFooterClick?.invoke()
            }
        }
    }

    inner class ItemViewHolder(private val binding: ItemMyBinding) :
        RecyclerView.ViewHolder(binding.root) {

        fun bind(item: Item) {
            binding.textView.text = item.name

            binding.root.setOnClickListener {
                itemClick.invoke(adapterPosition-1, item)
            }
        }
    }

    override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
        when (holder) {
            is HeaderViewHolder -> holder.bind()
            is FooterViewHolder -> holder.bind()
            is ItemViewHolder -> {
                val item = items[position - 1] // offset for header
                holder.bind(item)
            }
        }
    }
}
```

### 2.3 **Adapter 호출 코드**

```kotlin
val list = mutableListOf<Item>()
repeat(100) {
    list.add(Item("$it name"))
}

val adapter = MyAdapter(list,
    itemClick = { position, item ->
        Log.e(TAG, "itemClick $position $item")
    } ,
    onHeaderClick = {
        Log.e(TAG, "onHeaderClick")
    },
    onFooterClick = {
        Log.e(TAG, "onFooterClick")
    }
)
binding.recyclerView.adapter = adapter
```


---

## 3. 🔍 정리

| 항목               | 설명                               |
| ---------------- | -------------------------------- |
| ViewType 활용      | 위치에 따라 다른 레이아웃을 처리 가능            |
| Header/Footer 추가 | 별도 XML + ViewHolder 구성 필요        |
| 데이터 오프셋 처리       | 실제 데이터는 position-1부터 시작          |
| 확장성              | 광고, 추천 항목, 로딩 인디케이터 등 다양하게 활용 가능 |

---

## 4.🧠 **결론**

- getItemViewType()과 RecyclerView.ViewHolder 구조를 이해하면 복잡한 UI도 단순하게 관리할 수 있습니다. 
- Header/Footer 외에도 배너, 공지, 분류 그룹 등 다양한 형태의 다중 아이템 처리가 가능합니다.
- 실무에서도 가장 많이 쓰이는 구조이니 꼭 익혀두세요! 😊