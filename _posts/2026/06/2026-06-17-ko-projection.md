---
title: (Kotlin/코틀린) Kotlin star projection(*) 언제 쓰는가
tags: [ Kotlin, Android ]
style: fill
color: dark
description: Kotlin star projection(*)의 개념, Any?와의 차이, variance(out/in)와의 관계, 타입을 모를 때의 안전한 처리 방법을 Android 실전 예제로 정리합니다.
---

## 개요

- 제네릭 타입의 **구체 타입 인자를 모를 때** 사용하는 `*` (Star Projection)을 다룹니다.
- 이 글에서는 다음을 설명합니다.
  - star projection이 필요한 이유
  - `List<*>` vs `List<Any?>` 차이
  - variance(`out`/`in`)에 따른 star projection 동작
  - Android 실전 예제 (reflection, 타입 검사)

---

## 1. 왜 필요한가

```kotlin
// 구체 타입을 모르는 List를 매개변수로 받아야 할 때
fun printSize(list: List<*>) {
    println("크기: ${list.size}")
}

printSize(listOf(1, 2, 3))
printSize(listOf("a", "b"))
printSize(listOf(User("철수"), User("영희")))
// 타입 인자가 Int든 String이든 User든 상관없이 받을 수 있음
```

```kotlin
// ❌ 제네릭 타입 자체를 매개변수로 받으면 타입 인자가 고정됨
fun printSizeWrong(list: List<Int>) { /* Int 리스트만 받음 */ }

// ❌ Any로 받으면 List라는 사실조차 잃음
fun printSizeWrong2(list: Any) { /* list.size 접근 불가 */ }
```

---

## 2. List<*> vs List<Any?>

```kotlin
val numbers: List<Int> = listOf(1, 2, 3)

// List<*> — "타입은 모르지만 List다" (읽기는 안전, 쓰기는 제한)
val starList: List<*> = numbers
val item: Any? = starList[0]  // Any?로만 읽을 수 있음

// MutableList<*> — add 시 구체 타입을 모르므로 추가 불가 (Nothing 취급)
val mutableStarList: MutableList<*> = mutableListOf(1, 2, 3)
// mutableStarList.add(4)  // ❌ 컴파일 에러 — 어떤 타입인지 모르므로 안전하지 않음
```

```kotlin
// List<Any?> — "모든 타입을 섞어 담을 수 있는 List"
val mixedList: MutableList<Any?> = mutableListOf(1, "a", true)
mixedList.add(3.14)  // ✅ Any?는 모든 타입을 허용하므로 추가 가능

// List<*>는 List<Any?>의 "서브타입을 모르는 버전" — 다른 의미
val intList: List<Int> = listOf(1, 2, 3)
val starRef: List<*> = intList         // ✅ 가능 — 타입 인자를 숨김
// val anyRef: List<Any?> = intList    // ❌ List<Int>는 List<Any?>의 서브타입 아님 (invariant)
```

| 항목 | `List<*>` | `List<Any?>` |
|------|-----------|--------------|
| 의미 | 알 수 없는 특정 타입 T의 List | Any?를 담는 List (모든 타입 혼합 가능) |
| 대입 가능성 | 모든 `List<T>`를 대입 가능 | `List<Any?>` 자체만 대입 가능 |
| 쓰기(add) | 불가능(읽기 전용처럼 동작) | 가능 |

---

## 3. variance와 star projection의 관계

Kotlin은 타입 매개변수의 variance(`out`/`in`/무공변)에 따라 `*`의 동작이 달라집니다.

```kotlin
// out T — covariant. *는 Any?(상한)로 취급됨, 읽기만 가능
class Producer<out T>(private val item: T) {
    fun get(): T = item
}

val p: Producer<*> = Producer("hello")
val value: Any? = p.get()  // T가 Any?로 추론됨
```

```kotlin
// in T — contravariant. *는 Nothing(하한)으로 취급됨, 쓰기 불가
class Consumer<in T> {
    fun consume(item: T) { println(item) }
}

val c: Consumer<*> = Consumer<String>()
// c.consume("hi")  // ❌ 컴파일 에러 — T가 Nothing으로 취급되어 어떤 값도 전달 불가
```

