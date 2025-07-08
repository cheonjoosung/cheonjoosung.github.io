---
title: Kotlin Any 클래스 완벽 이해하기
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 모든 클래스가 상속받는 최상위 타입 Any 클래스의 역할과 주요 메서드, Java Object와의 차이를 정리합니다.
---

## ✨ 개요

코틀린 `Any`는 모든 클래스의 최상위 타입입니다. Java의 `Object`와 유사하지만, 더 간결하고 명확한 타입 계층을 제공함
클래스 선언 시 명시적으로 상속하지 않아도 자동으로 `Any`를 상속함

---

## 1. 🧩 Any 클래스의 역할과 기본 메서드

```kotlin
open class Any {
    open operator fun equals(other:Any?): Boolean
    open fun hashCode(): Int
    open fun toString(): Stringoverride fun equals(other: kotlin.Any?): Boolean {
        if (this === other) return true
        if (javaClass != other?.javaClass) return false
        return true
    }
}
```

- `equals()` 두 객체의 동등성 비교
- `hashCode()` 객체의 해시값 반환 (Map, Set 등)
- `toString()` 문자열 표현 

---

## 2. ⚙️ Java Object vs Kotlin Any

| 항목               | Kotlin Any                            | Java Object                    |
|--------------------|----------------------------------------|---------------------------------|
| 최상위 타입         | 모든 클래스의 부모                      | 모든 클래스의 부모              |
| 메서드 수           | 3개 (`equals`, `hashCode`, `toString`) | 11개 이상 (`wait`, `notify` 등 포함) |
| Thread 관련 메서드 | 없음                                   | 있음 (`wait`, `notify`)        |
| Nullable 여부       | `Any`는 non-nullable                   | `Object`는 nullable 가능        |

---

## 3. 🧪 Any? Unit 차이

- Any 는 non-null 객체 타입의 최상의 타입
- Any? 는 null 포함 가능 (모든 nullable 타입의 최상위)
- Unit 은 반환값이 없는 함수의 결과를 의미하며 void 와 비슷하지만 값이 존재

```kotlin
fun example(): Any? = null
fun doSomething(): Unit = println("Done")
```

---

## 4. 🧾 결론

- Any 는 코틀린에서 모든 클래스가 기본적으로 상속하는 최상위 타입
- Java 의 Object 보다 더 경량화되어 있으며 필요에 따라 명시적 캐스팅 or 확장 메서드로 기능 추가 가능
- 코틀린의 타입 시스템을 이해하려면 Any, Any?, Unit 등 차이 파악 필요