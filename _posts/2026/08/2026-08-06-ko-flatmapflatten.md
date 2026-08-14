---
title: (Kotlin/코틀린) flatMap vs flatten 비교
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 flatMap과 flatten 함수의 동작 원리와 차이점, map+flatten 패턴, 실전 활용 예제를 정리합니다.
---

---

## 1. flatten — 중첩 컬렉션 펼치기

`List<List<T>>`를 `List<T>`로 한 단계 펼칩니다.

```kotlin
val nested = listOf(
    listOf(1, 2, 3),
    listOf(4, 5),
    listOf(6, 7, 8, 9)
)

val flat = nested.flatten()
println(flat)  // [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

```
flatten 동작:
[[1,2,3], [4,5], [6,7,8,9]]
       ↓
[1, 2, 3, 4, 5, 6, 7, 8, 9]
```

한 단계만 펼치므로 3중 중첩은 두 번 호출해야 합니다.

```kotlin
val tripleNested = listOf(listOf(listOf(1, 2), listOf(3)), listOf(listOf(4)))
println(tripleNested.flatten())         // [[1, 2], [3], [4]]  ← 한 단계만
println(tripleNested.flatten().flatten()) // [1, 2, 3, 4]      ← 두 번
```

---

## 2. flatMap — map + flatten

각 원소를 **컬렉션으로 변환(map)한 뒤 펼칩니다(flatten)**.

```kotlin
val words = listOf("Hello Kotlin", "World", "Flat Map")

// 단어를 문자 리스트로 변환 후 펼치기
val chars = words.flatMap { it.toList() }
println(chars)
// [H, e, l, l, o,  , K, o, t, l, i, n, W, o, r, l, d, F, l, a, t,  , M, a, p]

// 문자열을 단어로 분리 후 펼치기
val allWords = words.flatMap { it.split(" ") }
println(allWords)  // [Hello, Kotlin, World, Flat, Map]
```

```
flatMap 동작:
["Hello Kotlin", "World"]
    ↓ map (각 문자열을 단어 리스트로)
[["Hello", "Kotlin"], ["World"]]
    ↓ flatten
["Hello", "Kotlin", "World"]
```

---

## 3. map vs flatMap

```kotlin
val sentences = listOf("사과 바나나", "딸기 포도 수박")

// map: 각 원소를 List로 변환 → List<List<String>> 반환
val mapped = sentences.map { it.split(" ") }
println(mapped)  // [[사과, 바나나], [딸기, 포도, 수박]]

// flatMap: 변환 후 펼침 → List<String> 반환
val flatMapped = sentences.flatMap { it.split(" ") }
println(flatMapped)  // [사과, 바나나, 딸기, 포도, 수박]
```

| 함수 | 반환 타입 | 중첩 제거 |
|------|----------|----------|
| `map { listOf(...) }` | `List<List<T>>` | ✗ |
| `flatMap { listOf(...) }` | `List<T>` | ✓ |
| `flatten()` | `List<T>` | ✓ (이미 중첩된 경우) |

---

## 4. 실전 패턴 — 카테고리별 태그 수집

```kotlin
data class Post(val title: String, val tags: List<String>)

val posts = listOf(
    Post("Kotlin 입문", listOf("kotlin", "programming", "android")),
    Post("Flow 완전 정리", listOf("kotlin", "coroutines", "flow")),
    Post("Compose 시작하기", listOf("android", "compose", "ui"))
)

// 모든 태그 목록 (중복 포함)
val allTags = posts.flatMap { it.tags }
println(allTags)
// [kotlin, programming, android, kotlin, coroutines, flow, android, compose, ui]

// 중복 제거 후 정렬
val uniqueTags = posts.flatMap { it.tags }.distinct().sorted()
println(uniqueTags)
// [android, compose, coroutines, flow, kotlin, programming, ui]

// 태그별 포스트 수
val tagCount = posts.flatMap { post -> post.tags.map { tag -> tag to post.title } }
    .groupBy({ it.first }, { it.second })
    .mapValues { it.value.size }
// {kotlin=2, programming=1, android=2, ...}
```

---

## 5. 실전 패턴 — 권한 목록 병합

```kotlin
data class Role(val name: String, val permissions: List<String>)

val userRoles = listOf(
    Role("EDITOR", listOf("read", "write", "comment")),
    Role("REVIEWER", listOf("read", "comment", "approve"))
)

// 사용자가 가진 전체 권한 (중복 제거)
val userPermissions = userRoles
    .flatMap { it.permissions }
    .distinct()

println(userPermissions)
// [read, write, comment, approve]

fun hasPermission(permission: String) = permission in userPermissions
println(hasPermission("write"))   // true
println(hasPermission("delete"))  // false
```

---

## 6. 실전 패턴 — 페이징 결과 병합

```kotlin
// 여러 페이지 결과를 하나로 합치는 패턴
suspend fun fetchAllUsers(totalPages: Int): List<User> {
    return (1..totalPages)
        .map { page -> api.fetchUsers(page) }  // List<List<User>>
        .flatten()                              // List<User>
}

// 또는 flatMap으로
suspend fun fetchAllUsers2(totalPages: Int): List<User> {
    return (1..totalPages).flatMap { page -> api.fetchUsers(page) }
}
```

---

## 7. flatMap과 null 처리

```kotlin
val values = listOf("1", "abc", "3", "def", "5")

// mapNotNull: null을 걸러내는 map
val parsed = values.mapNotNull { it.toIntOrNull() }
println(parsed)  // [1, 3, 5]

// flatMap으로 동일한 효과
val parsed2 = values.flatMap { s ->
    listOfNotNull(s.toIntOrNull())
}
println(parsed2)  // [1, 3, 5]
```

---

## 8. flatMapIndexed

인덱스가 필요한 flatMap입니다.

```kotlin
val groups = listOf(listOf("a", "b"), listOf("c", "d", "e"))

val indexed = groups.flatMapIndexed { groupIndex, group ->
    group.mapIndexed { itemIndex, item -> "[$groupIndex][$itemIndex] $item" }
}

indexed.forEach { println(it) }
// [0][0] a
// [0][1] b
// [1][0] c
// [1][1] d
// [1][2] e
```

---

## 9. 정리

- **`flatten()`**: `List<List<T>>` → `List<T>`, 이미 중첩된 컬렉션을 한 단계 펼침
- **`flatMap { }`**: `List<T>` 각 원소를 컬렉션으로 변환 후 펼침 (`map` + `flatten`)
- `map { listOf(...) }.flatten()` == `flatMap { listOf(...) }`
- 중복 제거가 필요하면 `.distinct()` 추가
- null 필터링에는 `mapNotNull`이 더 명확하지만 `flatMap { listOfNotNull(...) }`도 동일
