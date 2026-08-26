---
title: (Kotlin/코틀린) 함수 타입과 함수 리터럴 완전 정리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 함수 타입 표기법, 람다, 익명 함수, 함수 참조(::), 수신자 있는 함수 타입을 완전 정리합니다.
---

---

## 1. 함수 타입 표기법

```kotlin
// (파라미터 타입) -> 반환 타입
val sum: (Int, Int) -> Int = { a, b -> a + b }
val greet: (String) -> Unit = { name -> println("Hello, $name!") }
val noArgs: () -> Boolean = { true }

// nullable 함수 타입
val maybeAction: (() -> Unit)? = null
maybeAction?.invoke()  // null-safe 호출

// 고차 함수 파라미터
fun repeat(times: Int, action: (Int) -> Unit) {
    for (i in 0 until times) action(i)
}
repeat(3) { println("반복: $it") }
```

---

## 2. 람다 표현식

```kotlin
// 기본 람다
val multiply = { a: Int, b: Int -> a * b }

// 단일 파라미터: it 사용
val double: (Int) -> Int = { it * 2 }

// 마지막 파라미터가 함수이면 괄호 밖으로 이동
listOf(1, 2, 3).forEach { println(it) }

// 사용하지 않는 파라미터: _
mapOf("a" to 1).forEach { (_, value) -> println(value) }

// 여러 줄 람다: 마지막 표현식이 반환값
val process: (Int) -> String = { n ->
    val doubled = n * 2
    val squared = n * n
    "doubled=$doubled, squared=$squared"  // 반환
}
```

---

## 3. 익명 함수

람다와 달리 `return`으로 명시적 반환이 가능합니다.

```kotlin
// 익명 함수
val isEven = fun(n: Int): Boolean {
    return n % 2 == 0
}

// 람다의 return은 바깥 함수를 종료 (non-local return)
fun findFirst(list: List<Int>): Int? {
    list.forEach { n ->
        if (n > 10) return n  // forEach가 아닌 findFirst를 반환
    }
    return null
}

// 익명 함수의 return은 익명 함수만 종료 (local return)
fun findFirst2(list: List<Int>): Int? {
    list.forEach(fun(n) {
        if (n > 10) return  // 이 익명 함수만 종료, forEach 계속
    })
    return null
}
```

---

## 4. 함수 참조 (::)

기존 함수를 함수 타입 값으로 사용합니다.

```kotlin
fun isPositive(n: Int) = n > 0
fun doubleIt(n: Int) = n * 2

// 함수 참조
val ref: (Int) -> Boolean = ::isPositive
val numbers = listOf(-1, 2, -3, 4, 5)
println(numbers.filter(::isPositive))  // [2, 4, 5]
println(numbers.map(::doubleIt))       // [-2, 4, -6, 8, 10]

// 멤버 함수 참조
class Validator {
    fun isValid(s: String) = s.isNotBlank()
}
val validator = Validator()
val check: (String) -> Boolean = validator::isValid

// 생성자 참조
data class User(val name: String)
val createUser: (String) -> User = ::User
val users = listOf("Alice", "Bob").map(::User)
```

---

## 5. 수신자 있는 함수 타입

`T.() -> R` — 수신자 객체(`this`)가 있는 함수 타입입니다.

```kotlin
// 수신자 있는 함수 타입
val appendHello: StringBuilder.() -> Unit = {
    append("Hello")
    append(", World!")
}

val sb = StringBuilder()
sb.appendHello()
println(sb)  // Hello, World!

// 고차 함수에서 수신자 활용
fun buildString(action: StringBuilder.() -> Unit): String {
    val sb = StringBuilder()
    sb.action()  // 또는 action(sb)
    return sb.toString()
}

val result = buildString {
    append("Kotlin ")
    append("is ")
    append("great!")
}
println(result)  // Kotlin is great!
```

DSL 빌더 패턴의 핵심입니다.

---

## 6. 함수 타입의 invoke

```kotlin
val action: () -> Unit = { println("실행!") }

// 호출 방법 3가지 — 모두 동일
action()
action.invoke()
(action)()

// nullable 함수 타입 안전 호출
val maybeAction: (() -> Unit)? = null
maybeAction?.invoke()  // null이면 무시
```

---

## 7. 함수 합성

```kotlin
// 함수를 조합해 새 함수 생성
fun <A, B, C> compose(f: (B) -> C, g: (A) -> B): (A) -> C = { a -> f(g(a)) }

val trim: (String) -> String = String::trim
val uppercase: (String) -> String = String::uppercase
val exclaim: (String) -> String = { "$it!" }

val process = compose(exclaim, compose(uppercase, trim))
println(process("  hello  "))  // HELLO!

// 확장 함수로 더 자연스럽게
infix fun <A, B, C> ((A) -> B).andThen(f: (B) -> C): (A) -> C = { a -> f(this(a)) }

val pipeline = trim andThen uppercase andThen exclaim
println(pipeline("  world  "))  // WORLD!
```

---

## 8. 실전 패턴 — 이벤트 핸들러

```kotlin
class EventBus {
    private val handlers = mutableMapOf<String, MutableList<(Any) -> Unit>>()

    fun <T : Any> subscribe(event: String, handler: (T) -> Unit) {
        @Suppress("UNCHECKED_CAST")
        handlers.getOrPut(event) { mutableListOf() }
            .add(handler as (Any) -> Unit)
    }

    fun publish(event: String, data: Any) {
        handlers[event]?.forEach { it(data) }
    }
}

val bus = EventBus()
bus.subscribe<String>("login") { userId ->
    println("로그인: $userId")
}
bus.subscribe<String>("login") { userId ->
    println("로그인 로그: $userId at ${System.currentTimeMillis()}")
}
bus.publish("login", "user-123")
```

---

## 9. 정리

| 구분 | 표기 | 특징 |
|------|------|------|
| 함수 타입 | `(A, B) -> R` | 변수/파라미터 타입 선언 |
| 람다 | `{ a, b -> expr }` | 간결, non-local return 가능 |
| 익명 함수 | `fun(a: A): R { return }` | local return, 명시적 타입 |
| 함수 참조 | `::function` | 기존 함수를 값으로 |
| 수신자 함수 | `T.() -> R` | DSL 빌더 패턴의 핵심 |

- 단순 변환/필터: 람다
- 중간에 return 필요: 익명 함수 또는 람다 + `return@label`
- 기존 함수 재사용: 함수 참조 `::`
- DSL 빌더: 수신자 있는 함수 타입
