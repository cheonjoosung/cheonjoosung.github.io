---
title: (Kotlin/코틀린) takeIf, takeUnless 실전 패턴
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 takeIf, takeUnless 함수로 조건부 null 반환을 간결하게 표현하는 방법과 let, also와의 조합 패턴을 정리합니다.
---

---

## 1. takeIf란?

수신 객체가 **조건을 만족하면 객체 자체를 반환**하고, 만족하지 않으면 `null`을 반환합니다.

```kotlin
// 시그니처
fun <T> T.takeIf(predicate: (T) -> Boolean): T?

// 기본 사용
val number = 42
val result = number.takeIf { it > 10 }  // 42 (조건 만족)
val result2 = number.takeIf { it > 100 }  // null (조건 불만족)

// if-else와 비교
// 기존 방식
val value = if (number > 10) number else null

// takeIf
val value = number.takeIf { it > 10 }
```

---

## 2. takeUnless란?

`takeIf`의 반대 — **조건을 만족하지 않으면 객체를 반환**합니다.

```kotlin
// 시그니처
fun <T> T.takeUnless(predicate: (T) -> Boolean): T?

val name = "Alice"
val result = name.takeUnless { it.isBlank() }  // "Alice" (blank 아님 → 반환)

val empty = ""
val result2 = empty.takeUnless { it.isBlank() }  // null (blank → null)
```

---

## 3. takeIf vs takeUnless 선택

```kotlin
val text = "  hello  ".trim()

// takeIf: 조건이 true일 때 반환
val v1 = text.takeIf { it.isNotBlank() }  // "hello"

// takeUnless: 조건이 false일 때 반환 (= 조건 반전)
val v2 = text.takeUnless { it.isBlank() }  // "hello"

// 두 결과는 같지만 읽는 방향이 다름
// "블랭크가 아닐 때" → takeIf { isNotBlank() }
// "블랭크일 때 버리기" → takeUnless { isBlank() }
// 더 자연스럽게 읽히는 쪽을 선택
```

---

## 4. let과 조합 — 조건부 처리

```kotlin
// null이 아닐 때만 처리하는 패턴
val userId = getUserIdOrNull()

// 기존 방식
if (userId != null && userId.isNotBlank()) {
    loadUser(userId)
}

// takeIf + let
userId
    ?.takeIf { it.isNotBlank() }
    ?.let { loadUser(it) }
```

```kotlin
// 파일 처리 예시
fun readConfigFile(path: String): String? {
    return File(path)
        .takeIf { it.exists() && it.canRead() }
        ?.readText()
}
```

---

## 5. 실전 패턴 — 입력값 검증

```kotlin
data class SignUpRequest(val username: String, val password: String, val email: String)

fun validateUsername(input: String): String? =
    input.trim()
        .takeIf { it.length in 3..20 }
        ?.takeIf { it.all { c -> c.isLetterOrDigit() || c == '_' } }

fun validatePassword(input: String): String? =
    input.takeIf { it.length >= 8 }
         ?.takeIf { it.any { c -> c.isUpperCase() } }
         ?.takeIf { it.any { c -> c.isDigit() } }

// 사용
val username = validateUsername("Alice_123")  // "Alice_123"
val badUsername = validateUsername("A!")      // null (허용 안 된 문자)
val badPassword = validatePassword("simple")  // null (8자 미만)
```

---

## 6. 실전 패턴 — 캐시 조회

```kotlin
class Repository {
    private val cache = mutableMapOf<Int, User>()

    fun getUser(id: Int): User {
        return cache[id]
            ?.takeIf { !it.isExpired() }  // 캐시가 있고 만료 안 됐을 때만 사용
            ?: fetchFromNetwork(id).also { cache[id] = it }
    }
}
```

---

## 7. 실전 패턴 — 조건부 변환 체인

```kotlin
data class Product(val id: Int, val name: String, val stock: Int, val isActive: Boolean)

fun findAvailableProduct(products: List<Product>, id: Int): Product? {
    return products.find { it.id == id }
        ?.takeIf { it.isActive }
        ?.takeIf { it.stock > 0 }
}

// 결과에 따라 분기
val product = findAvailableProduct(products, productId)
    ?: run {
        log("상품을 찾을 수 없거나 구매 불가 상태")
        return@launch
    }
```

---

## 8. 실전 패턴 — ViewModel에서 UI 상태 제어

```kotlin
class SearchViewModel : ViewModel() {
    private val _query = MutableStateFlow("")
    val query = _query.asStateFlow()

    fun search(input: String) {
        // 빈 문자열이나 2글자 미만이면 검색하지 않음
        input.trim()
            .takeIf { it.length >= 2 }
            ?.let { query ->
                viewModelScope.launch {
                    _results.value = repository.search(query)
                }
            }
            ?: run {
                _results.value = emptyList()
            }
    }
}
```

---

## 9. takeIf를 남용하지 말아야 할 경우

```kotlin
// 과도한 체이닝 — 가독성 저하
val result = value
    .takeIf { conditionA(it) }
    ?.takeIf { conditionB(it) }
    ?.takeIf { conditionC(it) }
    ?.takeIf { conditionD(it) }

// 이런 경우는 require/filter/명시적 if가 더 명확
val result = if (conditionA(value) && conditionB(value) && conditionC(value)) {
    value
} else {
    null
}

// 단순 비교 — takeIf보다 직접 비교가 나음
val x = 5.takeIf { it > 3 }  // 가능하지만
val x = if (5 > 3) 5 else null  // 더 명확할 수 있음
```

---

## 10. 정리

- **`takeIf { condition }`**: 조건 true → 원본 반환, false → null
- **`takeUnless { condition }`**: 조건 false → 원본 반환, true → null
- `?.let { }` 와 조합해 null-safe 조건부 처리 체인 구성
- 입력 검증, 캐시 조회, 조건부 API 호출 등에 활용
- 조건이 3개 이상 체이닝되면 `if` 표현식이 더 가독성이 좋을 수 있음
