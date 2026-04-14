---
title: (Kotlin/코틀린) Nullable 타입 확장함수 완전 정리 - null 안전 처리 유틸 모음
tags: [ Kotlin ]
style: fill
color: dark
description: (Kotlin/코틀린) String?, Int?, List? 등 nullable 타입에서 바로 쓸 수 있는 확장함수를 null 안전 처리, 기본값, 조건 실행, 변환, 컬렉션까지 실전 예제와 함께 정리합니다.
---

## 개요

- Kotlin은 null 안전성을 언어 차원에서 지원합니다.
- 하지만 실무에서는 `?.`, `?:`, `!!` 가 반복되며 코드가 지저분해지는 경우가 많습니다.
- nullable 타입에 맞는 확장함수를 미리 만들어두면 코드가 훨씬 깔끔해집니다.
- 이 글에서는 `String?`, `Int?`, `List?` 등 nullable 타입에서 바로 호출할 수 있는 확장함수를 카테고리별로 정리합니다.

---

## 1. Nullable String 확장함수

```kotlin
/** null 또는 blank이면 기본값 반환 */
fun String?.orDefault(default: String = ""): String =
    if (this.isNullOrBlank()) default else this

/** null 또는 blank이면 null 반환 (정규화) */
fun String?.blankToNull(): String? =
    if (this.isNullOrBlank()) null else this

/** null이면 빈 문자열 */
fun String?.orEmpty2(): String = this ?: ""

/** null이면 특정 문자열로 대체 */
fun String?.orUnknown(): String = this ?: "알 수 없음"

/** null이 아닐 때만 변환 */
fun String?.mapIfNotNull(transform: (String) -> String): String? =
    this?.let(transform)

/** null이 아니고 조건을 만족할 때만 실행 */
fun String?.ifNotNullAnd(predicate: (String) -> Boolean, block: (String) -> Unit) {
    if (this != null && predicate(this)) block(this)
}

/** 안전한 trim (null이면 null 반환) */
fun String?.trimOrNull(): String? = this?.trim()?.takeIf { it.isNotEmpty() }

/** null-safe 길이 (null이면 0) */
val String?.safeLength: Int get() = this?.length ?: 0

/** null-safe 포함 여부 */
fun String?.containsSafe(other: String, ignoreCase: Boolean = false): Boolean =
    this?.contains(other, ignoreCase) ?: false

/** null-safe 시작 여부 */
fun String?.startsWithSafe(prefix: String): Boolean =
    this?.startsWith(prefix) ?: false
```

**사용 예시**

```kotlin
val name: String? = null
println(name.orDefault("이름 없음"))          // "이름 없음"
println(name.orUnknown())                    // "알 수 없음"
println(name.safeLength)                     // 0

val input: String? = "  "
println(input.blankToNull())                 // null
println(input.trimOrNull())                  // null

val value: String? = "Hello World"
println(value.containsSafe("world", true))  // true
value.ifNotNullAnd({ it.length > 5 }) {
    println("긴 문자열: $it")                // "긴 문자열: Hello World"
}
```

---

## 2. Nullable Number 확장함수

```kotlin
/** null이면 0 반환 */
fun Int?.orZero(): Int = this ?: 0
fun Long?.orZero(): Long = this ?: 0L
fun Double?.orZero(): Double = this ?: 0.0
fun Float?.orZero(): Float = this ?: 0f

/** null이면 기본값 반환 */
fun Int?.orDefault(default: Int): Int = this ?: default
fun Double?.orDefault(default: Double): Double = this ?: default

/** null 또는 0이면 true */
fun Int?.isNullOrZero(): Boolean = this == null || this == 0
fun Long?.isNullOrZero(): Boolean = this == null || this == 0L
fun Double?.isNullOrZero(): Boolean = this == null || this == 0.0

/** null 또는 0 이하이면 true */
fun Int?.isNullOrNotPositive(): Boolean = this == null || this <= 0

/** null이 아니고 양수일 때만 실행 */
inline fun Int?.ifPositive(block: (Int) -> Unit) {
    if (this != null && this > 0) block(this)
}

/** null이 아닐 때만 연산 */
fun Int?.safeAdd(other: Int?): Int = (this ?: 0) + (other ?: 0)
fun Double?.safeAdd(other: Double?): Double = (this ?: 0.0) + (other ?: 0.0)
fun Double?.safeMultiply(other: Double?): Double = (this ?: 0.0) * (other ?: 0.0)
```

**사용 예시**

