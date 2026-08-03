---
title: (Kotlin/코틀린) kotlinx.serialization 심화 — @SerialName, Polymorphism
tags: [ Kotlin ]
style: fill
color: dark
description: kotlinx.serialization의 @SerialName, @Transient, 다형성(Polymorphism) 직렬화와 커스텀 Serializer 구현 방법을 정리합니다.
---

---

## 1. @SerialName — JSON 키 이름 변경

```kotlin
@Serializable
data class User(
    @SerialName("user_id") val id: Int,
    @SerialName("full_name") val name: String,
    @SerialName("email_address") val email: String
)

val user = User(1, "Alice", "alice@example.com")
println(Json.encodeToString(user))
// {"user_id":1,"full_name":"Alice","email_address":"alice@example.com"}

// 역직렬화: JSON의 user_id → 코드의 id
val json = """{"user_id":1,"full_name":"Bob","email_address":"bob@example.com"}"""
val decoded = Json.decodeFromString<User>(json)
println(decoded.id)  // 1
```

스네이크 케이스 API와 카멜 케이스 Kotlin 코드를 자연스럽게 연결합니다.

---

## 2. namingStrategy — 전역 이름 변환

모든 필드에 `@SerialName` 대신 전역 네이밍 전략을 설정할 수 있습니다.

```kotlin
@Serializable
data class Product(
    val productId: Int,
    val productName: String,
    val unitPrice: Double
)

val json = Json {
    namingStrategy = JsonNamingStrategy.SnakeCase
}

val product = Product(1, "노트북", 1200000.0)
println(json.encodeToString(product))
// {"product_id":1,"product_name":"노트북","unit_price":1200000.0}
```

---

## 3. @Transient — 직렬화 제외

```kotlin
@Serializable
data class Session(
    val userId: Int,
    val token: String,
    @Transient val cachedData: String = ""  // JSON에 포함되지 않음
)

val session = Session(1, "abc123", "some-cache")
println(Json.encodeToString(session))
// {"userId":1,"token":"abc123"}

// 역직렬화 시 @Transient 필드는 기본값으로 설정됨
val decoded = Json.decodeFromString<Session>("""{"userId":1,"token":"abc123"}""")
println(decoded.cachedData)  // "" (기본값)
```

`@Transient`는 반드시 기본값이 있어야 합니다.

---

## 4. 다형성 직렬화 — sealed class

```kotlin
@Serializable
sealed class Shape {
    abstract val area: Double
}

@Serializable
data class Circle(val radius: Double) : Shape() {
    override val area = Math.PI * radius * radius
}

@Serializable
data class Rectangle(val width: Double, val height: Double) : Shape() {
    override val area = width * height
}
```

```kotlin
val json = Json { prettyPrint = true }

val shapes: List<Shape> = listOf(
    Circle(5.0),
    Rectangle(3.0, 4.0)
)

val encoded = json.encodeToString(shapes)
println(encoded)
// [
//     {"type":"Circle","radius":5.0,"area":78.53...},
//     {"type":"Rectangle","width":3.0,"height":4.0,"area":12.0}
// ]

val decoded = Json.decodeFromString<List<Shape>>(encoded)
println(decoded[0])  // Circle(radius=5.0)
println(decoded[1])  // Rectangle(width=3.0, height=4.0)
```

sealed class는 타입 정보(`type` 필드)가 자동으로 포함됩니다.

---

## 5. @SerialName으로 타입 키 변경

```kotlin
@Serializable
sealed class ApiResult {
    @Serializable
    @SerialName("success")
    data class Success(val data: String) : ApiResult()

    @Serializable
    @SerialName("error")
    data class Error(val code: Int, val message: String) : ApiResult()
}

val success = ApiResult.Success("응답 데이터")
println(Json.encodeToString<ApiResult>(success))
// {"type":"success","data":"응답 데이터"}

val error = ApiResult.Error(404, "Not Found")
println(Json.encodeToString<ApiResult>(error))
// {"type":"error","code":404,"message":"Not Found"}
```

---

## 6. classDiscriminator — 타입 키 이름 변경

기본 타입 구분자는 `"type"`이지만 변경할 수 있습니다.

```kotlin
@Serializable
@JsonClassDiscriminator("kind")
sealed class Event {
    @Serializable
    @SerialName("click")
    data class Click(val x: Int, val y: Int) : Event()

    @Serializable
    @SerialName("scroll")
    data class Scroll(val delta: Int) : Event()
}

val event: Event = Event.Click(100, 200)
println(Json.encodeToString(event))
// {"kind":"click","x":100,"y":200}
```

---

## 7. 커스텀 Serializer

직렬화 방식을 완전히 제어해야 할 때 `KSerializer`를 직접 구현합니다.

```kotlin
import kotlinx.serialization.*
import kotlinx.serialization.descriptors.*
import kotlinx.serialization.encoding.*

// LocalDate 직렬화 예시
object LocalDateSerializer : KSerializer<java.time.LocalDate> {
    override val descriptor: SerialDescriptor =
        PrimitiveSerialDescriptor("LocalDate", PrimitiveKind.STRING)

    override fun serialize(encoder: Encoder, value: java.time.LocalDate) {
        encoder.encodeString(value.toString())
    }

    override fun deserialize(decoder: Decoder): java.time.LocalDate {
        return java.time.LocalDate.parse(decoder.decodeString())
    }
}

@Serializable
data class Event(
    val name: String,
    @Serializable(with = LocalDateSerializer::class)
    val date: java.time.LocalDate
)

val event = Event("회의", java.time.LocalDate.of(2026, 7, 24))
println(Json.encodeToString(event))
// {"name":"회의","date":"2026-07-24"}
```

---

## 8. contextual 직렬화 — 전역 등록

```kotlin
val json = Json {
    serializersModule = SerializersModule {
        contextual(java.time.LocalDate::class, LocalDateSerializer)
    }
}

@Serializable
data class Schedule(
    @Contextual val date: java.time.LocalDate,
    val title: String
)
```

---

## 9. 정리

- **`@SerialName`**: JSON 키와 Kotlin 프로퍼티 이름 분리 (스네이크 케이스 API 대응)
- **`namingStrategy`**: 전역 네이밍 전략 설정으로 `@SerialName` 반복 제거
- **`@Transient`**: 직렬화 제외 필드 (기본값 필수)
- **sealed class**: 다형성 직렬화 자동 지원, `type` 필드로 타입 구분
- **커스텀 Serializer**: `KSerializer` 구현으로 표준 라이브러리 외 타입 지원
- `contextual`과 `SerializersModule`로 외부 타입을 전역 등록 가능
