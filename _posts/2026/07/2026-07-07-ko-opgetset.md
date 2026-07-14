---
title: (Kotlin/코틀린) operator fun get/set — 배열처럼 접근
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 operator fun get/set으로 객체를 배열/맵처럼 인덱스로 접근하는 패턴과 Matrix, 캐시, DSL 구성 실전 예제를 정리합니다.
---

---

## 1. operator fun get / set이란?

`get`과 `set` 연산자를 정의하면 객체를 **배열이나 맵처럼 `[]` 인덱스 문법으로 접근**할 수 있습니다.

```kotlin
obj[index]          → obj.get(index)
obj[index] = value  → obj.set(index, value)
```

```kotlin
class StringBox {
    private val data = mutableListOf<String>()

    operator fun get(index: Int): String = data[index]
    operator fun set(index: Int, value: String) { data[index] = value }

    fun add(value: String) = data.add(value)
}

val box = StringBox()
box.add("hello")
box.add("world")

println(box[0])   // hello
box[1] = "kotlin"
println(box[1])   // kotlin
```

---

## 2. 다차원 인덱스

`get`/`set`은 인자를 여러 개 받을 수 있습니다.

```kotlin
class Matrix(private val rows: Int, private val cols: Int) {
    private val data = Array(rows) { DoubleArray(cols) }

    operator fun get(row: Int, col: Int): Double = data[row][col]
    operator fun set(row: Int, col: Int, value: Double) { data[row][col] = value }
}

val matrix = Matrix(3, 3)
matrix[0, 0] = 1.0
matrix[1, 1] = 5.0
matrix[2, 2] = 9.0

println(matrix[1, 1])  // 5.0
```

---

## 3. 타입 키 인덱스

정수 외에도 어떤 타입이든 키로 사용할 수 있습니다.

```kotlin
class TypedConfig {
    private val settings = mutableMapOf<String, Any>()

    operator fun get(key: String): Any? = settings[key]
    operator fun set(key: String, value: Any) { settings[key] = value }
}

val config = TypedConfig()
config["timeout"] = 30
config["baseUrl"] = "https://api.example.com"
config["retries"] = 3

println(config["timeout"])  // 30
println(config["baseUrl"])  // https://api.example.com
```

---

## 4. 실전 패턴 — 캐시 구현

```kotlin
class Cache<K, V>(private val loader: (K) -> V) {
    private val store = mutableMapOf<K, V>()

    operator fun get(key: K): V = store.getOrPut(key) { loader(key) }

    operator fun set(key: K, value: V) { store[key] = value }

    fun invalidate(key: K) = store.remove(key)
}

// 사용
val userCache = Cache<String, User> { id ->
    repository.fetchUser(id)  // 캐시 미스 시 자동 로드
}

val user = userCache["user_123"]  // 첫 접근: DB 조회
val same = userCache["user_123"]  // 두 번째: 캐시에서 반환

// 수동 갱신
userCache["user_123"] = updatedUser
```

---

## 5. 실전 패턴 — 스파스 배열(Sparse Array)

```kotlin
class SparseArray<T> {
    private val data = mutableMapOf<Int, T>()
    var size = 0
        private set

    operator fun get(index: Int): T? = data[index]

    operator fun set(index: Int, value: T) {
        data[index] = value
        if (index >= size) size = index + 1
    }

    fun remove(index: Int) = data.remove(index)
}

val sparse = SparseArray<String>()
sparse[0]    = "첫 번째"
sparse[100]  = "백 번째"
sparse[9999] = "만 번째"

println(sparse[100])   // 백 번째
println(sparse[50])    // null (비어있는 인덱스)
println(sparse.size)   // 10000
```

---

## 6. 실전 패턴 — DSL 설정 빌더

```kotlin
class Headers {
    private val map = mutableMapOf<String, String>()

    operator fun get(name: String): String? = map[name]
    operator fun set(name: String, value: String) { map[name] = value }

    fun toMap(): Map<String, String> = map.toMap()
}

class RequestBuilder {
    val headers = Headers()
    var url: String = ""
    var method: String = "GET"
}

fun request(block: RequestBuilder.() -> Unit): RequestBuilder =
    RequestBuilder().apply(block)

// DSL 사용
val req = request {
    url = "https://api.example.com/users"
    method = "POST"
    headers["Content-Type"] = "application/json"
    headers["Authorization"] = "Bearer token123"
}

println(req.headers["Content-Type"])  // application/json
```

---

## 7. Kotlin 표준 라이브러리와의 일관성

`List`, `Map`, `Array` 등 표준 컬렉션도 동일하게 `operator fun get`/`set`으로 구현되어 있습니다.

```kotlin
val list = mutableListOf(1, 2, 3)
list[0]    // list.get(0)
list[0] = 10  // list.set(0, 10)

val map = mutableMapOf("a" to 1)
map["a"]    // map.get("a")
map["b"] = 2  // map.set("b", 2)
```

커스텀 클래스에 `get`/`set`을 정의하면 이 컬렉션들과 동일한 문법으로 접근할 수 있어 **API가 직관적**입니다.

---

## 8. componentN과의 차이

| 연산자 | 문법 | 주요 용도 |
|--------|------|----------|
| `get` | `obj[i]` | 인덱스 기반 접근 |
| `set` | `obj[i] = v` | 인덱스 기반 수정 |
| `componentN` | `val (a, b) = obj` | 구조 분해 |

```kotlin
data class Point(val x: Int, val y: Int)

// componentN (data class 자동 생성)
val (x, y) = Point(1, 2)

// get 연산자 (직접 구현)
operator fun Point.get(index: Int) = when (index) {
    0 -> x
    1 -> y
    else -> throw IndexOutOfBoundsException()
}

val p = Point(1, 2)
println(p[0])  // 1
println(p[1])  // 2
```

---

## 9. 정리

- `operator fun get(index)` → `obj[index]` 문법으로 접근
- `operator fun set(index, value)` → `obj[index] = value` 문법으로 수정
- 인덱스 타입은 `Int` 외에도 `String`, enum, 커스텀 타입 모두 가능
- 다차원 인덱스 지원: `obj[row, col]` → `get(row, col)`
- 캐시, 행렬, DSL 빌더 등 **컬렉션처럼 읽히는 API**를 만들 때 활용
