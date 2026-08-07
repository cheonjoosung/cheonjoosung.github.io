---
title: (Kotlin/코틀린) Java 상호운용 — @JvmStatic, @JvmField, @JvmOverloads
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin 코드를 Java에서 자연스럽게 호출할 수 있도록 하는 @JvmStatic, @JvmField, @JvmOverloads 어노테이션과 Java 상호운용 핵심 규칙을 정리합니다.
---

---

## 1. Kotlin-Java 상호운용 개요

Kotlin은 JVM에서 Java와 100% 호환됩니다.  
하지만 Kotlin의 일부 기능은 Java에서 **불편하게** 보입니다.  
`@Jvm*` 어노테이션들이 이 간극을 좁혀줍니다.

---

## 2. @JvmStatic — 정적 메서드 노출

Kotlin의 `companion object`는 Java에서 `Companion.method()`로 호출됩니다.  
`@JvmStatic`을 붙이면 Java에서 `ClassName.method()`로 직접 호출할 수 있습니다.

```kotlin
class MathUtils {
    companion object {
        fun square(n: Int) = n * n                  // Java: MathUtils.Companion.square(5)

        @JvmStatic
        fun cube(n: Int) = n * n * n                // Java: MathUtils.cube(5)
    }
}
```

```java
// Java 코드
int a = MathUtils.Companion.square(5);  // @JvmStatic 없음
int b = MathUtils.cube(5);              // @JvmStatic 있음 — 더 자연스러움
```

```kotlin
// object 선언에서도 동일하게 사용
object Logger {
    @JvmStatic
    fun log(message: String) = println("[LOG] $message")
}
```

```java
Logger.log("Hello");  // 정적 메서드처럼 호출
```

---

## 3. @JvmField — 프로퍼티를 필드로 노출

Kotlin 프로퍼티는 기본적으로 private 필드 + getter/setter로 컴파일됩니다.  
`@JvmField`는 **getter/setter 없이 public 필드**로 직접 노출합니다.

```kotlin
class Config {
    @JvmField
    val MAX_SIZE = 100           // Java: config.MAX_SIZE (필드 직접 접근)

    val version = "1.0"          // Java: config.getVersion() (getter 통해 접근)
}
```

```java
Config config = new Config();
int size = config.MAX_SIZE;       // @JvmField: 필드 직접 접근
String ver = config.getVersion(); // 일반 프로퍼티: getter 사용
```

```kotlin
// companion object 상수에 자주 사용
class DatabaseHelper {
    companion object {
        @JvmField
        val DB_NAME = "app_database"

        @JvmField
        val DB_VERSION = 1
    }
}
```

```java
String name = DatabaseHelper.DB_NAME;  // 정적 필드처럼 접근
```

---

## 4. @JvmOverloads — 기본값 파라미터 오버로드 생성

Kotlin의 기본값 파라미터는 Java에서 인식되지 않습니다.  
`@JvmOverloads`는 **가능한 모든 파라미터 조합의 오버로드 메서드를 자동 생성**합니다.

```kotlin
// @JvmOverloads 없이 Java에서는 모든 인자를 반드시 전달해야 함
fun createUser(
    name: String,
    age: Int = 0,
    email: String = "",
    isActive: Boolean = true
): User

// Java: createUser("Alice", 0, "", true)  // 기본값 직접 입력 필요
```

```kotlin
@JvmOverloads
fun createUser(
    name: String,
    age: Int = 0,
    email: String = "",
    isActive: Boolean = true
): User = User(name, age, email, isActive)

// 자동 생성되는 Java 오버로드:
// createUser(String name)
// createUser(String name, int age)
// createUser(String name, int age, String email)
// createUser(String name, int age, String email, boolean isActive)
```

```java
// Java에서 자연스러운 호출
User u1 = createUser("Alice");
User u2 = createUser("Bob", 25);
User u3 = createUser("Charlie", 30, "charlie@example.com");
```

### Android View 커스텀 시 필수

```kotlin
class CustomButton @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : Button(context, attrs, defStyleAttr)
```

XML 레이아웃이 `View(Context, AttributeSet)` 생성자를 호출하므로 `@JvmOverloads`가 없으면 XML에서 사용 불가능합니다.

---

## 5. @JvmName — 컴파일 이름 변경

```kotlin
// 동일한 JVM 시그니처가 충돌할 때 사용
@JvmName("filterUsers")
fun List<User>.filter(predicate: (User) -> Boolean): List<User> = this.filter(predicate)

// 파일 수준 함수의 클래스 이름 변경
@file:JvmName("UserUtils")  // 기본: UserKt → UserUtils
package com.example

fun createDefaultUser() = User(0, "Guest", "")
```

```java
// Java 호출
UserUtils.createDefaultUser();  // UserKt 대신 UserUtils
```

---

## 6. @JvmSuppressWildcards — 와일드카드 제거

```kotlin
// Kotlin의 타입 변성은 Java에서 와일드카드로 나타남
fun processItems(items: List<String>) { /* ... */ }
// Java 시그니처: void processItems(List<? extends String> items)

@JvmSuppressWildcards
fun processItems(items: List<String>) { /* ... */ }
// Java 시그니처: void processItems(List<String> items)
```

---

## 7. Kotlin → Java 컴파일 변환 요약

| Kotlin | Java 변환 결과 |
|--------|--------------|
| `val` 프로퍼티 | private 필드 + getter |
| `var` 프로퍼티 | private 필드 + getter + setter |
| `@JvmField val` | public 필드 |
| companion object 함수 | `Companion.method()` |
| `@JvmStatic` 함수 | 정적 메서드 |
| 기본값 파라미터 | Java에서 생략 불가 |
| `@JvmOverloads` | 오버로드 메서드 자동 생성 |
| 최상위 함수 (파일명Utils.kt) | `UtilsKt.function()` |
| `@file:JvmName` | 지정한 이름으로 변경 |
| `object` 싱글턴 | `ObjectName.INSTANCE.method()` |

---

## 8. 실전 — 라이브러리 개발 체크리스트

```kotlin
// Java 호환 라이브러리를 만들 때 적용할 패턴들

// 1. companion object 정적 팩토리 메서드
class ApiClient private constructor(val baseUrl: String) {
    companion object {
        @JvmStatic
        fun create(baseUrl: String) = ApiClient(baseUrl)
    }
}

// 2. 커스텀 View — @JvmOverloads 생성자
class LoadingButton @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null,
    defStyleAttr: Int = 0
) : Button(context, attrs, defStyleAttr)

// 3. 상수는 @JvmField
object Constants {
    @JvmField val BASE_URL = "https://api.example.com"
    @JvmField val TIMEOUT = 30_000L
}
```

---

## 9. 정리

- **`@JvmStatic`**: companion object/object 함수를 Java 정적 메서드로 노출
- **`@JvmField`**: 프로퍼티를 getter/setter 없이 public 필드로 노출
- **`@JvmOverloads`**: Kotlin 기본값 파라미터 → Java 오버로드 메서드 자동 생성 (커스텀 View 필수)
- **`@JvmName`**: 컴파일 후 이름 변경 (파일 레벨 함수 클래스명 변경 포함)
- **`@JvmSuppressWildcards`**: 제네릭 타입 변성의 와일드카드 제거
