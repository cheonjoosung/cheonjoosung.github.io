---
title: (Kotlin/코틀린) SAM 변환(Functional Interface) — Java 람다 대체
tags: [ Kotlin, Android ]
style: fill
color: dark
description: Kotlin SAM 변환과 fun interface의 개념, Kotlin 인터페이스와의 차이, Java 람다 호환성, Android 콜백 설계에서의 실전 활용을 정리합니다.
---

## 개요

- **SAM(Single Abstract Method) 변환** 과 `fun interface`를 다룹니다.
- 추상 메서드가 하나뿐인 인터페이스를 **람다식으로 대체**할 수 있게 하는 기능입니다.
- 이 글에서는 다음을 설명합니다.
  - SAM 변환이 필요한 이유
  - `fun interface` 선언과 람다 변환
  - 일반 인터페이스와의 차이
  - Java 인터페이스와의 SAM 변환 호환성
  - Android 콜백 설계 실전 예제

---

## 1. 왜 필요한가

```kotlin
// 일반 인터페이스 — 람다로 대체 불가
interface OnClickListener {
    fun onClick(view: View)
}

// ❌ 익명 객체로 매번 작성해야 함
button.setOnClickListener(object : OnClickListener {
    override fun onClick(view: View) {
        println("클릭됨")
    }
})
```

```kotlin
// fun interface로 선언하면 람다 변환 가능
fun interface OnClickListener {
    fun onClick(view: View)
}

// ✅ 람다식으로 간결하게 작성
button.setOnClickListener { view -> println("클릭됨: $view") }
```

---

## 2. fun interface 선언

```kotlin
// 추상 메서드는 정확히 1개만 가능
fun interface Validator<T> {
    fun validate(value: T): Boolean
}

// 사용 — 람다로 즉시 전달
val notEmpty: Validator<String> = Validator { it.isNotEmpty() }
val isEmail: Validator<String> = Validator { it.contains("@") }

fun checkAll(value: String, validators: List<Validator<String>>): Boolean =
    validators.all { it.validate(value) }

checkAll("user@test.com", listOf(notEmpty, isEmail))  // true
```

```kotlin
// default 메서드는 추가 가능 (추상 메서드는 1개 유지)
fun interface StringTransformer {
    fun transform(input: String): String

    fun transformTwice(input: String): String = transform(transform(input))
}

val upper = StringTransformer { it.uppercase() }
println(upper.transformTwice("hello"))  // HELLO
```

---

## 3. 일반 인터페이스 vs fun interface

```kotlin
// ❌ 추상 메서드가 2개 이상이면 fun interface 불가
fun interface Repository<T> {
    fun get(id: Long): T?
    fun save(item: T)  // 컴파일 에러 — SAM 조건 위반
}

// ✅ 추상 메서드가 여러 개면 일반 interface 사용
interface Repository<T> {
    fun get(id: Long): T?
    fun save(item: T)
}
```

| 항목 | 일반 `interface` | `fun interface` |
|------|-------------------|------------------|
| 추상 메서드 수 | 여러 개 가능 | 정확히 1개 |
| 람다 변환 | ❌ 불가 | ✅ 가능 |
| 구현 방법 | 클래스/객체로 구현 | 람다 또는 클래스 모두 가능 |
| 주 사용처 | 여러 동작을 묶는 계약 | 단일 동작(콜백, 전략) |

---

## 4. Java 인터페이스와 SAM 변환

```kotlin
// Java의 함수형 인터페이스(@FunctionalInterface)도 Kotlin에서 람다로 호출 가능
// java.lang.Runnable, java.util.Comparator 등

val runnable: Runnable = Runnable { println("실행") }

val comparator = Comparator<String> { a, b -> a.length - b.length }
listOf("ccc", "a", "bb").sortedWith(comparator)

// Java 메서드가 함수형 인터페이스를 매개변수로 받으면 람다로 직접 전달 가능
executor.execute { println("작업 실행") }  // Runnable 대신 람다
```

```kotlin
// Kotlin 1.4+ — Kotlin interface도 fun interface로 선언하면 동일하게 동작
fun interface Callback {
    fun onResult(result: String)
}

fun fetchData(callback: Callback) {
    callback.onResult("데이터")
}

// 람다로 전달
fetchData { result -> println(result) }
```

