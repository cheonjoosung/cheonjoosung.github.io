---
title: (Kotlin/코틀린) Java에서 Kotlin 호출 시 꼭 알아야 할 JVM 어노테이션 정리
tags: [ Kotlin ]
style: fill
color: dark
description: (Kotlin/코틀린) Java에서 Kotlin 코드를 호출할 때 사용하는 @JvmStatic, @JvmField, @JvmOverloads 등 JVM 어노테이션의 역할과 사용 이유를 실무 예시와 함께 정리합니다.
---

## 개요

- Kotlin은 JVM 위에서 동작하며, Java와 100% 상호운용이 가능합니다.
- 하지만 Kotlin의 문법은 Java 바이트코드로 컴파일될 때 Java와 다른 방식으로 변환됩니다.
- 이로 인해 Java에서 Kotlin 코드를 호출하면 어색하거나 불편한 경우가 생깁니다.
- 이를 해결하기 위한 것이 JVM 어노테이션입니다.
- 이 글에서는 다음을 설명합니다.
  - 왜 JVM 어노테이션이 필요한지
  - 각 어노테이션이 무엇을 하는지
  - 언제 어떤 어노테이션을 써야 하는지

---

## 1. @JvmStatic

### 문제 상황

Kotlin `companion object` 안에 정의된 함수는 Java에서 보면 static이 아닙니다.

```kotlin
class MyClass {
    companion object {
        fun create(): MyClass = MyClass()
    }
}
```

Java에서 호출 시:

```java
MyClass obj = MyClass.Companion.create(); // ❌ 어색함
```

### @JvmStatic 적용

```kotlin
class MyClass {
    companion object {
        @JvmStatic
        fun create(): MyClass = MyClass()
    }
}
```

Java에서 호출 시:

```java
MyClass obj = MyClass.create(); // ✅ 자연스러운 static 호출
```

### 동작 원리

- `@JvmStatic`을 붙이면 컴파일러가 해당 메서드를 **실제 static 메서드**로도 생성합니다.
- `companion object` 내부의 인스턴스 메서드는 그대로 유지되고, static 메서드가 추가됩니다.

### object에서도 사용 가능

```kotlin
object Logger {
    @JvmStatic
    fun log(msg: String) = println(msg)
}
```

```java
Logger.log("hello"); // ✅
```

---

## 2. @JvmField

### 문제 상황

Kotlin 프로퍼티는 Java에서 보면 getter/setter로 변환됩니다.

```kotlin
class User {
    val name: String = "홍길동"
}
```

Java에서 호출 시:

```java
String name = user.getName(); // getter를 통해 접근
```

### @JvmField 적용

```kotlin
class User {
    @JvmField
    val name: String = "홍길동"
}
```

Java에서 호출 시:

```java
String name = user.name; // ✅ 필드로 직접 접근
```

### 동작 원리

- `@JvmField`를 붙이면 getter/setter를 생성하지 않고 **public 필드로 노출**합니다.
- 프레임워크에서 필드에 직접 접근해야 할 때 유용합니다. (예: Gson, Room, JUnit)

### companion object에서 상수 선언

```kotlin
class Config {
    companion object {
        @JvmField
        val BASE_URL = "https://api.example.com"
    }
}
```

```java
String url = Config.BASE_URL; // ✅ static 필드처럼 접근
```

---

## 3. @JvmOverloads

### 문제 상황

Kotlin은 기본값(default parameter)을 지원하지만, Java는 지원하지 않습니다.

```kotlin
fun greet(name: String, greeting: String = "Hello") {
    println("$greeting, $name!")
}
```

Java에서 호출 시:

```java
greet("홍길동", "Hello"); // 모든 파라미터를 명시해야 함
greet("홍길동");          // ❌ 컴파일 에러 - 오버로드가 없음
```

### @JvmOverloads 적용

```kotlin
@JvmOverloads
fun greet(name: String, greeting: String = "Hello") {
    println("$greeting, $name!")
}
```

Java에서 호출 시:

```java
greet("홍길동", "Hello"); // ✅
greet("홍길동");          // ✅ 오버로드 자동 생성
```

### 동작 원리

- `@JvmOverloads`를 붙이면 컴파일러가 **각 기본값 파라미터에 대한 오버로드 메서드**를 자동 생성합니다.
- 파라미터가 3개이고 2개에 기본값이 있다면 오버로드 메서드가 2개 추가 생성됩니다.

### 생성자에도 사용 가능

