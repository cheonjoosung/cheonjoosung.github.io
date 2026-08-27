---
title: (Kotlin/코틀린) 고차 함수 실전 패턴 — 전략 패턴 대체
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 고차 함수로 전략 패턴, 데코레이터, 미들웨어, 파이프라인 패턴을 클래스 없이 구현하는 방법을 정리합니다.
---

---

## 1. 고차 함수란?

함수를 **파라미터로 받거나 반환하는 함수**입니다.

```kotlin
// 함수를 파라미터로 받음
fun transform(list: List<Int>, action: (Int) -> Int): List<Int> =
    list.map(action)

// 함수를 반환함
fun multiplier(factor: Int): (Int) -> Int = { n -> n * factor }

val triple = multiplier(3)
println(triple(5))   // 15
println(transform(listOf(1, 2, 3), triple))  // [3, 6, 9]
```

---

## 2. 전략 패턴 — 클래스 없이 함수로

```kotlin
// 전통적 전략 패턴 (클래스 기반)
interface SortStrategy { fun sort(list: MutableList<Int>) }
class BubbleSort : SortStrategy { override fun sort(list: MutableList<Int>) { /*...*/ } }
class QuickSort : SortStrategy { override fun sort(list: MutableList<Int>) { /*...*/ } }

// 고차 함수로 대체 — 클래스 불필요
typealias SortStrategy = (MutableList<Int>) -> Unit

val bubbleSort: SortStrategy = { list ->
    for (i in list.indices)
        for (j in 0 until list.size - i - 1)
            if (list[j] > list[j + 1]) { val t = list[j]; list[j] = list[j+1]; list[j+1] = t }
}

val quickSort: SortStrategy = { list -> list.sort() }

class DataProcessor(private val strategy: SortStrategy) {
    fun process(data: MutableList<Int>) = strategy(data)
}

val processor = DataProcessor(quickSort)
val data = mutableListOf(3, 1, 4, 1, 5, 9, 2, 6)
processor.process(data)
println(data)  // [1, 1, 2, 3, 4, 5, 6, 9]
```

---

## 3. 데코레이터 패턴 — 함수 래핑

```kotlin
// 로깅 데코레이터
fun <T, R> withLogging(name: String, fn: (T) -> R): (T) -> R = { input ->
    println("[$name] 시작: $input")
    val result = fn(input)
    println("[$name] 완료: $result")
    result
}

// 타이밍 데코레이터
fun <T, R> withTiming(fn: (T) -> R): (T) -> R = { input ->
    val start = System.currentTimeMillis()
    val result = fn(input)
    println("소요 시간: ${System.currentTimeMillis() - start}ms")
    result
}

// 재시도 데코레이터
fun <T, R> withRetry(maxRetries: Int, fn: (T) -> R): (T) -> R = { input ->
    var lastError: Exception? = null
    var result: R? = null
    repeat(maxRetries) { attempt ->
        try {
            result = fn(input)
            return@repeat
        } catch (e: Exception) {
            lastError = e
            println("시도 ${attempt + 1} 실패: ${e.message}")
        }
    }
    result ?: throw lastError!!
}

// 데코레이터 조합
val fetchUser: (Int) -> String = { id -> "User#$id" }

val decoratedFetch = withLogging("fetchUser",
    withTiming(
        withRetry(3, fetchUser)
    )
)

println(decoratedFetch(42))
```

---

## 4. 미들웨어 파이프라인

```kotlin
data class Request(val path: String, val headers: Map<String, String>, val body: String?)
data class Response(val status: Int, val body: String)

typealias Handler = (Request) -> Response
typealias Middleware = (Handler) -> Handler

// 인증 미들웨어
val authMiddleware: Middleware = { next ->
    { request ->
        if (request.headers["Authorization"] != null) {
            next(request)
        } else {
            Response(401, "Unauthorized")
        }
    }
}

// 로깅 미들웨어
val loggingMiddleware: Middleware = { next ->
    { request ->
        println("요청: ${request.path}")
        val response = next(request)
        println("응답: ${response.status}")
        response
    }
}

// 핸들러
val mainHandler: Handler = { request ->
    Response(200, "Hello from ${request.path}")
}

// 미들웨어 체이닝
fun chain(vararg middlewares: Middleware, handler: Handler): Handler =
    middlewares.foldRight(handler) { middleware, acc -> middleware(acc) }

val app = chain(loggingMiddleware, authMiddleware, handler = mainHandler)

val request = Request("/api/users", mapOf("Authorization" to "Bearer token"), null)
println(app(request))
```

