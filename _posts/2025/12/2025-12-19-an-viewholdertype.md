---
title: (Android/안드로이드) 이종 데이터를 ListAdapter로 보여주는 방법
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) 이종 데이터를 ListAdapter로 보여주는 방법 – ViewType으로 ViewHolder 나누기
---

## 1. 모델 정의

```kotlin
data class People(
    val id: Int,
    val name: String
)

sealed class RowItem {
    data class PeopleRow(val people: People) : RowItem()
    data class ImageRow(val url: String) : RowItem()
}
```

> 이종의 데이터를 처리할 때 sealed class 활용

- UI 모델과 도메인 모델 분리
  - People은 도메인 데이터
  - RowItem은 RecyclerView에 표시되기 위한 UI 모델
  - 즉, “People 리스트”와 “RecyclerView에 어떤 순서로 어떤 타입을 보여줄지” 분리해서 생각

---

## 2. “position=1에 이미지 삽입” 규칙으로 리스트 빌드 함수

- People이 0개면 빈 리스트 반환
- People이 1명만 있어도 “position=1”을 만들려면 index 1에 삽입 가능
- “position=1”은 0-based 기준으로 “두 번째 줄”입니다.

```kotlin
fun initView() {
    val url = "https://www.koreadaily.com/data/photo/2025/12/26/c87c5ca7-6bc1-4176-bdbd-5d8cca642984.jpg"
    val people = mutableListOf<People>()
    (1..20).forEach { people.add(People(it, "name [$it]")) }
    val items = buildItems(people, url)

    val adapter = MultiRowAdapter()
    binding.recyclerView.adapter = adapter
    adapter.submitList(items)
}
```

- 왜 Adapter에서 처리하지 않을까?
  - position 계산 로직이 onBindViewHolder에 섞임
  - DiffUtil 비교가 복잡해짐
  - 다른 화면에서 재사용 불가
- 리스트를 미리 완성된 상태로 만들어서 넘기면:
  - Adapter는 단순해짐
  - 테스트가 쉬워짐
  - 규칙 변경 시 Adapter 수정 불필요

---

## 3. ListAdapter (멀티 ViewHolder)

```kotlin
import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.DiffUtil
import androidx.recyclerview.widget.ListAdapter
import androidx.recyclerview.widget.RecyclerView
import com.bumptech.glide.Glide
import com.example.myapplication.databinding.ItemImageBinding
import com.example.myapplication.databinding.ItemPeopleBinding

class MultiRowAdapter(
    private val onPeopleClick: (People) -> Unit = {}
) : ListAdapter<RowItem, RecyclerView.ViewHolder>(Diff) {

    companion object {
        private const val VT_PEOPLE = 1
        private const val VT_IMAGE = 2
    }

    private object Diff : DiffUtil.ItemCallback<RowItem>() {
        override fun areItemsTheSame(oldItem: RowItem, newItem: RowItem): Boolean {
            return when {
                oldItem is RowItem.PeopleRow && newItem is RowItem.PeopleRow ->
                    oldItem.people.id == newItem.people.id

                oldItem is RowItem.ImageRow && newItem is RowItem.ImageRow ->
                    oldItem.url == newItem.url // url이 바뀌면 다른 아이템

                else -> false
            }
        }

        override fun areContentsTheSame(oldItem: RowItem, newItem: RowItem): Boolean {
            return oldItem == newItem
        }
    }

    override fun getItemViewType(position: Int): Int {
        return when (getItem(position)) {
            is RowItem.PeopleRow -> VT_PEOPLE
            is RowItem.ImageRow -> VT_IMAGE
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
        val inflater = LayoutInflater.from(parent.context)
        return when (viewType) {
            VT_PEOPLE -> PeopleVH(
                ItemPeopleBinding.inflate(inflater, parent, false),
                onPeopleClick
            )

            VT_IMAGE -> ImageVH(
                ItemImageBinding.inflate(inflater, parent, false)
            )

            else -> error("Unknown viewType=$viewType")
        }
    }

    override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
        when (val item = getItem(position)) {
            is RowItem.PeopleRow -> (holder as PeopleVH).bind(item.people)
            is RowItem.ImageRow -> (holder as ImageVH).bind(item.url)
        }
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

    class ImageVH(
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

## 4. Layout

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

## 5. “ViewHolder가 여러 개일 것도 가정” 확장 포인트

- 이 구조는 아이템 타입이 늘어나도 쉽게 확장됩니다.
  - RowItem에 타입 추가
  - VT_XXX 상수 추가
  - getItemViewType / onCreateViewHolder / onBindViewHolder / Diff만 확장

> 예: RowItem.LoadingRow, RowItem.HeaderRow, RowItem.ErrorRow 등

---

## 6. glide 의존성 추가

```groovy
implementation("io.coil-kt:coil:2.6.0")
```