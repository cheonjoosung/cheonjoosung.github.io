---
title: (Kotlin/코틀린) Arrow 라이브러리 기초 — Either, Option
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin 함수형 프로그래밍 라이브러리 Arrow의 Either와 Option 타입으로 에러 처리와 null 안전성을 다루는 방법을 정리합니다.
---

---

## 1. Arrow란?

Kotlin을 위한 함수형 프로그래밍 라이브러리입니다.  
`Either`, `Option`, `IO`, `Raise` 등 함수형 타입과 패턴을 제공합니다.

```kotlin
// build.gradle.kts
implementation("io.arrow-kt:arrow-core:1.2.4")
```

---

## 2. Either — 성공 또는 실패

`Either<Left, Right>`는 두 가지 값 중 하나를 가집니다.  
관례상 **Left = 실패/에러**, **Right = 성공**입니다.

```kotlin
import arrow.core.Either
import arrow.core.left
import arrow.core.right

// Either 생성
val success: Either<String, Int> = 42.right()
val failure: Either<String, Int> = "에러 발생".left()

// 값 처리
when (success) {
    is Either.Left  -> println("실패: ${success.value}")
    is Either.Right -> println("성공: ${success.value}")
}

// fold로 처리
val result = success.fold(
    ifLeft  = { error -> "실패: $error" },
    ifRight = { value -> "성공: $value" }
)
println(result)  // 성공: 42
```

---

## 3. Either vs Result vs sealed class

```kotlin
// Kotlin 내장 Result
val r1: Result<Int> = Result.success(42)
val r1f: Result<Int> = Result.failure(RuntimeException("에러"))

// sealed class
sealed class ApiResult<out T> {
    data class Success<T>(val data: T) : ApiResult<T>()
    data class Error(val message: String) : ApiResult<Nothing>()
}

// Either — 에러 타입도 명시적으로 지정 가능
sealed class DomainError {
    object NotFound : DomainError()
    data class NetworkError(val code: Int) : DomainError()
    data class ValidationError(val field: String) : DomainError()
}

suspend fun getUser(id: Int): Either<DomainError, User> =
    if (id > 0) User(id, "Alice").right()
    else DomainError.NotFound.left()
```

---

## 4. Either 체이닝 — map, flatMap

```kotlin
fun parseId(input: String): Either<String, Int> =
    input.toIntOrNull()?.right() ?: "숫자가 아닙니다: $input".left()

fun getUser(id: Int): Either<String, User> =
    if (id > 0) User(id, "User#$id").right()
    else "ID는 양수여야 합니다".left()

fun getEmail(user: User): Either<String, String> =
    "${user.name.lowercase()}@example.com".right()

// flatMap 체이닝: 하나라도 실패하면 즉시 Left 반환
val result = parseId("42")
    .flatMap { id -> getUser(id) }
    .flatMap { user -> getEmail(user) }
    .map { email -> email.uppercase() }

println(result)  // Right(USER#42@EXAMPLE.COM)

val failed = parseId("abc")
    .flatMap { id -> getUser(id) }
// Left(숫자가 아닙니다: abc) — 이후 단계 실행 안 됨
```

---

## 5. Either.catch — 예외를 Either로 변환

```kotlin
import arrow.core.Either

// 예외를 Either로 감싸기
val result: Either<Throwable, Int> = Either.catch {
    "123".toInt()  // 성공
}

val failed: Either<Throwable, Int> = Either.catch {
    "abc".toInt()  // NumberFormatException 발생
}

// 에러 타입 변환
val mapped = Either.catch { "abc".toInt() }
    .mapLeft { it.message ?: "알 수 없는 에러" }
// Left("For input string: \"abc\"")
```

---

## 6. Option — null 대신 명시적 부재

`Option<T>`는 값이 있거나(`Some<T>`) 없음(`None`)을 명시적으로 표현합니다.

```kotlin
import arrow.core.Option
import arrow.core.Some
import arrow.core.None
import arrow.core.toOption

// Option 생성
val some: Option<String> = Some("Hello")
val none: Option<String> = None

// nullable → Option
val value: Option<String> = "kotlin".toOption()
val nullValue: Option<String> = null.toOption()  // None

// 값 처리
when (some) {
    is Some -> println("값: ${some.value}")
    is None -> println("값 없음")
}

// getOrElse
println(some.getOrElse { "기본값" })  // Hello
println(none.getOrElse { "기본값" })  // 기본값
```

---

## 7. Option 체이닝

```kotlin
data class Config(val database: DatabaseConfig?)
data class DatabaseConfig(val host: String?)

val config = Config(DatabaseConfig("localhost"))

// Option 체이닝
val host: Option<String> = config.database.toOption()
    .flatMap { it.host.toOption() }
    .map { it.uppercase() }

println(host)  // Some(LOCALHOST)

// null 체이닝과 비교
val hostNullable: String? = config.database?.host?.uppercase()
// 두 방식 모두 동일한 결과지만 Option은 타입에 의도가 명확히 드러남
```

---

## 8. Raise DSL — 명령형 스타일 에러 처리

Arrow 1.2+의 `Raise` DSL은 `Either`를 명령형처럼 작성합니다.

```kotlin
import arrow.core.raise.either
import arrow.core.raise.ensure
import arrow.core.raise.ensureNotNull

data class User(val id: Int, val name: String, val email: String)

fun createUser(id: Int, name: String, email: String): Either<String, User> = either {
    ensure(id > 0) { "ID는 양수여야 합니다" }
    ensure(name.isNotBlank()) { "이름은 필수입니다" }
    ensure(email.contains("@")) { "이메일 형식이 잘못됐습니다" }

    User(id, name, email)  // 모든 조건 통과 시 반환
}

println(createUser(1, "Alice", "alice@example.com"))
// Right(User(id=1, name=Alice, email=alice@example.com))

println(createUser(-1, "Alice", "alice@example.com"))
// Left(ID는 양수여야 합니다)

println(createUser(1, "", "alice@example.com"))
// Left(이름은 필수입니다)
```

---

## 9. Arrow vs Kotlin 내장 기능

| 상황 | Kotlin 내장 | Arrow |
|------|------------|-------|
| 에러 처리 | `Result<T>`, sealed class | `Either<E, T>` |
| null 처리 | nullable `T?` | `Option<T>` |
| 에러 타입 명시 | 불가 (Result는 Throwable 고정) | 가능 (Any 타입) |
| 함수 합성 | 직접 구현 | `map`, `flatMap`, `zip` |
| 명령형 스타일 | try-catch | `either { }` DSL |

**단순한 앱**: Kotlin 내장 기능만으로 충분  
**복잡한 에러 처리, 함수형 조합**: Arrow 도입 고려

---

## 10. 정리

- **`Either<L, R>`**: 실패(Left) 또는 성공(Right) — 에러 타입을 명시적으로 표현
- **`Option<T>`**: `Some(value)` 또는 `None` — null의 명시적 대안
- `flatMap` 체이닝으로 실패 시 즉시 단락(short-circuit)
- `Either.catch { }` 로 예외를 Either로 변환
- `either { }` DSL로 명령형 스타일 에러 처리 가능
- 간단한 프로젝트는 sealed class + Result로도 충분
