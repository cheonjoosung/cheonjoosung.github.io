---
title: (Kotlin/코틀린) 확장함수 vs 멤버함수 우선순위 규칙
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin에서 확장함수와 멤버함수가 동일한 시그니처일 때 어떤 것이 호출되는지 우선순위 규칙과 실전에서 주의해야 할 패턴을 정리합니다.
---

---

## 1. 핵심 규칙: 멤버함수가 항상 우선

확장함수와 멤버함수의 시그니처가 동일하면 **멤버함수가 항상 이깁니다.**

```kotlin
class MyClass {
    fun greet() = "멤버함수: 안녕!"
}

fun MyClass.greet() = "확장함수: 안녕!"  // 멤버와 동일 시그니처

val obj = MyClass()
println(obj.greet())  // 멤버함수: 안녕! ← 확장함수 무시됨
```

컴파일러가 멤버함수를 먼저 찾고, 없을 때만 확장함수를 탐색합니다.

---

## 2. 오버로드는 공존 가능

파라미터 타입이 다르면 멤버함수와 확장함수가 각각 호출됩니다.

```kotlin
class Printer {
    fun print(text: String) = println("멤버: $text")
}

fun Printer.print(number: Int) = println("확장: $number")

val p = Printer()
p.print("hello")  // 멤버: hello
p.print(42)       // 확장: 42
```

---

## 3. 상속과 확장함수 — 정적 디스패치

확장함수는 **컴파일 타임 타입**을 기준으로 결정됩니다 (정적 디스패치).  
멤버함수는 **런타임 타입**을 기준으로 결정됩니다 (동적 디스패치).

```kotlin
open class Animal
class Dog : Animal()

fun Animal.sound() = "..."
fun Dog.sound() = "멍멍"

val animal: Animal = Dog()
println(animal.sound())  // ... ← 컴파일 타입(Animal) 기준!

// 반면 멤버함수는
open class Animal2 { open fun sound() = "..." }
class Dog2 : Animal2() { override fun sound() = "멍멍" }

val animal2: Animal2 = Dog2()
println(animal2.sound())  // 멍멍 ← 런타임 타입(Dog2) 기준
```

이것이 확장함수와 멤버함수의 가장 중요한 동작 차이입니다.

---

## 4. nullable receiver — 확장함수만 가능

확장함수는 `null` 수신자에 대해 정의할 수 있습니다.

```kotlin
fun String?.isNullOrBlank(): Boolean = this == null || this.isBlank()

val s: String? = null
println(s.isNullOrBlank())  // true — null에도 안전하게 호출
```

멤버함수는 인스턴스가 있어야 호출 가능하므로 null 수신자를 받을 수 없습니다.

---

## 5. 가시성과 확장함수

확장함수는 수신 클래스의 **private/protected 멤버에 접근 불가**합니다.

```kotlin
class Secret {
    private val key = "비밀키"
    fun getKey() = key  // 멤버함수: private 접근 가능
}

fun Secret.exposeKey() = key  // 컴파일 에러: private 접근 불가
fun Secret.exposeKey() = getKey()  // 가능: public 멤버 통해 접근
```

---

## 6. 실전 — 언제 확장함수 vs 멤버함수를 선택할까

| 상황 | 권장 |
|------|------|
| 클래스 소유자이고 핵심 기능 | 멤버함수 |
| 외부 라이브러리 클래스에 기능 추가 | 확장함수 |
| 특정 모듈에서만 쓰는 유틸 | 확장함수 |
| 다형성(override)이 필요한 경우 | 멤버함수 |
| null safe 처리 | 확장함수 |
| private 멤버 접근 필요 | 멤버함수 |

```kotlin
// 확장함수: String은 외부 클래스이므로 수정 불가
fun String.toSnakeCase(): String =
    replace(Regex("([A-Z])")) { "_${it.value.lowercase()}" }.trimStart('_')

println("HelloWorld".toSnakeCase())  // hello_world
```

---

## 7. 동반 객체 확장

`companion object`에도 확장함수를 추가할 수 있습니다.

```kotlin
class MyClass {
    companion object
}

fun MyClass.Companion.create(): MyClass = MyClass()

val obj = MyClass.create()  // companion object 확장 호출
```

---

## 8. 정리

- **멤버함수 > 확장함수**: 동일 시그니처면 멤버함수가 항상 우선
- 확장함수는 **정적 디스패치** — 컴파일 타임 타입 기준으로 결정
- 멤버함수는 **동적 디스패치** — 런타임 타입 기준 (override 작동)
- 확장함수는 수신 객체의 private 멤버에 접근 불가
- nullable receiver 확장함수로 null 안전 유틸 작성 가능
