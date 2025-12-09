---
title: (Android/안드로이드) ListAdapter submitList 갱신 안되는 문제 해결 방법 
tags: [Android,Kotlin]
style: fill
color: dark
description: (Android/안드로이드) ListAdapter 갱신 안됨 문제 해결 — DiffUtil과 submitList()에서 객체 값을 변경해도 업데이트가 되지 않는 이유
---

## ✨ 개요

- RecyclerView + ListAdapter + DiffUtil 조합을 사용할 때 가장 많이 겪는 문제 중 하나는 다음과 같은 상황이다.
  + ListAdapter에서 submitList()를 호출했는데 화면이 갱신되지 않는다.
  + Item의 내부값을 변경했는데 UI 반영이 안 된다.
  + 아이템 객체의 필드만 수정했는데 DiffUtil이 변화를 못 잡는다.
- 이 문제는 DiffUtil은 객체 동일성을 체크하는 구조이기 때문이며, 해결 방법은 기존 객체를 수정하는 것이 아니라 새로운 객체(copy)를 만들어 리스트에 넣는 것이다.

---

## 1.  요약

- ListAdapter는 DiffUtil을 이용해 리스트의 변경점만 갱신함
- DiffUtil은 참조가 같으면 같은 객체로 판단하는 경향이 있음
- 기존 리스트 아이템의 내부 값을 수정해도 DiffUtil은 변경을 감지하지 못함
- 해결법: immutable 구조 + data class copy로 새로운 객체 생성 후 submitList()

---

## 2. 왜 ListAdapter는 값이 변경되어도 갱신되지 않을까?

### 2-1 Item 모델

```kotlin
data class Item(
    val id: String,
    val text: String,
    var isExpanded: Boolean = false // 현재 펼침 여부
)
```

### 2-2 문제 상황 예시

```kotlin
item.isExpanded = true   // 내부 값 변경
submitList(currentList)  // ★ UI 갱신 안 됨
```
- 겉보기에 item의 값이 바뀌었으니 DiffUtil이 업데이트를 해야 할 것 같지만 실제로는 갱신되지 않는다.
- **그 이유는 ListAdapter의 구조 때문이다.**

---

## 3. ListAdapter & DiffUtil이 목록을 비교하는 방식

ListAdapter에 DiffUtil이 내부적으로 다음과 같은 비교를 수행한다.

```kotlin
object : DiffUtil.ItemCallback<Item>() {
    override fun areItemsTheSame(
        oldItem: Item,
        newItem: Item
    ): Boolean {
        return oldItem.id == newItem.id
    }

    override fun areContentsTheSame(
        oldItem: Item,
        newItem: Item
    ): Boolean {
        return oldItem == newItem
    }

}
```
- areItemsTheSame(oldItem, newItem) → 객체가 같은 항목인가? (id 비교)
- areContentsTheSame(oldItem, newItem) → 내용이 같은가? (equals 비교)
- 문제의 핵심 🔥
  + submitList()에 전달한 리스트 안의 객체가 **기존 객체와 동일한 객체(= 동일 주소)**이면 DiffUtil은 아래처럼 판단한다.

```kotlin
oldItem === newItem  → true
oldItem == newItem  → true
```

- 즉, 같은 객체니까 내용도 같겠지? → 변경 없음 처리되므로 UI는 갱신되지 않는다.
- DiffUtil은 객체의 내부 필드만 변경되는 것을 감지할 수 없다.

---

## 4. 그래서 immutable(불변) 리스트 + data class copy를 써야 한다

- ListAdapter는 변경된 내용을 반영하려면 새로운 객체가 필요하다.
- 잘못된 방식 (mutable 객체 직접 수정)

### 4-1. 잘못된 방식

```kotlin
currentList[position].isExpanded = true
submitList(currentList)  // 갱신 안 됨
```

### 4-2. 올바른 방식 (copy로 새로운 객체 생성)

```kotlin
val newList = currentList.toMutableList() // 새로운 리스트 필요
val oldItem = currentList[position]

newList[position] = oldItem.copy(
    isExpanded = !oldItem.isExpanded
)

submitList(newList)
```

---

## 5. 왜 copy()가 중요한가? (참조 변경)

- data class의 copy는 내부 값만 변경하는 것이 아니라 새로운 객체(new reference)를 생성한다.
- 덕분에 DiffUtil은 이 항목은 변경되었군이라고 정확히 감지한다.
- ```kotlin
  oldItem !== newItem   // 주소가 다름  
  oldItem != newItem    // equals 결과에도 차이 발생
  ```

---

## 6. ListAdapter에서 불변 객체를 강제하는 이유

- ListAdapter 설계 철학
  + 내부 리스트는 immutable
  + 외부에서 mutable하게 리스트를 수정하는 행위는 금지
  + submitList()를 호출할 때마다 새로운 리스트를 전달해야 함
- 따라서 항목을 변경하면 안 되고, 항상 새로운 항목을 만들어야 한다.
- 안드로이드 공식 문서에서도 명시한다:
  + DiffUtil은 mutable 객체 변경을 감지할 수 없다
  + 반드시 새로운 리스트와 새로운 객체를 전달해야 한다

---

## 7. 정리

| 문제                 | 원인                    | 해결책                 |
| ------------------ | --------------------- | ------------------- |
| 값 변경해도 갱신 안 됨      | DiffUtil이 변경을 감지하지 못함 | copy()로 새로운 객체 생성   |
| mutable 객체 수정      | 참조(address)가 동일함      | immutable + 새 리스트   |
| submitList 후에도 무반응 | 내부적으로 동일한 객체로 간주      | 새로운 객체 + 새로운 리스트 전달 |

---

## 8. Best Practice (실무 팁)

- data class 사용
  + equals 비교가 자동 구현됨
- 항목 업데이트는 항상 copy()
  + 내부 필드 수정은 금지
- submitList()에는 항상 새로운 리스트 전달
  + 기존 리스트를 그대로 넘기면 DiffUtil이 무시함
- 리스트 변경이 잦다면 state-holder 패턴 사용
  + ViewModel에서 불변 리스트를 관리하고 UI단은 observe로 반영