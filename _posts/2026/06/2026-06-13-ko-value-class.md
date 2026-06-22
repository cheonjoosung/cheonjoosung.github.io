---
title: (Kotlin/코틀린) Value class(Inline class) — 타입 안전성과 성능
tags: [ Kotlin, Android ]
style: fill
color: dark
description: Kotlin value class(인라인 클래스)의 개념, 타입 안전성 확보 방법, 런타임 성능 이점, 제약 사항과 Android 실전 활용까지 정리합니다.
---

## 개요

- Kotlin의 **`value class`(인라인 클래스)** 를 다룹니다.
- 단일 프로퍼티를 감싸면서도 **런타임에는 래핑 객체를 생성하지 않는** 특수한 클래스입니다.
- 이 글에서는 다음을 설명합니다.
  - `value class`가 필요한 이유 — Primitive Obsession 문제
  - 선언 방법과 제약 사항
  - 컴파일 시점 인라이닝 동작
  - Android 실전 예제

---

## 1. 왜 필요한가 — Primitive Obsession

```kotlin
// ❌ 문제 — 모두 String/Long이라 타입으로 구분되지 않음
fun sendMessage(userId: String, roomId: String, message: String) { /* ... */ }

// 인자 순서를 잘못 넣어도 컴파일 에러 없음
sendMessage(roomId = "room1", userId = "user1", message = "hi")  // 의도와 다름
sendMessage("room1", "user1", "hi")  // 순서 실수 — 컴파일러가 못 잡음
```

```kotlin
// ✅ data class로 감싸면 타입은 안전하지만 객체 생성 비용 발생
data class UserId(val value: String)
data class RoomId(val value: String)

fun sendMessage(userId: UserId, roomId: RoomId, message: String) { /* ... */ }
// 순서를 바꾸면 컴파일 에러 — 타입 안전
// 하지만 호출마다 UserId, RoomId 객체가 힙에 생성됨
```

```kotlin
// ✅✅ value class — 타입 안전 + 런타임 래핑 없음
@JvmInline
value class UserId(val value: String)

@JvmInline
value class RoomId(val value: String)

fun sendMessage(userId: UserId, roomId: RoomId, message: String) { /* ... */ }
sendMessage(UserId("user1"), RoomId("room1"), "hi")
// 컴파일 후에는 String만 사용하는 것과 동일한 성능
```

---

## 2. 선언 방법과 제약

```kotlin
@JvmInline
value class Age(val value: Int) {
    init {
        require(value >= 0) { "나이는 음수일 수 없습니다" }
    }
}

@JvmInline
value class Email(val value: String) {
    init {
        require(value.contains("@")) { "잘못된 이메일 형식" }
    }

    fun domain(): String = value.substringAfter("@")
}

val age = Age(25)
val email = Email("user@example.com")
println(email.domain())  // example.com
```

### 제약 사항

```kotlin
// ① 프로퍼티는 정확히 1개만 가능
@JvmInline
value class Point(val x: Int, val y: Int)  // ❌ 컴파일 에러

// ② 주 생성자 프로퍼티는 val만 가능 (var 불가)
@JvmInline
value class Score(var value: Int)  // ❌ 컴파일 에러

// ③ 상속 불가 — 인터페이스 구현은 가능
interface Identifiable { val id: String }

@JvmInline
value class UserId(val id: String) : Identifiable  // ✅ 가능

// ④ init 블록, 함수, 계산된 프로퍼티는 가능
@JvmInline
value class Percentage(val value: Double) {
    init { require(value in 0.0..100.0) }
    val isComplete: Boolean get() = value == 100.0
}
```

---

## 3. 컴파일 시점 인라이닝 동작

```kotlin
@JvmInline
value class UserId(val value: String)

fun printUserId(id: UserId) {
    println(id.value)
}

// 컴파일 후 (개념적으로) ↓
// fun printUserId(id: String) {
//     println(id)
// }
```

