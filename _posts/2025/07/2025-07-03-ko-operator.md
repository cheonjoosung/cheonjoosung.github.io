---
title:  Kotlin 연산자 오버로딩 완벽 가이드 - invoke까지 한눈에
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 operator 오버로딩 개념과 자주 쓰이는 연산자, 그리고 invoke 연산자까지 실전 예제로 자세히 설명합니다.
---

## ✨ 개요

코틀린 `operator` 키워들 롵ㅇ해 클래스에 다양한 연산자를 재정의(오버로딩)할 수 있습니다. 
이를 통해 수학적 연산자, 비교 연산자, 컬렉션 연산자 등을 자신만의 로직으로 확장 가능합니다.

---

## 1. 🧩 기본 연산자 오버러딩

예를 들어 + 연산자를 오버로딩하고 싶다면 plus 함수 구현

```kotlin
val p1 = Point(1, 2)
val p2 = Point(3, 4)

(p1 + p2).also { println(it) } // Point(x=4, y=6)

data class Point(val x: Int, val y: Int) {
    operator fun plus(other: Point): Point {
        return Point(x + other.x, y + other.y)
    }
}

```

---

## 2. ⚙️ 비교 연산자 오버로딩

compareTo 메서드를 통해 <, >, <=, >= 연산 지원

```kotlin
val v1 = Version(1, 0)
val v2 = Version(2, 0)

println(v1 < v2) // true

data class Version(val major: Int, val minor: Int) : Comparable<Version> {
    override fun compareTo(other: Version): Int =
        when {
            this.major != other.major -> this.major - other.major
            else -> this.minor - other.minor
        }
}
```

---

## 3. 🧪 invoke 연산자 오버로딩

`invoke`는 인스턴스를 함수처럼 호출할 수 있게 해주는 연산자입니다.

```kotlin
val greeter = Greeter("Hello")
greeter("Kotlin") // Hello, Kotlin!

class Greeter(val message: String) {
    operator fun invoke(name: String) {
        println("$message, $name!")
    }
}
```

---

## 4. 🧾 결론

코틀린의 operator 오버로딩은 직관적이고 선언적인 문법을 완성해 주며, DSL 설계에도 매우 강력합니다.
특히 `invoke`는 함수처럼 객체를 다룰 수 있게 만들어주어 코드의 표현력을 극대화합니다.
다만 남용 시 가독성을 해칠 수 있으므로 의도를 명확히 표현할 때만 활용하세요.