---
title: Android RecyclerView 카드형 좌우 슬라이드 화면 만들기 (ViewPager 효과 구현)
tags: [ Android ]
style: fill
color: dark
description: Android RecyclerView로 좌우 슬라이드 가능한 카드 UI를 구현하는 방법을 소개합니다. SnapHelper와 함께 부드러운 UX를 구성해보세요!
---

## ✨ 개요

`ViewPager` 또는 `ViewPager2`를 대체하여 **더 유연하고 커스터마이징이 쉬운 좌우 스와이프 UI**를 만들고 싶다면?  
**RecyclerView + SnapHelper** 조합을 사용해 **카드형 좌우 슬라이드 뷰**를 만들어보세요.

이 방식은 **캐러셀(Carousel)** 혹은 **카드 스와이프 UI** 구현에 자주 사용됩니다.

---

## 1. ✅ 주요 구성 요소

| 구성 요소           | 설명 |
|--------------------|------|
| RecyclerView       | 슬라이드 가능한 리스트 |
| LinearLayoutManager (HORIZONTAL) | 좌우 스크롤 방향 지정 |
| PagerSnapHelper / LinearSnapHelper | 한 번에 하나씩 정렬되는 스냅 효과 |
| ItemDecoration     | 카드 간 여백 설정 |
| CardView           | 각 아이템을 카드 형태로 표현 |

---

## 2. ✅ 구현 예제

### 2.1 XML 레이아웃

```xml
<androidx.recyclerview.widget.RecyclerView
    android:id="@+id/cardRecyclerView"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:clipToPadding="false"
    android:paddingStart="32dp"
    android:paddingEnd="32dp"
    android:overScrollMode="never" />
```
- clipToPadding="false" 설정으로 카드가 바깥쪽으로 드러나 보이는 효과

### 2.2 Card Item Layout

```xml
<androidx.cardview.widget.CardView
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="300dp"
    android:layout_height="200dp"
    android:layout_margin="8dp"
    android:elevation="4dp"
    android:radius="16dp">

    <TextView
        android:id="@+id/textView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:gravity="center"
        android:textSize="24sp"
        android:text="카드" />
</androidx.cardview.widget.CardView>
```

### 2.3 Adapter

```kotlin
class CardAdapter(private val items: List<String>) :
    RecyclerView.Adapter<CardAdapter.CardViewHolder>() {

    inner class CardViewHolder(val binding: ItemCardBinding) : RecyclerView.ViewHolder(binding.root)

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): CardViewHolder {
        val inflater = LayoutInflater.from(parent.context)
        val binding = ItemCardBinding.inflate(inflater, parent, false)
        return CardViewHolder(binding)
    }

    override fun onBindViewHolder(holder: CardViewHolder, position: Int) {
        holder.binding.textView.text = items[position]
    }

    override fun getItemCount(): Int = items.size
}
```

### 2.4. MainActivity

```kotlin
class MainActivity : AppCompatActivity() {

    private lateinit var binding: ActivityMainBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)

        val items = listOf("🍎 사과", "🍌 바나나", "🍇 포도", "🍓 딸기")
        val adapter = CardAdapter(items)

        binding.cardRecyclerView.layoutManager = LinearLayoutManager(this, LinearLayoutManager.HORIZONTAL, false)
        binding.cardRecyclerView.adapter = adapter

        // 페이지 단위로 스냅 (하나씩 넘어감)
        PagerSnapHelper().attachToRecyclerView(binding.cardRecyclerView)

        // 카드 간 여백 추가
        binding.cardRecyclerView.addItemDecoration(object : RecyclerView.ItemDecoration() {
            override fun getItemOffsets(outRect: Rect, view: View, parent: RecyclerView, state: RecyclerView.State) {
                outRect.right = 24
            }
        })
    }
}
```

---

## 3.🧠 **장점 요약**

| 항목        | RecyclerView + SnapHelper 방식 | ViewPager2 방식           |
| --------- | ---------------------------- | ----------------------- |
| 커스터마이징    | 매우 자유로움 (여백, 애니메이션 등)        | 제한적                     |
| View 재사용성 | ViewHolder 패턴으로 성능 우수        | Fragment 기반이라 메모리 부담 있음 |
| 애니메이션 제어  | 직접 구현 가능                     | 제한적                     |
| 활용 범위     | Carousel, 카드 UI, 뱃지 등 다양한 형태 | 주로 뷰페이저 용도              |

---

## 4.🧠 **결론**

- RecyclerView와 SnapHelper를 조합하면 ViewPager를 대체할 수 있는 더 유연한 좌우 슬라이드 UI를 만들 수 있습니다.
- 커스터마이징이 필요한 캐러셀, 카드형 UI에 특히 유용하며 성능과 UX 모두 향상됩니다.