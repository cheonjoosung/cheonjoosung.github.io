---
title: (Kotlin/코틀린) kotlinx.serialization 기초 완전 정리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin 공식 직렬화 라이브러리 kotlinx.serialization의 기본 사용법과 JSON 인코딩/디코딩, @Serializable 어노테이션 활용법을 정리합니다.
---

---

## 1. kotlinx.serialization이란?

JetBrains가 공식 제공하는 Kotlin 직렬화 라이브러리입니다.  
Kotlin 타입 시스템과 완벽하게 통합되어 **컴파일 타임에 직렬화 코드를 자동 생성**합니다.

```kotlin
// Gson/Moshi와의 핵심 차이
// Gson: 리플렉션 기반 → 런타임 처리, ProGuard 주의
// kotlinx.serialization: 컴파일 타임 코드 생성 → 빠르고 안전
```

---

## 2. 의존성 추가

```kotlin
// build.gradle.kts (프로젝트 수준)
plugins {
    kotlin("plugin.serialization") version "1.9.0"
}

// build.gradle.kts (앱 수준)
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.0")
}
```

---

## 3. 기본 사용 — @Serializable

```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json
import kotlinx.serialization.encodeToString
import kotlinx.serialization.decodeFromString

@Serializable
data class User(
    val id: Int,
    val name: String,
    val email: String
)

// 직렬화 (객체 → JSON 문자열)
val user = User(1, "Alice", "alice@example.com")
val json = Json.encodeToString(user)
println(json)
// {"id":1,"name":"Alice","email":"alice@example.com"}

// 역직렬화 (JSON 문자열 → 객체)
val decoded = Json.decodeFromString<User>(json)
println(decoded)
// User(id=1, name=Alice, email=alice@example.com)
```

---

## 4. 컬렉션 직렬화

```kotlin
@Serializable
data class Product(val id: Int, val name: String, val price: Double)

// List
val products = listOf(
    Product(1, "노트북", 1200000.0),
    Product(2, "마우스", 35000.0)
)

val jsonList = Json.encodeToString(products)
// [{"id":1,"name":"노트북","price":1200000.0},{"id":2,"name":"마우스","price":35000.0}]

val decodedList = Json.decodeFromString<List<Product>>(jsonList)

// Map
val productMap = mapOf("A" to Product(1, "키보드", 85000.0))
val jsonMap = Json.encodeToString(productMap)
// {"A":{"id":1,"name":"키보드","price":85000.0}}
```

---

## 5. nullable과 기본값

```kotlin
@Serializable
data class Profile(
    val id: Int,
    val name: String,
    val bio: String? = null,          // nullable, JSON에 없으면 null
    val followerCount: Int = 0,       // 기본값, JSON에 없으면 0
    val isVerified: Boolean = false
)

// JSON에 일부 필드가 없어도 기본값으로 처리
val json = """{"id":1,"name":"Bob"}"""
val profile = Json.decodeFromString<Profile>(json)
println(profile)
// Profile(id=1, name=Bob, bio=null, followerCount=0, isVerified=false)

// null 필드를 JSON에서 제외 (기본값은 포함)
val configured = Json { explicitNulls = false }
val encoded = configured.encodeToString(profile)
// {"id":1,"name":"Bob","followerCount":0,"isVerified":false}
```

---

## 6. Json 설정 커스터마이징

```kotlin
val json = Json {
    prettyPrint = true         // 들여쓰기 포함 출력
    ignoreUnknownKeys = true   // 모르는 키 무시 (역직렬화 시)
    isLenient = true           // 느슨한 JSON 파싱 (따옴표 없는 문자열 등)
    encodeDefaults = true      // 기본값도 JSON에 포함
    explicitNulls = false      // null 필드 JSON 출력 제외
    coerceInputValues = true   // 잘못된 값을 기본값으로 대체
}

val user = User(1, "Alice", "alice@example.com")
println(json.encodeToString(user))
// {
//     "id": 1,
//     "name": "Alice",
//     "email": "alice@example.com"
// }
```

---

## 7. 중첩 클래스

```kotlin
@Serializable
data class Address(
    val street: String,
    val city: String,
    val zipCode: String
)

@Serializable
data class Order(
    val orderId: String,
    val customer: User,
    val shippingAddress: Address,
    val items: List<Product>
)

val order = Order(
    orderId = "ORD-001",
    customer = User(1, "Alice", "alice@example.com"),
    shippingAddress = Address("강남대로 1", "서울", "06100"),
    items = listOf(Product(1, "노트북", 1200000.0))
)

val json = Json { prettyPrint = true }
println(json.encodeToString(order))
```

중첩된 모든 클래스에 `@Serializable`이 필요합니다.

---

## 8. enum 직렬화

```kotlin
@Serializable
enum class Status {
    PENDING, ACTIVE, INACTIVE
}

@Serializable
data class Account(val id: Int, val status: Status)

val account = Account(1, Status.ACTIVE)
val json = Json.encodeToString(account)
println(json)  // {"id":1,"status":"ACTIVE"}

val decoded = Json.decodeFromString<Account>(json)
println(decoded.status)  // ACTIVE
```

---

## 9. 실전 — Retrofit과 함께 사용

```kotlin
// build.gradle.kts
implementation("com.jakewharton.retrofit2:retrofit2-kotlinx-serialization-converter:1.0.0")

// Retrofit 설정
val contentType = "application/json".toMediaType()
val retrofit = Retrofit.Builder()
    .baseUrl("https://api.example.com/")
    .addConverterFactory(Json.asConverterFactory(contentType))
    .build()

// API 인터페이스
interface UserApi {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Int): User

    @POST("users")
    suspend fun createUser(@Body user: User): User
}
```

---

## 10. 정리

- `@Serializable` 어노테이션 하나로 직렬화/역직렬화 코드 자동 생성
- `Json.encodeToString()` / `Json.decodeFromString<T>()` 기본 API
- nullable, 기본값 완벽 지원 — JSON 필드 없어도 기본값 적용
- `Json { }` 블록으로 prettyPrint, ignoreUnknownKeys 등 세밀한 설정
- 중첩 클래스, 컬렉션, enum 모두 지원
- Retrofit과 컨버터 팩토리로 쉽게 통합 가능