---

## 5. Android 실전 예제

### 콜백 인터페이스 단순화

```kotlin
fun interface OnItemClickListener<T> {
    fun onItemClick(item: T, position: Int)
}

class UserAdapter(
    private val items: List<User>,
    private val onItemClick: OnItemClickListener<User>
) : RecyclerView.Adapter<UserViewHolder>() {

    override fun onBindViewHolder(holder: UserViewHolder, position: Int) {
        val user = items[position]
        holder.itemView.setOnClickListener {
            onItemClick.onItemClick(user, position)
        }
        holder.bind(user)
    }

    override fun getItemCount() = items.size
}

// 사용 — 람다로 간결하게 전달
val adapter = UserAdapter(users, OnItemClickListener { user, position ->
    println("$position 번째 클릭: ${user.name}")
})
```

### 네트워크 콜백

```kotlin
fun interface ApiCallback<T> {
    fun onComplete(result: Result<T>)
}

class LegacyApiClient {
    fun fetchUser(id: Long, callback: ApiCallback<User>) {
        // 비동기 처리 후 콜백 호출
        thread {
            try {
                val user = api.getUser(id)
                callback.onComplete(Result.success(user))
            } catch (e: Exception) {
                callback.onComplete(Result.failure(e))
            }
        }
    }
}

// 사용
apiClient.fetchUser(1L) { result ->
    result.onSuccess { user -> println("성공: $user") }
        .onFailure { e -> println("실패: ${e.message}") }
}
```

### 전략 패턴과 결합

```kotlin
fun interface SortStrategy<T> {
    fun sort(items: List<T>): List<T>
}

class ProductListSorter(private var strategy: SortStrategy<Product>) {
    fun setStrategy(strategy: SortStrategy<Product>) {
        this.strategy = strategy
    }

    fun getSorted(products: List<Product>): List<Product> = strategy.sort(products)
}

val byPriceAsc = SortStrategy<Product> { it.sortedBy { p -> p.price } }
val byNameAsc  = SortStrategy<Product> { it.sortedBy { p -> p.name } }

val sorter = ProductListSorter(byPriceAsc)
sorter.setStrategy(byNameAsc)
```

---

## 6. 흔한 실수

```kotlin
// ❌ 일반 람다(함수 타입)와 fun interface 혼동
// 함수 타입을 직접 쓰면 인터페이스 구현체가 필요 없는 경우도 많음
fun setOnClick(listener: (View) -> Unit) { /* ... */ }
setOnClick { view -> println(view) }

// fun interface는 "명명된 메서드"와 "추가 default 메서드"가 필요할 때만 사용
fun interface OnClickListener {
    fun onClick(view: View)
    fun onLongClick(view: View): Boolean = false  // default 메서드 활용
}
```

```kotlin
// ❌ 추상 메서드가 여러 개인데 fun interface로 선언 시도 → 컴파일 에러
// ✅ 여러 콜백이 필요하면 일반 interface + 각각 default 분리
interface MultiCallback {
    fun onSuccess(data: String)
    fun onError(e: Throwable)
}
```

---

## 7. 정리

| 항목 | 내용 |
|------|------|
| `fun interface` | 추상 메서드 1개인 인터페이스 → 람다 변환 가능 |
| 조건 | 추상 메서드 정확히 1개, default 메서드는 추가 가능 |
| 일반 interface | 추상 메서드 여러 개 → 람다 변환 불가 |
| Java 호환 | `@FunctionalInterface` 도 동일하게 람다로 호출 가능 |
| Android 활용 | RecyclerView 클릭 리스너, 네트워크 콜백, 전략 패턴 |

- `fun interface`는 **"이름이 있는 람다"** 입니다.
- 단순 함수 타입(`(T) -> R`)보다 의미를 명확히 표현하고 default 메서드를 추가할 수 있을 때 사용합니다.

---

## 참고

- [Kotlin 공식 문서 — Functional interfaces](https://kotlinlang.org/docs/fun-interfaces.html)
- [Value class 포스팅 보기](/posts/2026-06-13-ko-value-class)