```kotlin
// 무공변 T (out/in 없음) — *는 "T : 상한 upper bound"로 취급, 읽기만 안전
class Box<T>(var value: T)

val box: Box<*> = Box("hello")
val item: Any? = box.value   // 읽기는 가능
// box.value = "world"        // ❌ 쓰기 불가 — 구체 타입을 모르므로 안전하지 않음
```

| 선언 | `*`의 의미 | 가능한 연산 |
|------|------------|-------------|
| `Producer<out T>` | `Any?` (상한으로 치환) | 읽기만 가능 |
| `Consumer<in T>` | `Nothing` (하한으로 치환) | 거의 모든 연산 불가 |
| `Box<T>` (무공변) | `Any?` (upper bound 기준) | 읽기만 가능 |

---

## 4. star projection vs Any 타입 매개변수

```kotlin
// 함수에서 "타입은 모르지만 어떤 제네릭 클래스인지는 안다"
fun describe(list: List<*>): String = "리스트, 크기 ${list.size}"

// "완전히 타입을 모른다" — 제네릭 자체도 모름
fun describeAny(value: Any?): String = "값: $value"

describe(listOf(1, 2, 3))      // "리스트, 크기 3" — List라는 구조 정보 유지
describeAny(listOf(1, 2, 3))   // "값: [1, 2, 3]" — List인지 알 수 없음, toString만 가능
```

---

## 5. Android 실전 예제

### Reflection 기반 처리

```kotlin
fun isEmptyCollection(value: Any?): Boolean {
    return when (value) {
        is List<*>  -> value.isEmpty()
        is Map<*, *> -> value.isEmpty()
        is Set<*>   -> value.isEmpty()
        else -> false
    }
}

// Bundle/Intent extras 검사 시 활용
fun logExtras(bundle: Bundle) {
    for (key in bundle.keySet()) {
        val value = bundle.get(key)
        if (value is List<*>) {
            println("$key: List 크기 ${value.size}")
        }
    }
}
```

### JSON 파싱 유틸에서 타입 무관 처리

```kotlin
fun flattenJsonArray(value: Any?): List<String> {
    return when (value) {
        is List<*> -> value.map { it.toString() }
        is Array<*> -> value.map { it.toString() }
        else -> listOf(value.toString())
    }
}
```

### RecyclerView Adapter — 타입을 모르는 ViewHolder 캐스팅 처리

```kotlin
fun bindAnyAdapter(adapter: RecyclerView.Adapter<*>) {
    // 구체 ViewHolder 타입은 모르지만 Adapter라는 사실은 알고 있음
    println("아이템 수: ${adapter.itemCount}")
}

// Class<*> — 클래스 타입 인자를 모를 때
fun getSimpleName(clazz: Class<*>): String = clazz.simpleName
getSimpleName(User::class.java)
getSimpleName(Post::class.java)
```

---

## 6. 정리

| 항목 | 내용 |
|------|------|
| `*` | 제네릭의 구체 타입 인자를 모를 때 사용하는 projection |
| `List<*>` | 어떤 타입의 List인지 모름, 읽기 전용처럼 동작 |
| `List<Any?>` | 모든 타입을 섞어 담는 List, 쓰기 가능 |
| `out T`의 `*` | `Any?`로 취급 (상한) |
| `in T`의 `*` | `Nothing`으로 취급 (하한), 거의 모든 연산 불가 |
| 주 사용처 | Reflection, 타입 검사(`is List<*>`), 제네릭 구조만 필요한 함수 |

- star projection은 **"구조는 알지만 타입 인자는 모른다"** 는 상태를 표현합니다.
- 완전히 타입 정보가 필요 없으면 `Any?`, 제네릭 구조(List, Map 등)는 유지하되 타입 인자를 무시하려면 `*`를 사용합니다.

---

## 참고

- [Kotlin 공식 문서 — Generics: in, out, where](https://kotlinlang.org/docs/generics.html)
- [Kotlin 제네릭 심화 포스팅 보기](/posts/2026-06-16-ko-generics-upper-bound)