```kotlin
val count: Int? = null
println(count.orZero())              // 0
println(count.isNullOrZero())        // true
println(count.isNullOrNotPositive()) // true

val price: Int? = 3000
price.ifPositive { println("가격: ${it}원") } // "가격: 3000원"

val a: Double? = 1.5
val b: Double? = null
println(a.safeAdd(b))               // 1.5
println(a.safeMultiply(b))          // 0.0
```

---

## 3. Nullable Boolean 확장함수

```kotlin
/** null이면 false */
fun Boolean?.orFalse(): Boolean = this ?: false

/** null이면 true */
fun Boolean?.orTrue(): Boolean = this ?: true

/** null 또는 false이면 true (부정) */
fun Boolean?.isNullOrFalse(): Boolean = this == null || this == false

/** true일 때만 실행 */
inline fun Boolean?.ifTrue(block: () -> Unit): Boolean? {
    if (this == true) block()
    return this
}

/** false일 때만 실행 */
inline fun Boolean?.ifFalse(block: () -> Unit): Boolean? {
    if (this == false) block()
    return this
}

/** null일 때 실행 */
inline fun Boolean?.ifNull(block: () -> Unit): Boolean? {
    if (this == null) block()
    return this
}
```

**사용 예시**

```kotlin
val isEnabled: Boolean? = null
println(isEnabled.orFalse())     // false
println(isEnabled.orTrue())      // true
println(isEnabled.isNullOrFalse()) // true

val isLoggedIn: Boolean? = true
isLoggedIn
    .ifTrue { println("로그인 성공") }   // 실행됨
    .ifFalse { println("로그인 실패") }  // 실행 안됨
    .ifNull { println("상태 불명") }     // 실행 안됨
```

---

## 4. Nullable Collection 확장함수

```kotlin
/** null이면 빈 리스트 */
fun <T> List<T>?.orEmpty2(): List<T> = this ?: emptyList()

/** null 또는 비어있으면 true */
fun <T> Collection<T>?.isNullOrEmpty2(): Boolean = this == null || this.isEmpty()

/** null 또는 비어있으면 기본 리스트 반환 */
fun <T> List<T>?.orDefault(default: List<T>): List<T> =
    if (this.isNullOrEmpty()) default else this

/** null이 아니고 비어있지 않을 때만 실행 */
inline fun <T> List<T>?.ifNotNullOrEmpty(block: (List<T>) -> Unit) {
    if (!this.isNullOrEmpty()) block(this!!)
}

/** null-safe 사이즈 */
val <T> Collection<T>?.safeSize: Int get() = this?.size ?: 0

/** null-safe 첫 번째 요소 */
fun <T> List<T>?.firstOrNull2(): T? = this?.firstOrNull()

/** null-safe 마지막 요소 */
fun <T> List<T>?.lastOrNull2(): T? = this?.lastOrNull()

/** null 요소 제거 후 non-null 리스트 반환 */
fun <T : Any> List<T?>?.filterNotNullOrEmpty(): List<T> =
    this?.filterNotNull() ?: emptyList()

/** null-safe map 변환 */
fun <T, R> List<T>?.mapOrEmpty(transform: (T) -> R): List<R> =
    this?.map(transform) ?: emptyList()

/** null-safe filter */
fun <T> List<T>?.filterOrEmpty(predicate: (T) -> Boolean): List<T> =
    this?.filter(predicate) ?: emptyList()
```

**사용 예시**

```kotlin
val items: List<String>? = null
println(items.orEmpty2())             // []
println(items.safeSize)               // 0
println(items.isNullOrEmpty2())       // true
println(items.mapOrEmpty { it.uppercase() }) // []

val names: List<String>? = listOf("Alice", "Bob", "Charlie")
names.ifNotNullOrEmpty { println("목록: $it") }
// "목록: [Alice, Bob, Charlie]"

val mixed: List<String?>? = listOf("a", null, "b", null, "c")
println(mixed.filterNotNullOrEmpty()) // [a, b, c]

println(names.filterOrEmpty { it.length > 3 }) // [Alice, Charlie]
```

---

## 5. 범용 Nullable 확장함수