---

## 5. 파이프라인 패턴

```kotlin
// 데이터 변환 파이프라인
class Pipeline<T>(private val value: T) {
    fun <R> pipe(transform: (T) -> R): Pipeline<R> = Pipeline(transform(value))
    fun get(): T = value
}

fun <T> T.pipeline() = Pipeline(this)

val result = "  Hello, Kotlin!  "
    .pipeline()
    .pipe { it.trim() }
    .pipe { it.lowercase() }
    .pipe { it.replace(",", "") }
    .pipe { it.split(" ") }
    .pipe { it.filter { word -> word.length > 3 } }
    .get()

println(result)  // [hello, kotlin!]

// 함수 리스트로 파이프라인
fun <T> pipeline(vararg steps: (T) -> T): (T) -> T =
    steps.reduce { f, g -> { x -> g(f(x)) } }

val textProcess = pipeline<String>(
    String::trim,
    String::lowercase,
    { it.replace("  ", " ") }
)

println(textProcess("  HELLO   WORLD  "))  // hello world
```

---

## 6. 커맨드 패턴 — 함수로 명령 저장

```kotlin
class CommandHistory {
    private val history = ArrayDeque<() -> Unit>()
    private val undoStack = ArrayDeque<() -> Unit>()

    fun execute(command: () -> Unit, undo: () -> Unit) {
        command()
        history.addLast(command)
        undoStack.addLast(undo)
    }

    fun undo() {
        undoStack.removeLastOrNull()?.invoke()
        history.removeLastOrNull()
    }
}

val document = StringBuilder()
val history = CommandHistory()

history.execute(
    command = { document.append("Hello") },
    undo = { document.delete(document.length - 5, document.length) }
)
history.execute(
    command = { document.append(", World!") },
    undo = { document.delete(document.length - 8, document.length) }
)

println(document)  // Hello, World!
history.undo()
println(document)  // Hello
history.undo()
println(document)  // (빈 문자열)
```

---

## 7. 실전 패턴 — 유효성 검사 체인

```kotlin
typealias Validator<T> = (T) -> String?  // null이면 통과, String이면 에러 메시지

fun <T> allOf(vararg validators: Validator<T>): Validator<T> = { value ->
    validators.firstNotNullOfOrNull { it(value) }
}

val notBlank: Validator<String> = { if (it.isBlank()) "빈 값 불가" else null }
val minLength: (Int) -> Validator<String> = { min -> { if (it.length < min) "${min}자 이상 필요" else null } }
val noSpecialChars: Validator<String> = { if (it.any { c -> !c.isLetterOrDigit() }) "특수문자 불가" else null }

val usernameValidator = allOf(notBlank, minLength(3), noSpecialChars)

println(usernameValidator(""))          // 빈 값 불가
println(usernameValidator("ab"))        // 3자 이상 필요
println(usernameValidator("alice@"))    // 특수문자 불가
println(usernameValidator("alice123"))  // null (통과)
```

---

## 8. 정리

- **전략 패턴**: `typealias Strategy = (Input) -> Output` — 클래스 계층 불필요
- **데코레이터**: `(T) -> R`을 받아 `(T) -> R`을 반환하는 함수로 기능 추가
- **미들웨어**: `(Handler) -> Handler` 타입으로 요청/응답 체이닝
- **파이프라인**: `foldRight`로 함수 합성, 데이터 순차 변환
- **커맨드**: 실행과 취소를 함수 쌍으로 저장
- 고차 함수 패턴은 클래스 기반보다 간결하고 테스트하기 쉬움
