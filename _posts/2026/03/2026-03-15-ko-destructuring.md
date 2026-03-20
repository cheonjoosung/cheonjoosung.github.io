---
title: (Kotlin/코틀린) 구조분해 선언(Destructuring) 완전 정리
tags: [ Android,Kotlin ]
style: fill
color: dark
description: (Kotlin/코틀린) 구조분해 선언(Destructuring)의 동작 원리와 componentN 함수, data class, Pair/Triple, Map.Entry, lambda 파라미터, 실무 사용 시 주의점까지 예제 중심으로 정리합니다.
---

## 개요

- 코틀린의 구조분해 선언(Destructuring Declaration) 은 객체나 컬렉션의 값을 여러 변수로 한 번에 꺼내는 문법입니다.
- 처음 보면 단순 문법 설탕처럼 보이지만, 내부적으로는 `componentN()` 함수 호출로 동작합니다. 즉, “편한 문법” 정도로만 알고 쓰면 되고, 원리를 알면 `Pair`, `data class`, `Map`, `lambda`, `for문`까지 훨씬 정확하게 사용할 수 있습니다.
- 이 글에서는 다음을 정리합니다.
  - 구조분해 선언이 무엇인지
  - 내부적으로 어떻게 동작하는지
  - data class, Pair, Triple, Map에서 어떻게 쓰는지
  - lambda/for문에서의 활용
  - 실무에서 주의해야 할 점

---

## 1. 구조분해 선언이란?

```kotlin
val person = Person("Kim", 30)
val (name, age) = person
```

- 이렇게 하면 person 객체 안의 값을 각각 name, age 변수로 나눠 받을 수 있습니다.

---

## 2. 내부적으로 어떻게 동작하나?

```kotlin
val (name, age) = person

// 내부 동작
val name = person.component1()
val age = person.component2()
```

- 첫 번째 변수 → component1()
- 두 번째 변수 → component2()
- 세 번째 변수 → component3()
- 이 순서로 호출

---

### 3. data class에서 구조분해가 가능한 이유

data class는 컴파일러가 자동으로 componentN() 함수를 만들어줍니다.

```kotlin
data class Person(
    val name: String,
    val age: Int
)

// 위 코드 작성 시 아래의 함수들이 자동 생성
operator fun component1(): String = name
operator fun component2(): Int = age

// 자동 생성 된 componentN() 이 있기에 호출 가능
val person = Person("Kim", 30)
val (name, age) = person

println(name) // Kim
println(age)  // 30
```

---

## 4. Pair와 Triple에서 자주 사용된다

### 4.1 Pair

```kotlin
val pair = "Kotlin" to 100
val (language, score) = pair

println(language) // Kotlin
println(score)    // 100
```

### 4.2 Triple

```kotlin
val triple = Triple("Android", 14, true)
val (platform, version, isStable) = triple
```

---

## 5. Map 순회에서 자주 쓰는 패턴

```kotlin
val map = mapOf(
    "A" to 100,
    "B" to 200
)

for ((key, value) in map) {
    println("$key = $value")
}
```

- 이 문법이 가능한 이유는 Map.Entry에도 component1(), component2()가 지원되기 때문입니다.
- 실제 의미는
  - ```kotlin
    for (entry in map) {
      val key = entry.component1()
      val value = entry.component2()
    }
    ```

---

## 6. 람다에서도 구조분해 가능

```kotlin
map.forEach { (key, value) ->
    println("$key -> $value")
}
```

---

## 7. 사용하지 않는 값은 _로 무시할 수 있다

```kotlin
val person = Person("Kim", 30)
val (name, _) = person

println(name)
```

---

## 8. 구조분해는 “간결함”보다 “명확함”이 우선이다

- 구조분해는 분명 편리하지만, 항상 좋은 것은 아닙니다.
- 예를 들어 프로퍼티가 너무 많으면 오히려 읽기 어려워질 수 있습니다.

```kotlin
// 가독성 떨어짐
val (id, name, email, phone, address, birth, status) = user

// 명시적으로 꺼내는 것이 가독성 좋음
val name = user.name
val email = user.email
```
- 다음 조건 중 하나라도 걸리면 Smart Cast가 차단됩니다.
  - mutable property
  - open property
  - custom getter
  - multi-thread risk

---

## 9. 구조분해 vs 일반 프로퍼티 접근

| 항목     | 구조분해      | 일반 접근     |
| ------ | --------- | --------- |
| 코드 길이  | 짧음        | 다소 길 수 있음 |
| 가독성    | 상황에 따라 좋음 | 명확함       |
| 순서 의존성 | 있음        | 없음        |
| 실수 가능성 | 순서 실수 가능  | 적음        |