```kotlin
/** null이면 예외 throw (커스텀 메시지) */
fun <T : Any> T?.requireNotNull(message: String = "값이 null입니다"): T =
    this ?: throw IllegalStateException(message)

/** null이면 block 실행 후 null 반환, 아니면 그대로 반환 */
inline fun <T> T?.onNull(block: () -> Unit): T? {
    if (this == null) block()
    return this
}

/** null이 아닐 때 block 실행, 체이닝 가능 */
inline fun <T> T?.onNotNull(block: (T) -> Unit): T? {
    if (this != null) block(this)
    return this
}

/** null이면 A, 아니면 B 반환 */
fun <T, R> T?.fold(onNull: () -> R, onNotNull: (T) -> R): R =
    if (this == null) onNull() else onNotNull(this)

/** null 여부에 따라 두 값 중 하나 선택 */
fun <T> T?.ifNullElse(nullValue: T, notNullValue: T): T =
    if (this == null) nullValue else notNullValue

/** 조건을 만족하지 않으면 null로 변환 */
fun <T> T?.takeIfNotNull(predicate: (T) -> Boolean): T? =
    this?.takeIf(predicate)

/** 연속된 nullable 체이닝을 가독성 있게 */
infix fun <T> T?.or(default: T): T = this ?: default
```

**사용 예시**

```kotlin
val user: String? = null
user.onNull { println("사용자 없음") }    // "사용자 없음"
    .onNotNull { println("사용자: $it") } // 실행 안됨

val token: String? = "abc123"
val result = token.fold(
    onNull = { "토큰 없음" },
    onNotNull = { "토큰: $it" }
)
println(result) // "토큰: abc123"

val score: Int? = 45
println(score.takeIfNotNull { it >= 60 })  // null (60 미만)
println(score.ifNullElse(0, score!!))      // 45

val value: String? = null
println(value or "기본값")   // "기본값"
```

---

## 6. 실전 조합 예제

### 예제 1 — API 응답 처리

```kotlin
data class UserResponse(
    val name: String?,
    val age: Int?,
    val email: String?,
    val tags: List<String>?
)

fun UserResponse.toDisplayText(): String {
    val displayName = name.orDefault("이름 없음")
    val displayAge  = age.ifNullElse(0, age!!)
    val displayMail = email.blankToNull() ?: "이메일 미등록"
    val tagList     = tags.mapOrEmpty { "#$it" }.joinToString(" ")
    return "$displayName ($displayAge세) - $displayMail $tagList".trim()
}

val response = UserResponse(
    name  = "홍길동",
    age   = null,
    email = "  ",
    tags  = listOf("Kotlin", "Android")
)
println(response.toDisplayText())
// "홍길동 (0세) - 이메일 미등록 #Kotlin #Android"
```

---

### 예제 2 — 설정값 로드

```kotlin
fun loadConfig(
    host: String?,
    port: Int?,
    timeout: Int?,
    retryCount: Int?
): String {
    return buildString {
        append("host=${host.orDefault("localhost")}")
        append(", port=${port.orDefault(8080)}")
        append(", timeout=${timeout.orDefault(30)}s")
        append(", retry=${retryCount.orZero()}회")
    }
}

println(loadConfig(null, 9090, null, null))
// "host=localhost, port=9090, timeout=30s, retry=0회"
```

---

### 예제 3 — 로그인 상태별 분기

```kotlin
val isLoggedIn: Boolean? = null  // null = 로딩 중

isLoggedIn
    .ifTrue  { println("메인 화면으로 이동") }
    .ifFalse { println("로그인 화면으로 이동") }
    .ifNull  { println("로딩 중...") }
// "로딩 중..."
```

---

### 예제 4 — RecyclerView 아이템 처리

```kotlin
fun renderItems(items: List<String>?) {
    items
        .ifNotNullOrEmpty { list ->
            println("아이템 ${list.safeSize}개 표시")
            list.forEachIndexed { i, item -> println("  $i: $item") }
        }
        .onNull { println("빈 화면 표시") }
}

renderItems(null)
// "빈 화면 표시"

renderItems(listOf("Apple", "Banana"))
// "아이템 2개 표시"
// "  0: Apple"
// "  1: Banana"
```

---

## 7. 정리

| 타입 | 주요 확장함수 |
|------|-------------|
| `String?` | `orDefault`, `blankToNull`, `orUnknown`, `trimOrNull`, `safeLength`, `containsSafe` |
| `Int? / Double?` | `orZero`, `orDefault`, `isNullOrZero`, `ifPositive`, `safeAdd` |
| `Boolean?` | `orFalse`, `orTrue`, `isNullOrFalse`, `ifTrue`, `ifFalse`, `ifNull` |
| `List? / Collection?` | `orEmpty2`, `isNullOrEmpty2`, `ifNotNullOrEmpty`, `safeSize`, `mapOrEmpty`, `filterOrEmpty` |
| 범용 `T?` | `onNull`, `onNotNull`, `fold`, `takeIfNotNull`, `requireNotNull`, `or` |

- 프로젝트의 `NullableExt.kt` 파일에 모아두고 팀 전체에서 공유하면 null 처리 코드가 일관되고 간결해집니다.
