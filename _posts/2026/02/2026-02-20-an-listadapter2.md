---
title: (Android/안드로이드) ListAdapter 사용 시 반드시 알아야 할 주의사항 정리
tags: [ Android,Kotlin ]
style: fill
color: dark
description: (Android/안드로이드) RecyclerView ListAdapter 사용 시 발생하는 데이터 갱신 오류, DiffUtil 오작동, mutable list 문제, submitList 동작 원리, payload 활용, 성능 최적화까지 실무 체크포인트 정리
---

## 개요

- ListAdapter는 DiffUtil을 내부에 포함한 RecyclerView 어댑터입니다.
- 하지만 submitList를 잘못 쓰면 화면이 안 바뀌거나, UI가 꼬이거나, 성능이 급격히 떨어질 수 있습니다.

---

## 1. 구조

- ```text
  submitList(newList)
      ↓
  AsyncDiffUtil (background thread)
      ↓
  calculateDiff()
      ↓ 
  dispatchUpdatesTo()
      ↓
  RecyclerView 갱신
  ```
- 👉 핵심: 기존 리스트와 새 리스트를 비교해서 변경 부분만 갱신

---

## 2. ❗ 가장 많이 터지는 실수: MutableList 재사용

### ❌ 잘못된 코드

```kotlin
val list = currentList
list.add(newItem)
adapter.submitList(list)
```

- `currentList`와 `list`가 같은 참조
- DiffUtil은 "변경 없음"으로 판단
- UI 갱신 안 됨

### ✅ 올바른 패턴

```kotlin
val newList = currentList.toMutableList().apply {
    add(newItem)
}
adapter.submitList(newList)
```

- 👉 항상 새로운 인스턴스를 submit 해야 함.

---

### 3. areItemsTheSame vs areContentsTheSame 차이

```kotlin
object DiffCallback : DiffUtil.ItemCallback<Item>() {

    override fun areItemsTheSame(oldItem: Item, newItem: Item): Boolean {
        return oldItem.id == newItem.id
    }

    override fun areContentsTheSame(oldItem: Item, newItem: Item): Boolean {
        return oldItem == newItem
    }
}
```

| 메서드                | 역할                     |
| ------------------ | ---------------------- |
| areItemsTheSame    | 같은 아이템인가? (식별자 비교)     |
| areContentsTheSame | 내용이 같은가? (UI 갱신 필요 여부) |

---

## 4. data class equals() 의존 문제

- areContentsTheSame에서 oldItem == newItem을 쓰면
- data class의 equals에 의존합니다.

```kotlin
// 문제 상황
data class Item(
    val id: Long,
    val name: String,
    var isSelected: Boolean
)
```

- isSelected를 변경했는데 UI가 안 바뀐다면?
  - 기존 객체를 mutate한 경우
  - equals 결과가 동일하다고 판단될 수 있음
  - 👉 모델은 가급적 immutable로 설계

---

## 5. submitList() 연속 호출 시 동작 주의

```kotlin
adapter.submitList(list1)
adapter.submitList(list2)
```

- Diff는 비동기로 계산됩니다.
  - 이전 Diff 계산이 끝나기 전에 새 submit 호출 가능
  - 중간 상태가 무시될 수 있음

---

## 6. currentList 직접 수정 금지

```kotlin
adapter.currentList[0].name = "new"
```

- ListAdapter는 immutable 가정
- 내부 상태 꼬임
- Diff 오작동

---

## 7. payload 활용 (부분 갱신 최적화)

DiffUtil에서 payload를 반환하면 불필요한 전체 bind를 줄일 수 있습니다.

```kotlin
override fun getChangePayload(oldItem: Item, newItem: Item): Any? {
    return if (oldItem.isSelected != newItem.isSelected) {
        "SELECT_CHANGED"
    } else null
}

override fun onBindViewHolder(
    holder: ViewHolder,
    position: Int,
    payloads: MutableList<Any>
) {
    if (payloads.isNotEmpty()) {
        holder.updateSelection(payloads[0])
    } else {
        super.onBindViewHolder(holder, position, payloads)
    }
}
```

---

## 8. 대용량 리스트 성능 이슈

- DiffUtil은 O(N)~O(N²) 비용 발생 가능.
  - 5,000개 이상 리스트
  - 자주 submitList 호출
  - → 프레임 드랍 가능
- 대응 전략
  - Paging3 사용
  - 데이터 변경 최소화
  - payload 적극 활용