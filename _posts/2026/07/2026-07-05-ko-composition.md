---
title: (Kotlin/코틀린) 함수 합성(Function Composition) 완전 정리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin에서 함수 합성(Function Composition)을 구현하는 방법과 andThen, compose 패턴, 파이프라인 구성 실전 예제를 정리합니다.
---

---

## 1. 함수 합성이란?

**함수 합성(Function Composition)**은 두 함수 `f`와 `g`를 결합해 새로운 함수 `g(f(x))`를 만드는 기법입니다.

```
f: A → B
g: B → C
g ∘ f: A → C    (f 먼저, g 나중)
```

```kotlin
val double = { x: Int -> x * 2 }
val addTen = { x: Int -> x + 10 }

// 직접 합성
val doubleThenAdd = { x: Int -> addTen(double(x)) }
println(doubleThenAdd(5))  // 20  (5 * 2 = 10, 10 + 10 = 20)
```

---

## 2. andThen / compose 확장 함수 구현

Kotlin 표준 라이브러리에는 함수 합성 연산자가 내장되어 있지 않아 직접 구현합니다.

```kotlin
// andThen: f 먼저 실행 후 g 실행 (왼쪽 → 오른쪽 순서)
infix fun <A, B, C> ((A) -> B).andThen(g: (B) -> C): (A) -> C =
    { a -> g(this(a)) }

// compose: g 먼저 실행 후 f 실행 (수학적 합성 순서)
infix fun <A, B, C> ((B) -> C).compose(f: (A) -> B): (A) -> C =
    { a -> this(f(a)) }
```

```kotlin
val double = { x: Int -> x * 2 }
val addTen = { x: Int -> x + 10 }
val toString = { x: Int -> "결과: $x" }

// andThen: double → addTen → toString 순서
val pipeline = double andThen addTen andThen toString
println(pipeline(5))  // 결과: 20

// compose: 수학적 표기 — toString ∘ addTen ∘ double
val composed = toString compose addTen compose double
println(composed(5))  // 결과: 20
```

---

## 3. 파이프라인 패턴

여러 변환 단계를 순서대로 연결합니다.

```kotlin
data class RawUser(val name: String, val email: String, val age: Int)
data class ValidatedUser(val name: String, val email: String, val age: Int)
data class UserDto(val displayName: String, val contact: String)

val trim: (RawUser) -> RawUser = { it.copy(name = it.name.trim(), email = it.email.trim()) }
val validate: (RawUser) -> ValidatedUser = { raw ->
    require(raw.name.isNotEmpty()) { "이름 필수" }
    require(raw.email.contains("@")) { "이메일 형식 오류" }
    require(raw.age in 0..150) { "나이 범위 오류" }
    ValidatedUser(raw.name, raw.email, raw.age)
}
val toDto: (ValidatedUser) -> UserDto = { user ->
    UserDto(
        displayName = "${user.name} (${user.age}세)",
        contact = user.email
    )
}

// 파이프라인 구성
val processUser = trim andThen validate andThen toDto

val raw = RawUser("  Alice  ", "alice@example.com", 28)
println(processUser(raw))
// UserDto(displayName=Alice (28세), contact=alice@example.com)
```

---

## 4. 리스트 변환에 적용

```kotlin
val normalize: (String) -> String = { it.trim().lowercase() }
val removeSpecial: (String) -> String = { it.replace(Regex("[^a-z0-9]"), "") }
val addPrefix: (String) -> String = { "user_$it" }

val processUsername = normalize andThen removeSpecial andThen addPrefix

val usernames = listOf("  Alice! ", "BOB_123", " charlie ")
println(usernames.map(processUsername))
// [user_alice, user_bob123, user_charlie]
```

---

## 5. 술어 함수 합성 (Predicate Composition)

```kotlin
// 술어 합성 — and, or, not
infix fun <T> ((T) -> Boolean).and(other: (T) -> Boolean): (T) -> Boolean =
    { t -> this(t) && other(t) }

infix fun <T> ((T) -> Boolean).or(other: (T) -> Boolean): (T) -> Boolean =
    { t -> this(t) || other(t) }

fun <T> ((T) -> Boolean).not(): (T) -> Boolean = { t -> !this(t) }
```

```kotlin
data class Product(val name: String, val price: Int, val inStock: Boolean)

val isAffordable: (Product) -> Boolean = { it.price < 50_000 }
val isAvailable: (Product) -> Boolean = { it.inStock }
val isPremium: (Product) -> Boolean = { it.price >= 100_000 }

// 조건 조합
val isGoodDeal = isAffordable and isAvailable
val isWorthChecking = isAvailable and isPremium.not()

val products = listOf(
    Product("A", 30_000, true),
    Product("B", 80_000, false),
    Product("C", 20_000, true)
)

println(products.filter(isGoodDeal))    // [A, C]
println(products.filter(isWorthChecking)) // [A, C]
```

---

## 6. Flow / 컬렉션과의 자연스러운 통합

```kotlin
// 변환 단계를 함수로 정의하고 재사용
val parsePrice: (String) -> Int = { it.replace(",", "").toInt() }
val applyDiscount: (Int) -> Int = { (it * 0.9).toInt() }
val formatPrice: (Int) -> String = { "${String.format("%,d", it)}원" }

val displayPrice = parsePrice andThen applyDiscount andThen formatPrice

listOf("10,000", "50,000", "100,000")
    .map(displayPrice)
    .forEach { println(it) }
// 9,000원
// 45,000원
// 90,000원
```

---

## 7. 함수 합성 vs 메서드 체이닝

| 구분 | 함수 합성 | 메서드 체이닝 |
|------|----------|-------------|
| 재사용 | 합성 함수 자체가 재사용 가능 | 체인 자체를 재사용하기 어려움 |
| 전달 | 함수를 값으로 전달 가능 | 전달 어려움 |
| 타입 | 컴파일 타임 타입 검사 | 컴파일 타임 타입 검사 |
| 적합 상황 | 변환 파이프라인, 고차 함수 | 컬렉션 처리, 빌더 패턴 |

```kotlin
// 메서드 체이닝 (일반적)
val result = listOf(1, 2, 3, 4, 5)
    .filter { it % 2 == 0 }
    .map { it * 2 }

// 함수 합성 (재사용 가능한 파이프라인)
val isEven: (Int) -> Boolean = { it % 2 == 0 }
val double: (Int) -> Int = { it * 2 }

val processNumber = double  // 단순하지만 조합 가능
listOf(1, 2, 3, 4, 5).filter(isEven).map(processNumber)
```

---

## 8. 정리

- **함수 합성**: 작은 함수들을 조합해 새 함수를 만드는 패턴
- `andThen`: 왼쪽 함수 → 오른쪽 함수 순서 (직관적)
- `compose`: 수학적 합성 순서 (오른쪽 → 왼쪽)
- 파이프라인 패턴으로 데이터 변환 단계를 선언적으로 표현 가능
- 술어 합성(`and`, `or`, `not`)으로 복잡한 조건을 모듈화
- Kotlin의 기본 라이브러리엔 없으므로 직접 확장함수로 구현
