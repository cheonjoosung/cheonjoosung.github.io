---
title: (Kotlin/코틀린) 패키지 레벨 함수 vs 싱글턴 object 선택 기준
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 최상위(패키지 레벨) 함수와 싱글턴 object의 차이점, 각각 적합한 사용 시나리오와 Java 상호운용 시 주의사항을 정리합니다.
---

---

## 1. 패키지 레벨 함수

클래스 없이 파일 최상위에 선언하는 함수입니다.

```kotlin
// StringUtils.kt
package com.example.utils

fun String.isValidEmail(): Boolean =
    contains("@") && contains(".")

fun formatPhone(number: String): String =
    number.replace(Regex("[^0-9]"), "")
        .let { "+82-${it.substring(1)}" }

fun generateId(): String = java.util.UUID.randomUUID().toString()
```

```kotlin
// 사용 — import만 하면 바로 사용
import com.example.utils.formatPhone
import com.example.utils.generateId

val phone = formatPhone("010-1234-5678")
val id = generateId()
```

---

## 2. 싱글턴 object

전역적으로 하나만 존재하는 인스턴스입니다.

```kotlin
object Logger {
    private var level = LogLevel.INFO
    private val logs = mutableListOf<String>()

    fun setLevel(newLevel: LogLevel) { level = newLevel }

    fun log(message: String) {
        val entry = "[${level}] $message"
        logs.add(entry)
        println(entry)
    }

    fun getLogs(): List<String> = logs.toList()
}

// 사용
Logger.setLevel(LogLevel.DEBUG)
Logger.log("앱 시작")
```

---

## 3. 핵심 차이

| 구분 | 패키지 레벨 함수 | 싱글턴 object |
|------|----------------|--------------|
| 상태 | 없음 (순수 함수에 적합) | 있음 (멤버 변수 보유 가능) |
| 인스턴스 | 없음 | 하나의 인스턴스 |
| 초기화 | 없음 | 처음 접근 시 한 번 |
| 인터페이스 구현 | 불가 | 가능 |
| 테스트 | 쉬움 (의존성 없음) | 어려움 (상태 공유) |
| Java 호출 | `파일명Kt.함수명()` | `ObjectName.함수명()` |

---

## 4. 패키지 레벨 함수가 적합한 경우

```kotlin
// ✅ 순수 유틸리티 함수 — 상태 없음
fun Int.toKoreanWon(): String = "${this.toString().reversed().chunked(3).joinToString(",").reversed()}원"

fun List<Double>.standardDeviation(): Double {
    val mean = average()
    return Math.sqrt(map { (it - mean) * (it - mean) }.average())
}

fun String.camelToSnake(): String =
    replace(Regex("([A-Z])")) { "_${it.value.lowercase()}" }.trimStart('_')

// ✅ 확장 함수 — 특정 타입에 기능 추가
fun Context.dp(value: Int): Int = (value * resources.displayMetrics.density).toInt()
fun View.visible() { visibility = View.VISIBLE }
fun View.gone() { visibility = View.GONE }
```

---

## 5. 싱글턴 object가 적합한 경우

```kotlin
// ✅ 상태가 필요한 전역 관리자
object SessionManager {
    private var currentUser: User? = null
    private var token: String? = null

    fun login(user: User, token: String) {
        currentUser = user
        this.token = token
    }

    fun logout() {
        currentUser = null
        token = null
    }

    fun isLoggedIn() = currentUser != null
    fun getCurrentUser() = currentUser
}

// ✅ 인터페이스를 구현해야 하는 경우
object DefaultErrorHandler : ErrorHandler {
    override fun handle(error: Throwable) {
        println("에러: ${error.message}")
    }
}

// ✅ 상수/팩토리 메서드 모음
object NetworkConfig {
    const val BASE_URL = "https://api.example.com"
    const val TIMEOUT = 30L
    const val MAX_RETRY = 3

    fun createHeaders(token: String?) = buildMap {
        put("Content-Type", "application/json")
        token?.let { put("Authorization", "Bearer $it") }
    }
}
```

---

## 6. companion object — 클래스에 속한 싱글턴

```kotlin
class User private constructor(
    val id: Int,
    val name: String
) {
    companion object {
        // 팩토리 메서드
        fun create(id: Int, name: String): User {
            require(id > 0) { "ID는 양수여야 합니다" }
            require(name.isNotBlank()) { "이름은 필수입니다" }
            return User(id, name)
        }

        // 상수
        const val MAX_NAME_LENGTH = 50
    }
}

val user = User.create(1, "Alice")
println(User.MAX_NAME_LENGTH)  // 50
```

---

## 7. Java 상호운용

```kotlin
// 패키지 레벨 함수: Java에서 파일명Kt.메서드명()으로 호출
// StringUtils.kt → Java: StringUtilsKt.formatPhone("...")

// @JvmName으로 파일 클래스 이름 변경
@file:JvmName("StringUtils")
package com.example.utils
// Java: StringUtils.formatPhone("...")

// object: Java에서 ObjectName.INSTANCE.메서드명() 또는 @JvmStatic
object Config {
    @JvmStatic val BASE_URL = "https://api.example.com"
    @JvmField val TIMEOUT = 30L
}
// Java: Config.BASE_URL (static 접근)
```

---

## 8. 테스트 관점

```kotlin
// 패키지 레벨 함수: 테스트 쉬움
@Test
fun `camelToSnake converts correctly`() {
    assertEquals("hello_world", "helloWorld".camelToSnake())
}

// object (상태 있음): 테스트마다 초기화 필요
@Before
fun setUp() {
    // 테스트 간 상태 공유로 인한 오염 방지
    SessionManager.logout()
}

@Test
fun `login sets current user`() {
    SessionManager.login(User(1, "Alice"), "token")
    assertTrue(SessionManager.isLoggedIn())
}
```

---

## 9. 선택 가이드

```
상태(필드)가 필요한가?
    ↓ Yes → object (또는 class + DI)
    ↓ No  → 패키지 레벨 함수

인터페이스를 구현해야 하는가?
    ↓ Yes → object
    ↓ No  → 패키지 레벨 함수 가능

특정 클래스와 논리적으로 연관됐는가?
    ↓ Yes → companion object
    ↓ No  → 패키지 레벨 함수 또는 독립 object

테스트에서 Mock이 필요한가?
    ↓ Yes → interface + 구현 클래스 (object 보다 유연)
    ↓ No  → object 또는 패키지 레벨 함수
```

---

## 10. 정리

- **패키지 레벨 함수**: 상태 없는 순수 유틸리티, 확장 함수 — 테스트 쉽고 간결
- **`object`**: 상태 있는 전역 싱글턴, 인터페이스 구현 필요 시
- **`companion object`**: 클래스에 속한 팩토리 메서드, 상수 정의
- Java 호출 시 패키지 레벨 함수는 `@file:JvmName`, object는 `@JvmStatic` 권장
- 상태가 없다면 패키지 레벨 함수가 더 간결하고 테스트하기 쉬움
