---
title: (Kotlin/코틀린) Nullable receiver 확장함수 완전 정리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin에서 nullable receiver를 가진 확장함수를 정의하고 null 안전 유틸리티를 작성하는 패턴과 표준 라이브러리 활용 방법을 정리합니다.
---

---

## 1. Nullable receiver 확장함수란?

수신 타입에 `?`를 붙여 **null인 객체에도 안전하게 호출**할 수 있는 확장함수입니다.

```kotlin
fun String?.orDefault(default: String = ""): String = this ?: default

val name: String? = null
println(name.orDefault("이름 없음"))  // 이름 없음
println("Alice".orDefault())          // Alice
```

`null` 체크 없이 호출할 수 있어 null 처리 코드가 간결해집니다.

---

## 2. 표준 라이브러리의 nullable receiver 예시

Kotlin 표준 라이브러리도 이 패턴을 많이 사용합니다.

```kotlin
// isNullOrEmpty, isNullOrBlank
val s: String? = null
println(s.isNullOrEmpty())   // true
println(s.isNullOrBlank())   // true

// toString()은 Any?에 정의되어 null에도 호출 가능
val obj: Any? = null
println(obj.toString())  // "null"
```

---

## 3. 내부에서 null 처리

함수 내부에서 `this`가 null일 때의 동작을 명시적으로 정의합니다.

```kotlin
fun List<*>?.safeSize(): Int = this?.size ?: 0

val list: List<Int>? = null
println(list.safeSize())       // 0
println(listOf(1, 2, 3).safeSize())  // 3
```

```kotlin
fun Any?.isNull(): Boolean = this == null
fun Any?.isNotNull(): Boolean = this != null

val x: String? = "hello"
if (x.isNotNull()) println("값 있음")
```

---

## 4. 실전 패턴 — Android View 확장

```kotlin
fun View?.show() { this?.visibility = View.VISIBLE }
fun View?.hide() { this?.visibility = View.GONE }
fun View?.invisible() { this?.visibility = View.INVISIBLE }

// null이어도 안전하게 호출
binding?.progressBar.show()
binding?.errorView.hide()
```

```kotlin
// 클릭 리스너도 nullable receiver로
fun View?.onClick(action: () -> Unit) {
    this?.setOnClickListener { action() }
}

button?.onClick { viewModel.submit() }
```

---

## 5. 실전 패턴 — 컬렉션 유틸

```kotlin
fun <T> Collection<T>?.orEmpty(): List<T> = this?.toList() ?: emptyList()
// 표준 라이브러리에 이미 있지만, 확장 패턴 이해용

fun <T> List<T>?.firstOrDefault(default: T): T = this?.firstOrNull() ?: default
fun <T> List<T>?.lastOrDefault(default: T): T = this?.lastOrNull() ?: default

val items: List<String>? = null
println(items.firstOrDefault("없음"))  // 없음
println(listOf("a", "b").firstOrDefault("없음"))  // a
```

---

## 6. 일반 확장함수 vs nullable receiver 비교

```kotlin
// 일반 확장함수: null이면 NPE
fun String.shout() = toUpperCase() + "!"
val s: String? = "hello"
// s.shout()  // 컴파일 에러: String?에 String 확장함수 호출 불가
s?.shout()   // 안전 호출 연산자 사용 필요

// nullable receiver: null 포함 바로 호출 가능
fun String?.shoutSafe() = (this ?: "").uppercase() + "!"
s.shoutSafe()   // HELLO! — 안전 호출 없이도 가능
null.shoutSafe() // !
```

---

## 7. 주의 사항

```kotlin
// this가 null일 때 처리를 잊으면 NPE 발생 가능
fun String?.badExtension(): Int = length  // ⚠️ this가 null이면 NPE

// 올바른 패턴
fun String?.safeLength(): Int = this?.length ?: 0
```

nullable receiver는 편리하지만 내부에서 `this`를 사용할 때는 반드시 null 체크(`?.` 또는 `?: `)가 필요합니다.

---

## 8. 정리

- `fun T?.extension()` 형태로 null 수신자도 처리 가능한 확장함수 정의
- 내부에서 `this`는 nullable이므로 `?.` 또는 `?:` 로 처리 필요
- 표준 라이브러리의 `isNullOrEmpty()`, `toString()` 등이 동일 패턴
- Android View, 컬렉션 등 null이 자주 발생하는 곳에서 null-safe 유틸 작성에 활용
