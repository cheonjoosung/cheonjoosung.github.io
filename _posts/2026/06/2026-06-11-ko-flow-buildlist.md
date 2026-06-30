---
title: (Kotlin/코틀린) Flow와 함께 쓰는 buildList, buildMap 패턴
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 buildList, buildMap, buildSet 빌더 함수의 동작 원리와 Flow 변환 연산자 내부에서 활용하는 실전 패턴을 정리합니다.
---

---

## 1. buildList / buildMap / buildSet이란?

가변 컬렉션을 만든 뒤 **불변 컬렉션으로 변환**하는 빌더 패턴입니다.  
내부적으로 `MutableList`/`MutableMap`/`MutableSet`을 사용하지만, 외부에는 읽기 전용 타입을 노출합니다.

```kotlin
val list = buildList {
    add(1)
    add(2)
    addAll(listOf(3, 4))
}
// list: List<Int> = [1, 2, 3, 4] (불변)
```

```kotlin
val map = buildMap {
    put("a", 1)
    put("b", 2)
}
// map: Map<String, Int> = {a=1, b=2} (불변)
```

```kotlin
val set = buildSet {
    add(1)
    add(2)
    add(1)  // 중복 무시
}
// set: Set<Int> = [1, 2]
```

### 기존 방식과 비교

```kotlin
// 기존 방식: 가변 리스트를 직접 노출하거나 별도 변환 필요
fun createList(): List<Int> {
    val result = mutableListOf<Int>()
    result.add(1)
    result.add(2)
    return result.toList()  // 변환 단계 필요
}

// buildList: 한 번에 작성, 자동으로 불변 타입 반환
fun createList(): List<Int> = buildList {
    add(1)
    add(2)
}
```

---

## 2. 조건부 빌드 패턴

```kotlin
fun buildUserList(includeAdmins: Boolean, users: List<User>): List<User> = buildList {
    addAll(users.filter { !it.isAdmin })
    if (includeAdmins) {
        addAll(users.filter { it.isAdmin })
    }
}
```

```kotlin
val items = buildList {
    add("기본 항목")
    if (isPremium) add("프리미엄 항목")
    if (hasDiscount) add("할인 항목")
    repeat(3) { add("추가 항목 $it") }
}
```

---

## 3. Flow 변환 연산자에서 활용

`map`, `transform` 같은 연산자 내부에서 리스트를 가공할 때 `buildList`로 가독성을 높일 수 있습니다.

```kotlin
data class RawItem(val id: String, val category: String, val isVisible: Boolean)
data class UiItem(val id: String, val label: String)

rawItemsFlow
    .map { rawItems ->
        buildList {
            for (item in rawItems) {
                if (!item.isVisible) continue
                add(UiItem(id = item.id, label = "${item.category}: ${item.id}"))
            }
            if (isEmpty()) {
                add(UiItem(id = "empty", label = "표시할 항목이 없습니다"))
            }
        }
    }
    .collect { uiItems -> adapter.submitList(uiItems) }
```

### 섹션 헤더 추가 패턴

```kotlin
sealed class ListItem {
    data class Header(val title: String) : ListItem()
    data class Content(val item: Product) : ListItem()
}

productsFlow
    .map { products ->
        val grouped = products.groupBy { it.category }
        buildList {
            grouped.forEach { (category, items) ->
                add(ListItem.Header(category))
                items.forEach { add(ListItem.Content(it)) }
            }
        }
    }
    .collect { listItems -> adapter.submitList(listItems) }
```

---

## 4. buildMap 실전 활용

### Flow에서 인덱싱 맵 생성

```kotlin
usersFlow
    .map { users ->
        buildMap {
            users.forEach { user -> put(user.id, user) }
        }
    }
    .collect { userMap: Map<String, User> ->
        // ID로 빠르게 조회 가능한 맵
        lookupCache = userMap
    }
```

### 쿼리 파라미터 구성

```kotlin
fun buildSearchParams(
    query: String,
    category: String?,
    minPrice: Int?,
    maxPrice: Int?
): Map<String, String> = buildMap {
    put("q", query)
    category?.let { put("category", it) }
    minPrice?.let { put("minPrice", it.toString()) }
    maxPrice?.let { put("maxPrice", it.toString()) }
}

searchQueryFlow
    .map { params -> buildSearchParams(params.query, params.category, params.min, params.max) }
    .flatMapLatest { repo.search(it) }
    .collect { results -> updateUi(results) }
```

---

## 5. capacity 힌트로 성능 최적화

대량 데이터를 다룰 때 초기 용량을 지정하면 내부 배열 재할당을 줄일 수 있습니다.

```kotlin
fun processLargeFlow(items: List<RawItem>): List<ProcessedItem> =
    buildList(capacity = items.size) {  // 정확한 크기 힌트
        items.forEach { add(it.toProcessed()) }
    }
```

```kotlin
buildMap<String, User>(capacity = expectedUserCount) {
    users.forEach { put(it.id, it) }
}
```

---

## 6. transform 연산자와 결합

```kotlin
// 하나의 입력에서 여러 값을 방출하며 buildList로 그룹 단위 처리
ordersFlow
    .transform { orders ->
        val grouped = buildMap<String, List<Order>> {
            orders.groupBy { it.customerId }.forEach { (id, list) -> put(id, list) }
        }
        grouped.forEach { (customerId, customerOrders) ->
            emit(CustomerOrderSummary(customerId, customerOrders))
        }
    }
    .collect { summary -> processSummary(summary) }
```

---

## 7. 일반 컬렉션 연산과 비교

| 상황 | 권장 방식 |
|------|----------|
| 단순 map/filter 변환 | 표준 컬렉션 연산자 (`map`, `filter`) |
| 조건부 항목 추가/분기 로직 복잡 | `buildList` |
| 여러 출처를 하나의 Map으로 합치기 | `buildMap` |
| 중복 제거하며 누적 | `buildSet` |
| 단순 단일 변환 | `map { }`로 충분, buildList 불필요 |

```kotlin
// 단순 변환은 buildList보다 map이 간결
val doubled = list.map { it * 2 }              // 권장
val doubled2 = buildList { list.forEach { add(it * 2) } }  // 불필요하게 복잡
```

---

## 8. 정리

- **`buildList`/`buildMap`/`buildSet`**: 가변 빌더로 작성 후 불변 컬렉션으로 변환
- Flow의 `map`/`transform` 내부에서 **조건부 로직이 복잡한 변환**에 적합
- 단순 1:1 변환에는 표준 `map`/`filter`가 더 간결 — 남용하지 않기
- `capacity` 힌트로 대량 데이터 처리 시 성능 개선 가능
