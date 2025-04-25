---
title: Android RecyclerView DiffUtil로 효율적인 리스트 갱신하기 (ListAdapter 활용)
tags: [Android]
style: fill
color: dark
description: Android notifyDataSetChanged()를 대체하는 RecyclerView 효율적 갱신 방법, DiffUtil + ListAdapter 사용법 정리
---
---

## ✨ 개요

Android 앱에서 `RecyclerView`는 리스트 UI를 구현할 때 가장 많이 사용되는 컴포넌트입니다.  
하지만 리스트 데이터가 변경될 때마다 `notifyDataSetChanged()`를 호출하면 **비효율적이고 부드럽지 못한 UI**가 발생할 수 있습니다.

이를 해결하기 위한 강력한 도구가 바로 **`DiffUtil` + `ListAdapter`** 입니다.  
이번 글에서는 이 두 가지를 활용하여 **최소한의 변경으로 최대한의 성능을 얻는 방법**을 소개합니다.

---

## 1. notifyDataSetChanged()
```kotlin
adapter.notifyDataSetChanged()
```
- 모든 아이템을 무조건 새로 그리게 됩니다.
- 변경된 항목이 1개여도 전체 아이템이 갱신됩니다.
- RecyclerView 애니메이션 및 성능 저하

## 2. ✅ 해결책: DiffUtil + ListAdapter
🔹 DiffUtil: 두 리스트 간의 차이 계산
- 변경된 항목만 찾아서 효율적으로 업데이트
- areItemsTheSame / areContentsTheSame 정의 필요

🔹 ListAdapter: DiffUtil 내장 RecyclerView Adapter
- submitList()로 새로운 리스트 전달 시 자동 비교 및 애니메이션 처리
- ViewHolder, onCreateViewHolder(), onBindViewHolder()는 기존과 동일하게 구현

### 🧩 사용 예제

- 데이터 클래스 정의 

```kotlin
data class User(val id: Int, val name: String)
```

- DiffUtil.ItemCallback 구현

```kotlin
import androidx.recyclerview.widget.DiffUtil

class UserDiffCallback : DiffUtil.ItemCallback<User>() {
  override fun areItemsTheSame(oldItem: User, newItem: User): Boolean {
    return oldItem.id == newItem.id // 고유 식별자로 비교
  }

  override fun areContentsTheSame(oldItem: User, newItem: User): Boolean {
    return oldItem == newItem // 전체 내용 비교
  }
}
```

- ListAdapter 생성

```kotlin
import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.ListAdapter
import androidx.recyclerview.widget.RecyclerView
import com.example.sample.databinding.ItemUserBinding

class UserAdapter : ListAdapter<User, UserAdapter.UserViewHolder>(UserDiffCallback()) {

  inner class UserViewHolder(private val binding: ItemUserBinding) :
    RecyclerView.ViewHolder(binding.root) {
    fun bind(item: User) {
      binding.textViewName.text = item.name
    }
  }

  override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): UserViewHolder {
    val binding = ItemUserBinding.inflate(LayoutInflater.from(parent.context), parent, false)
    return UserViewHolder(binding)
  }

  override fun onBindViewHolder(holder: UserViewHolder, position: Int) {
    holder.bind(getItem(position))
  }
}
```

- submitList()로 데이터 갱신

```kotlin
val adapter = UserAdapter()
recyclerView.adapter = adapter

val users = listOf(User(1, "철수"), User(2, "영희"))
adapter.submitList(users) // 자동 비교 후 필요한 아이템만 갱신됨
```

---

## 3. 차이점 비교

| 항목             | DiffUtil + ListAdapter              | notifyDataSetChanged()               |
|------------------|-------------------------------------|--------------------------------------|
| **갱신 효율**     | 변경된 항목만 자동 갱신             | 전체 항목 강제 갱신                  |
| **성능 및 애니메이션** | 자연스럽고 부드러운 UI 애니메이션 처리 | 깜빡임, 애니메이션 없음             |
| **코드 재사용성** | ViewHolder 로직 그대로 사용 가능     | 동일                                 |
| **유지보수성**    | 수정/비교 기준 명확하게 설정 가능   | 단순하지만 유연성 떨어짐            |


---

## 4. 결론
RecyclerView의 데이터가 자주 변경되는 앱이라면 notifyDataSetChanged() 대신 반드시 DiffUtil + ListAdapter를 사용하세요.
- UI 성능 향상
- 애니메이션 자동 처리
- 코드 구조 명확화

이제 여러분의 RecyclerView도 효율적으로 바꿔보세요! 😊