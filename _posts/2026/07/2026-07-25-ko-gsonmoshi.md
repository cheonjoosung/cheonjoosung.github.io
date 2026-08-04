---
title: (Kotlin/코틀린) Gson vs Moshi vs kotlinx.serialization 완전 비교
tags: [ Kotlin ]
style: fill
color: dark
description: Android/Kotlin 프로젝트에서 자주 사용하는 Gson, Moshi, kotlinx.serialization 세 가지 JSON 라이브러리의 특징, 성능, 장단점을 비교 정리합니다.
---

---

## 1. 세 라이브러리 한눈에 비교

| 구분 | Gson | Moshi | kotlinx.serialization |
|------|------|-------|-----------------------|
| 개발사 | Google | Square | JetBrains |
| 처리 방식 | 리플렉션 | 리플렉션 + 코드젠 | 컴파일 타임 코드젠 |
| Kotlin 지원 | 부분적 | 좋음 | 완벽 |
| nullable 처리 | 위험 | 안전 | 안전 |
| 성능 | 보통 | 빠름 | 가장 빠름 |
| 바이너리 크기 | 중간 | 작음 | 작음 |
| ProGuard/R8 | 주의 필요 | 주의 필요 | 안전 |
| Multiplatform | ✗ | ✗ | ✓ |

---

## 2. Gson

### 기본 사용

```kotlin
// build.gradle.kts
implementation("com.google.code.gson:gson:2.10.1")
```

```kotlin
import com.google.gson.Gson
import com.google.gson.annotations.SerializedName

data class User(
    @SerializedName("user_id") val id: Int,
    val name: String,
    val email: String
)

val gson = Gson()

// 직렬화
val user = User(1, "Alice", "alice@example.com")
val json = gson.toJson(user)
// {"user_id":1,"name":"Alice","email":"alice@example.com"}

// 역직렬화
val decoded = gson.fromJson(json, User::class.java)
```

### Gson의 Kotlin 문제점

```kotlin
// 문제 1: non-null 프로퍼티에 null이 들어올 수 있음
data class User(val name: String)  // non-null 선언

val json = """{"name":null}"""
val user = gson.fromJson(json, User::class.java)
println(user.name)  // null! (NPE 위험)

// 문제 2: 기본 생성자가 없는 클래스는 초기화 블록 미실행
data class User(val name: String) {
    init { require(name.isNotBlank()) }  // 이 검증이 실행 안 됨
}
```

Gson은 리플렉션으로 `init` 블록을 우회하고 non-null을 null로 설정할 수 있습니다.

---

## 3. Moshi

### 기본 사용

```kotlin
// build.gradle.kts
implementation("com.squareup.moshi:moshi:1.15.0")
implementation("com.squareup.moshi:moshi-kotlin:1.15.0")
// 코드 생성 사용 시
kapt("com.squareup.moshi:moshi-kotlin-codegen:1.15.0")
```

```kotlin
import com.squareup.moshi.Json
import com.squareup.moshi.JsonClass
import com.squareup.moshi.Moshi
import com.squareup.moshi.kotlin.reflect.KotlinJsonAdapterFactory

@JsonClass(generateAdapter = true)  // 코드 생성 어댑터
data class User(
    @Json(name = "user_id") val id: Int,
    val name: String,
    val email: String
)

val moshi = Moshi.Builder()
    .addLast(KotlinJsonAdapterFactory())  // 리플렉션 사용 시
    .build()

val adapter = moshi.adapter(User::class.java)

val json = adapter.toJson(User(1, "Alice", "alice@example.com"))
val decoded = adapter.fromJson(json)
```

### Moshi의 장점

```kotlin
// Kotlin nullable 타입을 올바르게 처리
@JsonClass(generateAdapter = true)
data class Response(
    val data: String?,    // nullable → JSON에 null이어도 안전
    val count: Int = 0    // 기본값 → JSON에 없어도 OK
)

// 리플렉션 없이 빠른 코드젠 어댑터
// @JsonClass(generateAdapter = true) 로 컴파일 타임 생성
```

---

## 4. kotlinx.serialization

```kotlin
// build.gradle.kts
plugins { kotlin("plugin.serialization") version "1.9.0" }
implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.0")
```

```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json

@Serializable
data class User(
    @kotlinx.serialization.SerialName("user_id") val id: Int,
    val name: String,
    val email: String
)

val user = User(1, "Alice", "alice@example.com")
val json = Json.encodeToString(user)
val decoded = Json.decodeFromString<User>(json)
```

---

## 5. 성능 비교

```kotlin
// 1만 개 객체 직렬화 벤치마크 (참고용)
// Gson:   ~180ms
// Moshi:  ~90ms  (코드젠 어댑터)
// kotlinx: ~60ms (컴파일 타임 생성)

// 실제 앱에서는 큰 차이가 없을 수 있으나,
// 대량 데이터 처리 시 kotlinx가 유리
```

---

## 6. sealed class / 다형성 처리

```kotlin
// Gson: 별도 RuntimeTypeAdapterFactory 설정 필요 (복잡)
val gson = GsonBuilder()
    .registerTypeAdapterFactory(
        RuntimeTypeAdapterFactory.of(Shape::class.java, "type")
            .registerSubtype(Circle::class.java, "circle")
            .registerSubtype(Rectangle::class.java, "rectangle")
    ).build()

// Moshi: 별도 어댑터 또는 sealed-class 플러그인 필요
// build.gradle: implementation("dev.zacsweers.moshix:moshi-sealed-runtime:...")

// kotlinx.serialization: sealed class 기본 지원
@Serializable
sealed class Shape
@Serializable data class Circle(val r: Double) : Shape()
@Serializable data class Rect(val w: Double, val h: Double) : Shape()

// 즉시 사용 가능
Json.encodeToString<Shape>(Circle(5.0))
```

---

## 7. Kotlin Multiplatform 지원

```kotlin
// kotlinx.serialization만 KMP 지원
// Android, iOS, JS, Native 공유 코드에서 사용 가능

// commonMain
@Serializable
data class SharedModel(val id: Int, val name: String)

// Gson, Moshi는 JVM 전용 → KMP 불가
```

---

## 8. 선택 가이드

```
새 프로젝트 시작?
    ↓ Yes
→ kotlinx.serialization 권장
    - KMP 지원, 컴파일 타임 안전, 가장 빠름

기존 프로젝트에 Gson 사용 중?
    ↓ Yes
→ Gson 유지 (마이그레이션 비용 vs 이득 고려)
  또는 단계적으로 kotlinx로 전환

Retrofit과 함께 사용?
    - Gson: GsonConverterFactory (기본 제공)
    - Moshi: MoshiConverterFactory
    - kotlinx: kotlinx-serialization-converter (별도)

Kotlin 타입 안전성 중요?
    → Moshi 또는 kotlinx.serialization (Gson 피하기)

Android 전용 + 작은 앱?
    → Moshi (구글 개발자도 자주 추천)
```

---

## 9. 정리

- **Gson**: 가장 오래됨, Kotlin과 완벽하지 않음, 레거시 프로젝트에서 주로 사용
- **Moshi**: Kotlin 친화적, Square 제품, 코드젠 + 리플렉션 선택 가능
- **kotlinx.serialization**: JetBrains 공식, KMP 지원, 컴파일 타임 코드 생성, Kotlin 타입 완벽 지원
- 신규 프로젝트라면 `kotlinx.serialization` 또는 `Moshi` 권장
- Gson의 non-null 타입에 null 주입 문제는 런타임 NPE의 주요 원인
