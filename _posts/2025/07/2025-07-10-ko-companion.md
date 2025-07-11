---
title: Kotlin Companion Object 완전정복
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin Companion Object 완전정복
---

## ✨ 개요

Kotlin을 사용하다 보면 companion object라는 특이한 문법을 자주 접하게 됩니다. 
자바의 static과 유사하지만, Kotlin만의 방식으로 풀어낸 이 기능을 제대로 이해하면 코틀린 코드가 훨씬 더 깔끔하고 유연해집니다.

---

## 1. 🧩 Companion Object란?

- `companion object` 는 클래스 내부에 선언된 싱글톤 객체
- 해당 클래스의 정적(static) 멤버처럼 동작
- 클래스가 인스턴스화 되지 않아도 `클래스명.함수()` 형태로 호출 가능

```kotlin
class MyClass {
    companion object {
        const val VERSION = "1.0"

        fun create(): MyClass {
            return MyClass()
        }
    }
}

fun main() {
    println(MyClass.VERSION)       // 1.0
    val obj = MyClass.create()     // static처럼 호출
}
```

---

## 2. ⚙️ 사용하는 이유

- Kotlin에는 static 키워드가 없음 → companion object가 이를 대체
- 팩토리 메서드, 상수(const), 유틸리티 함수 등을 담기 위함
- 자바의 static 멤버와의 상호운용(Java Interop)을 위해

---

## 3. 🧪 자바와 비교

| 기능            | Java                       | Kotlin                                  |
| ------------- | -------------------------- | --------------------------------------- |
| 정적 필드         | `static final int x = 10;` | `companion object { const val x = 10 }` |
| 정적 메서드        | `static void print()`      | `companion object { fun print() }`      |
| 클래스 로더 당 1개   | O                          | O                                       |
| 인스턴스 없이 접근 가능 | O                          | O                                       |

---

## 4. 🧾 다양한 예시

- 상수 정의
```kotlin
class AppConfig {
    companion object {
        const val BASE_URL = "https://api.example.com"
    }
}
println(AppConfig.BASE_URL)
```

- 팩토리 메서드
```kotlin
class User private constructor(val name: String) {
    companion object {
        fun newGuest() = User("Guest")
    }
}
val guest = User.newGuest()
```

- 이름 지정된 Companion object
```kotlin
class Sample {
    companion object Factory {
        fun of() = Sample()
    }
}
val s = Sample.of()
```

- 자바와의 호환
```kotlin
class Utils {
    companion object {
        @JvmStatic fun greet() = println("Hello from Kotlin")
    }
}
```

---

## 5. 🧾 결론

`companion object`는 Kotlin에서 정적 멤버를 선언할 수 있는 유일한 방법입니다. 
상수 관리, 팩토리 메서드 정의, 유틸 함수 구성에 매우 유용하며, Java와의 상호운용에서도 중요한 역할을 합니다.
Kotlin을 사용하는 개발자라면 반드시 숙지하고 능숙하게 활용할 수 있어야 합니다.