```kotlin
// 단, 다음 경우는 실제 박싱(boxing)이 발생함
// ① nullable 타입으로 사용할 때
fun process(id: UserId?) { /* ... */ }   // UserId 객체로 박싱됨

// ② 제네릭 타입 인자로 사용할 때
val list: List<UserId> = listOf(UserId("a"))  // 박싱됨

// ③ 다른 타입(인터페이스)으로 취급될 때
val identifiable: Identifiable = UserId("a")  // 박싱됨
```

---

## 4. value class vs data class 비교

| 항목 | `value class` | `data class` |
|------|----------------|---------------|
| 프로퍼티 수 | 1개만 | 여러 개 가능 |
| 런타임 객체 생성 | 인라이닝되어 생성 안 됨 (일반적인 경우) | 항상 생성 |
| equals/hashCode/toString | 자동 생성 | 자동 생성 |
| 주 사용처 | 단일 값 타입 안전성 (ID, 단위) | 여러 필드를 가진 모델 |
| 성능 | 박싱 없을 시 Primitive와 동일 | 객체 생성 비용 존재 |

---

## 5. Android 실전 예제

### 단위 혼동 방지

```kotlin
@JvmInline
value class Px(val value: Float)

@JvmInline
value class Dp(val value: Float)

fun Dp.toPx(density: Float): Px = Px(value * density)

// ❌ 실수 방지 — Px와 Dp를 섞어 쓸 수 없음
fun setMargin(margin: Px) { view.setPadding(margin.value.toInt(), 0, 0, 0) }

val marginDp = Dp(16f)
// setMargin(marginDp)               // ❌ 컴파일 에러 — 타입 다름
setMargin(marginDp.toPx(density = 2.0f))  // ✅
```

### ID 타입 안전성

```kotlin
@JvmInline
value class UserId(val value: Long)

@JvmInline
value class PostId(val value: Long)

interface PostRepository {
    suspend fun getPostsByUser(userId: UserId): List<Post>
    suspend fun getPost(postId: PostId): Post?
}

// userId, postId 순서를 헷갈려도 컴파일러가 잡아줌
class PostRepositoryImpl(private val api: PostApi) : PostRepository {
    override suspend fun getPostsByUser(userId: UserId): List<Post> =
        api.fetchPostsByUser(userId.value)

    override suspend fun getPost(postId: PostId): Post? =
        api.fetchPost(postId.value)
}
```

### 결과 코드 타입화

```kotlin
@JvmInline
value class HttpStatusCode(val code: Int) {
    val isSuccess: Boolean get() = code in 200..299
    val isClientError: Boolean get() = code in 400..499
    val isServerError: Boolean get() = code in 500..599
}

fun handleResponse(status: HttpStatusCode) {
    when {
        status.isSuccess      -> println("성공")
        status.isClientError  -> println("클라이언트 오류")
        status.isServerError  -> println("서버 오류")
    }
}
```

---

## 6. 정리

| 항목 | 내용 |
|------|------|
| 목적 | Primitive Obsession 해결 — 타입 안전성 확보, 성능 비용 최소화 |
| 선언 | `@JvmInline value class Name(val value: T)` |
| 제약 | 프로퍼티 1개, val만 가능, 상속 불가(인터페이스는 가능) |
| 성능 | 일반적인 경우 박싱 없음, nullable·제네릭·인터페이스 사용 시 박싱 |
| Android 활용 | 단위 혼동 방지(Px/Dp), ID 타입 분리, 상태 코드 래핑 |

- `value class`는 **"타입은 늘리고 객체는 늘리지 않는"** 방법입니다.
- ID, 단위, 코드처럼 의미가 다른 원시값을 구분할 때 적극 활용할 수 있습니다.

---

## 참고

- [Kotlin 공식 문서 — Inline classes](https://kotlinlang.org/docs/inline-classes.html)
- [StateFlow vs SharedFlow 포스팅 보기](/posts/2026-06-01-ko-stateflow-vs-sharedflow)