```kotlin
class Button @JvmOverloads constructor(
    context: Context,
    text: String = "",
    enabled: Boolean = true
)
```

---

## 4. @JvmName

### 문제 상황

Kotlin 확장 함수나 파일의 이름이 Java에서 보기 어색하거나 충돌이 생길 때 사용합니다.

```kotlin
// StringUtils.kt
fun String.isEmailValid(): Boolean { ... }
```

Java에서는 파일명 기반으로 `StringUtilsKt.isEmailValid(str)`로 호출됩니다.

### @JvmName 적용 - 파일 이름 변경

```kotlin
@file:JvmName("StringUtils")

// StringUtils.kt
fun String.isEmailValid(): Boolean { ... }
```

```java
StringUtils.isEmailValid(str); // ✅ 깔끔한 이름
```

### @JvmName 적용 - 함수 이름 변경

```kotlin
@JvmName("isEmailValidJava")
fun String.isEmailValid(): Boolean { ... }
```

```java
StringUtils.isEmailValidJava(str); // ✅
```

### getter/setter 이름 변경

```kotlin
val isActive: Boolean
    @JvmName("isActive") get() = true
```

---

## 5. @JvmSuppressWildcards / @JvmWildcard

### 문제 상황

Kotlin 제네릭 타입이 Java에서 wildcard(`? extends`, `? super`)로 변환될 때 의도와 다르게 동작하는 경우가 있습니다.

```kotlin
fun process(list: List<String>) { ... }
// Java에서는 List<? extends String>으로 변환될 수 있음
```

### @JvmSuppressWildcards

```kotlin
fun process(list: List<@JvmSuppressWildcards String>) { ... }
// Java에서 List<String>으로 유지
```

### @JvmWildcard

```kotlin
fun produce(): List<@JvmWildcard String>
// Java에서 List<? extends String>으로 강제 변환
```

---

## 6. @Throws

### 문제 상황

Kotlin은 checked exception이 없습니다. 하지만 Java에서는 checked exception을 try-catch로 처리해야 합니다.

```kotlin
fun readFile(path: String): String {
    return File(path).readText()
}
```

Java에서 호출 시 IOException을 잡으려 해도 컴파일러가 강제하지 않습니다.

### @Throws 적용

```kotlin
@Throws(IOException::class)
fun readFile(path: String): String {
    return File(path).readText()
}
```

Java에서 호출 시:

```java
try {
    String content = readFile("path/to/file"); // ✅ IOException 처리 강제
} catch (IOException e) {
    e.printStackTrace();
}
```

---

## 7. 어노테이션 요약

| 어노테이션 | 역할 | 주요 사용처 |
|------------|------|------------|
| `@JvmStatic` | companion object / object 멤버를 static으로 노출 | 팩토리 메서드, 유틸 함수 |
| `@JvmField` | 프로퍼티를 getter/setter 없이 public 필드로 노출 | 상수, 프레임워크 연동 |
| `@JvmOverloads` | 기본값 파라미터를 Java용 오버로드로 생성 | 생성자, 일반 함수 |
| `@JvmName` | 파일명 또는 함수명을 Java에서 보이는 이름으로 변경 | 확장 함수, 파일 이름 정리 |
| `@JvmSuppressWildcards` | 제네릭 타입의 wildcard 변환 억제 | 제네릭 함수 |
| `@JvmWildcard` | 제네릭 타입을 wildcard로 강제 변환 | 제네릭 함수 |
| `@Throws` | Kotlin 함수에 Java checked exception 선언 추가 | Java 연동 코드 |

---

## 8. 실무 팁

- **순수 Kotlin 프로젝트**라면 JVM 어노테이션은 거의 불필요합니다.
- **Java와 혼용**되는 프로젝트라면 `@JvmStatic`, `@JvmField`, `@JvmOverloads`는 자주 씁니다.
- **라이브러리 개발** 시에는 Java 호환성을 위해 적극적으로 사용하는 것이 좋습니다.
- `const val`은 `@JvmField`와 유사하게 동작합니다. 컴파일 타임 상수는 `const val`을 우선 사용하세요.

```kotlin
companion object {
    const val MAX_COUNT = 100    // Java에서 static final로 접근 가능
    @JvmField val TAG = "MyTag"  // 런타임 값은 @JvmField 사용
}
```

---

## 참고

- [Kotlin 공식 문서 - Calling Kotlin from Java](https://kotlinlang.org/docs/java-to-kotlin-interop.html)
