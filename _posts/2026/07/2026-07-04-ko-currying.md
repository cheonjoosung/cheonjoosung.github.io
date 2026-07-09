---
title: (Kotlin/코틀린) 커링(Currying) 패턴 완전 정리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin에서 커링(Currying) 패턴을 구현하는 방법과 부분 적용(Partial Application), 실전 활용 사례를 정리합니다.
---

---

## 1. 커링이란?

**커링(Currying)**은 여러 인자를 받는 함수를 **인자 하나씩 받는 함수들의 체인**으로 변환하는 기법입니다.

```
f(a, b, c) → f(a)(b)(c)
```

```kotlin
// 일반 함수
fun add(a: Int, b: Int): Int = a + b

// 커링된 함수
fun curriedAdd(a: Int): (Int) -> Int = { b -> a + b }

val add5 = curriedAdd(5)   // 5를 더하는 함수 생성
println(add5(3))            // 8
println(add5(10))           // 15
```

인자를 미리 고정해 **특수화된 함수를 새로 만드는** 것이 핵심입니다.

---

## 2. 부분 적용(Partial Application)

커링과 유사하지만, **일부 인자만 고정**하고 나머지를 나중에 받습니다.

```kotlin
// 3인자 함수
fun formatMessage(prefix: String, level: String, message: String): String =
    "[$prefix][$level] $message"

// 부분 적용: prefix 고정
fun appLogger(level: String, message: String) =
    formatMessage("MyApp", level, message)

// level도 고정
fun errorLogger(message: String) =
    appLogger("ERROR", message)

println(errorLogger("서버 연결 실패"))
// [MyApp][ERROR] 서버 연결 실패
```

---

## 3. Kotlin에서 커링 구현

### 확장 함수로 구현

```kotlin
// 2인자 함수를 커링으로 변환
fun <A, B, C> ((A, B) -> C).curried(): (A) -> (B) -> C =
    { a -> { b -> this(a, b) } }

// 3인자 함수
fun <A, B, C, D> ((A, B, C) -> D).curried(): (A) -> (B) -> (C) -> D =
    { a -> { b -> { c -> this(a, b, c) } } }
```

```kotlin
// 사용
val multiply = { a: Int, b: Int -> a * b }
val curriedMultiply = multiply.curried()

val triple = curriedMultiply(3)  // 3을 곱하는 함수
println(triple(5))  // 15
println(triple(10)) // 30
```

### 역방향 변환 (uncurry)

```kotlin
fun <A, B, C> ((A) -> (B) -> C).uncurried(): (A, B) -> C =
    { a, b -> this(a)(b) }

val normalMultiply = curriedMultiply.uncurried()
println(normalMultiply(3, 5))  // 15
```

---

## 4. 실전 패턴 — 필터 함수 조합

```kotlin
// 커링을 활용한 필터 조건 생성
val isOlderThan: (Int) -> (User) -> Boolean =
    { age -> { user -> user.age > age } }

val isOlderThan20 = isOlderThan(20)
val isOlderThan30 = isOlderThan(30)

val users = listOf(
    User("Alice", 25),
    User("Bob", 17),
    User("Charlie", 35)
)

users.filter(isOlderThan20)  // [Alice, Charlie]
users.filter(isOlderThan30)  // [Charlie]
```

---

## 5. 실전 패턴 — 로거 팩토리

```kotlin
data class Logger(val tag: String, val level: String) {
    fun log(message: String) = println("[$tag][$level] $message")
}

// 커링으로 로거 생성 팩토리
val makeLogger: (String) -> (String) -> Logger =
    { tag -> { level -> Logger(tag, level) } }

val appLoggers = makeLogger("MyApp")
val errorLogger = appLoggers("ERROR")
val infoLogger  = appLoggers("INFO")

errorLogger.log("DB 연결 실패")  // [MyApp][ERROR] DB 연결 실패
infoLogger.log("서버 시작됨")    // [MyApp][INFO] 서버 시작됨
```

---

## 6. 실전 패턴 — API 요청 빌더

```kotlin
// 베이스 URL을 고정하고 엔드포인트만 바꾸는 패턴
val buildRequest: (String) -> (String) -> (Map<String, String>) -> String =
    { baseUrl ->
        { endpoint ->
            { params ->
                val query = params.entries.joinToString("&") { "${it.key}=${it.value}" }
                "$baseUrl/$endpoint?$query"
            }
        }
    }

val apiRequest = buildRequest("https://api.example.com")
val userRequest = apiRequest("users")

println(userRequest(mapOf("id" to "123")))
// https://api.example.com/users?id=123
println(userRequest(mapOf("page" to "1", "size" to "20")))
// https://api.example.com/users?page=1&size=20
```

---

## 7. 커링 vs 기본 인자값(Default Parameter)

Kotlin은 기본 인자값이 있어 커링을 쓰지 않아도 되는 경우가 많습니다.

```kotlin
// 기본 인자값으로 해결 가능한 경우 — 커링 불필요
fun log(message: String, level: String = "INFO", tag: String = "App") =
    println("[$tag][$level] $message")

log("시작됨")                  // [App][INFO] 시작됨
log("오류 발생", level = "ERROR")
```

```kotlin
// 커링이 더 유리한 경우:
// 함수를 값으로 전달하거나, 고차 함수에 넘겨야 할 때
val loggers = listOf("INFO", "WARN", "ERROR")
    .map { level -> { msg: String -> log(msg, level) } }  // 각 레벨의 로거 함수 생성

loggers[2]("치명적 오류")  // [App][ERROR] 치명적 오류
```

| 상황 | 권장 방식 |
|------|----------|
| 단순 인자 기본값 | 기본 인자값 |
| 함수를 값으로 전달 | 커링 |
| 고차 함수의 인자로 활용 | 커링 |
| 함수 조합(compose) | 커링 |

---

## 8. 정리

- **커링**: 다인자 함수를 인자 하나씩 받는 함수 체인으로 변환
- **부분 적용**: 일부 인자를 미리 고정해 특수화된 함수 생성
- Kotlin은 기본 인자값, named argument가 강력하므로 커링을 남용할 필요 없음
- 함수를 **일급 객체로 전달하거나 조합**해야 할 때 커링이 빛남
