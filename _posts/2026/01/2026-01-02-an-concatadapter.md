---
title: (Android/안드로이드) RecyclerView ConcatAdapter로 이종 데이터 합쳐 보여주기
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) RecyclerView ConcatAdapter로 이종 데이터 합쳐 보여주기
---

## 1. 왜 ConcatAdapter인가?

- ListAdapter + multiple viewType은 “한 Adapter 안에 모든 타입을 처리”합니다.
- 반면 ConcatAdapter는 각 섹션을 독립 Adapter로 분리해서 결합합니다.

- ConcatAdapter가 유리한 경우
  - 헤더/배너/푸터/로딩/에러 등 섹션이 명확함
  - 각 섹션을 독립적으로 테스트/교체/재사용하고 싶음
  - 화면이 커지고 아이템 타입이 많아질 예정

---

## 2. 모델

```kotlin
data class People(
    val id: Int,
    val name: String
)
```

---

## 3. Adapter 구성 (3개로 분리)

- PeopleAdapter : People 리스트 표시 (ListAdapter)
- BannerAdapter : 이미지 1개 표시 (단일 아이템 Adapter)
- ConcatAdapter : People(첫 1개) + Banner + People(나머지) 결합

---

## 4. PeopleAdapter (ListAdapter)

```kotlin
import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.DiffUtil
import androidx.recyclerview.widget.ListAdapter
import androidx.recyclerview.widget.RecyclerView
import com.example.myapplication.databinding.ItemPeopleBinding

class PeopleAdapter(
    private val onPeopleClick: (People) -> Unit = {}
) : ListAdapter<People, PeopleAdapter.PeopleVH>(Diff) {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): PeopleVH {
        val inflater = LayoutInflater.from(parent.context)
        return PeopleVH(ItemPeopleBinding.inflate(inflater, parent, false), onPeopleClick)
    }

    override fun onBindViewHolder(holder: PeopleVH, position: Int) {
        holder.bind(getItem(position))
    }

    class PeopleVH(
        private val binding: ItemPeopleBinding,
        private val onClick: (People) -> Unit
    ) : RecyclerView.ViewHolder(binding.root) {

        fun bind(item: People) = with(binding) {
            tvName.text = item.name
            root.setOnClickListener { onClick(item) }
        }
    }

    private object Diff : DiffUtil.ItemCallback<People>() {
        override fun areItemsTheSame(oldItem: People, newItem: People): Boolean =
            oldItem.id == newItem.id

        override fun areContentsTheSame(oldItem: People, newItem: People): Boolean =
            oldItem == newItem
    }
}
```
---

## 5. BannerAdapter(단일 아이템 Adapter)

```kotlin
import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.RecyclerView
import com.bumptech.glide.Glide
import com.example.myapplication.databinding.ItemImageBinding

class BannerAdapter : RecyclerView.Adapter<BannerAdapter.BannerVH>() {

    private var url: String? = null

    fun submitUrl(newUrl: String?) {
        url = newUrl
        notifyDataSetChanged() // 단일 한개라 성능 영향도 낮음
    }

    override fun getItemCount(): Int = if (url.isNullOrBlank()) 0 else 1

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): BannerVH {
        val inflater = LayoutInflater.from(parent.context)
        return BannerVH(ItemImageBinding.inflate(inflater, parent, false))
    }

    override fun onBindViewHolder(holder: BannerVH, position: Int) {
        holder.bind(requireNotNull(url))
    }

    class BannerVH(
        private val binding: ItemImageBinding
    ) : RecyclerView.ViewHolder(binding.root) {

        fun bind(url: String) {
            Glide.with(binding.root.context)
                .load(url)
                .into(binding.ivBanner)
        }
    }
}
```

---

## 6. “position=1에 배너”를 만드는 ConcatAdapter 구성

- 핵심은 People 리스트를 두 덩어리로 쪼개서 ConcatAdapter에 붙이는 것입니다.
  - topPeopleAdapter : People 1개(첫 번째 줄)
  - bannerAdapter : 배너 1개(두 번째 줄)
  - restPeopleAdapter : 나머지 People

```kotlin
 fun initView() {
    val url = "https://www.koreadaily.com/data/photo/2025/12/26/c87c5ca7-6bc1-4176-bdbd-5d8cca642984.jpg"
    val bannerAdapter = BannerAdapter(url) // 1번째
    val topPeopleAdapter = PeopleAdapter() // 0번째
    val restPeopleAdapter = PeopleAdapter() // 2번째 이상

    val people = mutableListOf<People>()
    (0..20).forEach { people.add(People(it, "name [$it]")) }
    
    topPeopleAdapter.submitList(listOf(people.first()))
    restPeopleAdapter.submitList(people.subList(1, people.size))

    binding.recyclerView.adapter = ConcatAdapter(
        ConcatAdapter.Config.Builder()
            // 뷰타입 충돌 회피 옵션 (권장)
            .setIsolateViewTypes(true)
            .build(),
        topPeopleAdapter, // position 0
        bannerAdapter,               // position 1
        restPeopleAdapter            // position >= 2
    )
}
```

---

## 7. Layout

### item_people.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="16dp">

    <TextView
        android:id="@+id/tvName"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:textSize="16sp"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent"
        app:layout_constraintTop_toTopOf="parent"/>
</androidx.constraintlayout.widget.ConstraintLayout>
```

### image_people.xml

```xml
<?xml version="1.0" encoding="utf-8"?>
<FrameLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:padding="16dp">

    <ImageView
        android:id="@+id/ivBanner"
        android:layout_width="match_parent"
        android:layout_height="160dp"
        android:scaleType="centerCrop" />
</FrameLayout>

```

---

## 8. 의존성 추가

```groovy
// gradle
implementation("io.coil-kt:coil:2.6.0")

// ConcatAdapter (1.3 이상부터 안정성 높으므로 권장)
implementation("androidx.recyclerview:recyclerview:1.3.2")
```