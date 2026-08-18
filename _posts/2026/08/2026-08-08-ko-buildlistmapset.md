---
title: (Kotlin/코틀린) buildList, buildMap, buildSet 완전 정리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 buildList, buildMap, buildSet 빌더 함수로 불변 컬렉션을 명확하게 생성하는 방법과 실전 패턴을 정리합니다.
---

---

## 1. 왜 빌더 함수가 필요한가?

불변 컬렉션을 복잡한 조건으로 구성할 때, 기존 방식은 가변 컬렉션을 만들고 변환해야 합니다.

```kotlin
// 기존 방식 — 가변 리스트 사용 후 변환
fun buildMenuItems(isAdmin: Boolean): List<String> {
    val items = mutableListOf<String>()
    items.add("홈")
    items.add("프로필")
    if (isAdmin) {
        items.add("관리자 패널")
        items.add("사용자 관리")
    }
    return items.toList()
}

// buildList — 간결하고 의도가 명확
fun buildMenuItems(isAdmin: Boolean): List<String> = buildList {
    add("홈")
    add("프로필")
    if (isAdmin) {
        add("관리자 패널")
        add("사용자 관리")
    }
}
```

---

## 2. buildList

`MutableList`의 모든 API를 블록 안에서 사용할 수 있고, 결과는 **불변 List**로 반환됩니다.

```kotlin
// 기본 사용
val numbers = buildList {
    add(1)
    add(2)
    addAll(listOf(3, 4, 5))
}
println(numbers)  // [1, 2, 3, 4, 5]

// 용량 힌트 (선택)
val sized = buildList(capacity = 10) {
    repeat(5) { add(it) }
}

// 조건부 추가
val items = buildList {
    add("기본 아이템")
    if (System.currentTimeMillis() % 2 == 0L) add("짝수 시간 아이템")
    addAll((1..3).map { "반복 아이템 $it" })
}
```

---

## 3. buildMap

`MutableMap`의 API를 사용하며, 결과는 **불변 Map**으로 반환됩니다.

```kotlin
val config = buildMap {
    put("host", "localhost")
    put("port", "8080")
    put("debug", "false")
}
println(config)  // {host=localhost, port=8080, debug=false}

// 조건부 항목 추가
fun buildRequestHeaders(
    token: String?,
    language: String = "ko"
): Map<String, String> = buildMap {
    put("Content-Type", "application/json")
    put("Accept-Language", language)
    if (token != null) {
        put("Authorization", "Bearer $token")
    }
    put("X-Timestamp", System.currentTimeMillis().toString())
}
```

---

## 4. buildSet

`MutableSet`의 API를 사용하며, 결과는 **불변 Set**으로 반환됩니다.

```kotlin
val tags = buildSet {
    add("kotlin")
    add("android")
    addAll(listOf("coroutines", "flow"))
    add("kotlin")  // 중복 자동 제거
}
println(tags)  // [kotlin, android, coroutines, flow]

// 여러 소스에서 태그 수집
fun collectTags(
    postTags: List<String>,
    userTags: List<String>,
    defaultTags: List<String>
): Set<String> = buildSet {
    addAll(postTags)
    addAll(userTags)
    addAll(defaultTags)
    remove("deprecated")  // 특정 태그 제거
}
```

---

## 5. 실전 패턴 — API 요청 파라미터 빌드

```kotlin
data class SearchQuery(
    val keyword: String,
    val category: String? = null,
    val minPrice: Int? = null,
    val maxPrice: Int? = null,
    val sortBy: String = "relevance",
    val page: Int = 1
)

fun SearchQuery.toQueryParams(): Map<String, String> = buildMap {
    put("q", keyword)
    put("sort", sortBy)
    put("page", page.toString())
    category?.let { put("category", it) }
    minPrice?.let { put("min_price", it.toString()) }
    maxPrice?.let { put("max_price", it.toString()) }
}

val query = SearchQuery(
    keyword = "노트북",
    category = "전자제품",
    minPrice = 500000,
    sortBy = "price_asc"
)
println(query.toQueryParams())
// {q=노트북, sort=price_asc, page=1, category=전자제품, min_price=500000}
```

---

## 6. 실전 패턴 — 권한 목록 동적 구성

```kotlin
enum class Permission {
    READ, WRITE, DELETE, ADMIN, EXPORT, IMPORT
}

fun buildPermissions(
    role: String,
    additionalPerms: List<Permission> = emptyList()
): Set<Permission> = buildSet {
    // 기본 권한
    add(Permission.READ)

    when (role) {
        "EDITOR" -> {
            add(Permission.WRITE)
            add(Permission.IMPORT)
        }
        "MANAGER" -> {
            addAll(listOf(Permission.WRITE, Permission.DELETE, Permission.EXPORT, Permission.IMPORT))
        }
        "ADMIN" -> {
            addAll(Permission.values().toList())
        }
    }

    // 추가 권한
    addAll(additionalPerms)
}

val editorPerms = buildPermissions("EDITOR", listOf(Permission.EXPORT))
println(editorPerms)  // [READ, WRITE, IMPORT, EXPORT]
```

---

## 7. 실전 패턴 — 중첩 구조 빌드

```kotlin
data class MenuItem(val label: String, val route: String, val children: List<MenuItem> = emptyList())

fun buildNavigationMenu(userRole: String): List<MenuItem> = buildList {
    add(MenuItem("홈", "/home"))
    add(MenuItem("프로필", "/profile"))

    add(MenuItem("설정", "/settings", buildList {
        add(MenuItem("계정", "/settings/account"))
        add(MenuItem("알림", "/settings/notifications"))
        if (userRole == "ADMIN") {
            add(MenuItem("보안", "/settings/security"))
        }
    }))

    if (userRole == "ADMIN") {
        add(MenuItem("관리자", "/admin", buildList {
            add(MenuItem("사용자 관리", "/admin/users"))
            add(MenuItem("통계", "/admin/stats"))
            add(MenuItem("시스템", "/admin/system"))
        }))
    }
}

val menu = buildNavigationMenu("ADMIN")
menu.forEach { item ->
    println("${item.label} (${item.children.size}개 하위 메뉴)")
}
```

---

## 8. listOf + filter vs buildList 비교

```kotlin
val items = listOf("apple", "banana", "cherry", "date")

// listOf + filter: 전체 생성 후 필터
val filtered = items.filter { it.length > 5 }

// buildList: 조건을 build 시점에 반영
val built = buildList {
    for (item in items) {
        if (item.length > 5) add(item)
    }
}

// 복잡한 조건에서는 buildList가 더 명확
fun buildValidItems(raw: List<String>): List<String> = buildList {
    for (item in raw) {
        val trimmed = item.trim()
        if (trimmed.isNotEmpty() && !trimmed.startsWith("#")) {
            add(trimmed.lowercase())
        }
    }
}
```

---

## 9. 정리

- **`buildList { }`**: MutableList API 사용 → 불변 List 반환
- **`buildMap { }`**: MutableMap API 사용 → 불변 Map 반환
- **`buildSet { }`**: MutableSet API 사용 → 불변 Set 반환
- 조건부 원소 추가, 여러 소스 병합, 중첩 구조 생성에 적합
- `mutableListOf() + toList()` 패턴보다 의도가 명확하고 간결
- Kotlin 1.6부터 Stable API
