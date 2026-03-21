---
title: (Kotlin/코틀린) data class copy 완전 정리 - 불변 객체, 얕은 복사, 실무 패턴까지
tags: [ Android,Kotlin ]
style: fill
color: dark
description: (Kotlin/코틀린) data class의 copy() 함수 동작 원리와 얕은 복사 특징, 불변 객체 설계, RecyclerView/ListAdapter와의 관계, 실무에서 자주 사용하는 패턴까지 정리합니다.
---

## 개요

- Kotlin data class를 사용하면 자동으로 제공되는 함수 중 하나가 copy()입니다.
- ```kotlin
  data class User(
    val name: String,
    val age: Int
  )

  val user1 = User("Kim", 20)
  val user2 = user1.copy(age = 21)
  ```
- 이처럼 특정 값만 변경한 새로운 객체를 만들 수 있습니다.
- 하지만 실무에서는 다음과 같은 오해/문제가 자주 발생합니다.
  - copy는 깊은 복사인가?
  - 내부 객체도 복사되나?
  - ListAdapter에서 왜 중요하지?
  - 왜 immutable 패턴과 같이 써야 하나?

---

## 1. copy()란 무엇인가?

`copy()`는 기존 객체를 기반으로 새로운 객체를 생성하는 함수입니다.

```kotlin
val newUser = oldUser.copy(age = 30)

// 런타임 시
fun copy(name: String = this.name, age: Int = this.age): User {
    return User(name, age)
}
```

- 기존 객체는 변경되지 않음
- 새로운 객체 생성
- 일부 값만 변경 가능

---

## 2. copy()는 얕은 복사 (Shallow Copy)

```kotlin
data class Profile(
    val name: String,
    val hobbies: MutableList<String>
)

val p1 = Profile("Kim", mutableListOf("Game"))
val p2 = p1.copy()

// 같은 리스트 참조
p1.hobbies === p2.hobbies  // true

// 문제 상황 (원본도 같이 변경 됨)
p2.hobbies.add("Music")
println(p1.hobbies) // [Game, Music]
```

- copy()는 “객체 자체만 새로 만들고 내부 참조는 그대로 유지”한다.

---

### 3. 깊은 복사가 필요한 경우

```kotlin
val p2 = p1.copy(
    hobbies = p1.hobbies.toMutableList()
)

val p2 = Profile(
    name = p1.name,
    hobbies = p1.hobbies.toMutableList()
)
```

- 👉 내부까지 복사하려면 직접 처리해야 함

---

## 4. 왜 copy()는 immutable과 같이 쓰나?

copy는 불변 객체(immutable) 패턴에서 가장 강력합니다.

```kotlin
val newUser = user.copy(age = 30)
```

- 상태 변경이 명확
- 이전 상태 유지 가능
- Diff 비교 정확

---

## 5. ListAdapter / DiffUtil에서 중요한 이유

ListAdapter는 객체 변경 여부를 기준으로 UI를 갱신합니다.

```kotlin
// ❌ 잘못된 방식
user.age = 30
adapter.notifyItemChanged(position)

// ✅ 올바른 방식
val newList = currentList.map {
    if (it.id == user.id) it.copy(age = 30)
    else it
}

adapter.submitList(newList)
```

- 새로운 객체 생성 → DiffUtil 정확하게 동작

---

## 6. default parameter 활용

copy는 default parameter 기반이라 매우 유연합니다.

```kotlin
val user = User("Kim", 20)

val u1 = user.copy()
val u2 = user.copy(name = "Lee")
val u3 = user.copy(age = 40)
```

---

## 7. copy()와 성능

- copy는 새로운 객체를 생성합니다.
  - GC 증가 가능
  - 대량 리스트에서 비용 발생
- 하지만 대부분의 경우:
  - 👉 가독성과 안정성이 더 중요

---

## 8. 실무에서 자주 쓰는 패턴

```kotlin
// 상태 업데이트
_uiState.value = _uiState.value.copy(
    isLoading = true
)

// 선택 상태 변경
val updated = list.map {
    it.copy(isSelected = it.id == selectedId)
}

// 부분 업데이트
val newUser = oldUser.copy(
    name = inputName
)

// reducer 패턴
state = state.copy(
    items = newItems,
    isLoading = false
)
```