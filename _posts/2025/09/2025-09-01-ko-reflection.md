---
title: Kotlin 리플렉션(Reflection) 완벽 가이드 – 클래스 정보 동적 조회와 활용법
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin 리플렉션(Reflection) 완벽 가이드 – 클래스 정보 동적 조회와 활용법
---

## ✨ 개요

Kotlin 리플렉션(Reflection)은 코드 실행 중에 클래스, 함수, 프로퍼티 등의 정보를 동적으로 조회하거나 조작할 수 있게 해주는 기능입니다.
자바에서도 자주 쓰이지만, Kotlin에서는 더 간결하고 안전하게 사용할 수 있는 방법이 존재합니다.

이번 포스팅에서는 Kotlin 리플렉션의 개념부터 실전 예제, 사용 시 주의사항까지 한 번에 정리해드립니다.
---

## 1. Kotlin 리플렉션이란?

- 리플렉션(Reflection)이란 코드 실행 중 객체의 메타정보(클래스, 메서드, 프로퍼티 등)를 조회하고 조작하는 기능입니다.
- Kotlin에서는 Java 리플렉션 외에도 자체적으로 kotlin-reflect 라이브러리를 통해 더 간편하고 안전한 리플렉션을 제공합니다.

📦 Kotlin에서 리플렉션 기능을 사용하려면 다음 의존성이 필요합니다:
```kotlin
// build.gradle.kts
dependencies {
    implementation("org.jetbrains.kotlin:kotlin-reflect")
}
```

---

## 🧪 2. 기본 사용 예제

🔹 클래스 참조 얻기
```kotlin
val clazz = String::class
println(clazz.simpleName)  // 출력: String
```

🔹 인스턴스로부터 클래스 참조 얻기
```kotlin
val name = "Hello"
val kClass = name::class
println(kClass.qualifiedName) // 출력: kotlin.String
```

---

## 3. 클래스의 프로퍼티와 메서드 접근

📌 프로퍼티 리스트 가져오기
```kotlin
data class User(val name: String, val age: Int)

val props = User::class.members
props.forEach { println(it.name) }
```

📌 특정 프로퍼티 값 가져오기
```kotlin
val user = User("Joo", 30)
val nameProp = User::name
println(nameProp.get(user)) // 출력: Joo
```

---

## 4. 리플렉션 고급 사용 예제

```kotlin
package org.example.com

import kotlin.reflect.full.createInstance
import kotlin.reflect.jvm.isAccessible


// 1️⃣ 샘플 클래스 정의
class Print {
    fun print() {
        println("Print 클래스의 print() 메소드 실행")
    }
}

class Num {
    fun add(a: Int, b: Int): Int = a + b
}

// 2️⃣ 리플렉션을 이용해 메서드 실행하는 함수
fun invokeMethod(className: String, methodName: String, vararg args: Any?): Any? {
    val clazz = Class.forName(className).kotlin

    // 인스턴스 생성
    val instance = clazz.createInstance()

    // 메서드 찾기
    val method = clazz.members.firstOrNull {
        it.name == methodName && it.parameters.size == args.size + 1 // 첫 파라미터는 instance 자신
    } ?: throw NoSuchMethodException("Method $methodName not found in $className")
    
    method.isAccessible = true

    // 메서드 실행
    return method.call(instance, *args)
}

fun main() {
    // Print 클래스의 print() 메서드 호출
    invokeMethod("org.example.com.Print", "print")

    // Num 클래스의 add(a, b) 메서드 호출
    val result = invokeMethod("org.example.com.Num", "add", 3, 5)
    println("Num 클래스의 add() 결과: $result")
}
```
- 리플렉션 대상 클래스가 패키지에 포함되어있지 않은 경우 동작이 안될 수 있음
 
---

## 5. 리플렉션의 한계 및 주의사항

| 항목           | 내용                       |
| ------------ | ------------------------ |
| 퍼포먼스 이슈      | 정적 호출보다 속도가 느림           |
| 코드 가독성 저하 가능 | 동적 호출은 디버깅이 어려움          |
| Proguard 문제  | 리플렉션 사용하는 클래스가 난독화될 수 있음 |

✅ 반드시 필요한 경우에만 리플렉션 사용! 남발 금지!
---

## 6. 결론

| 핵심 요약 🔑                          |
| --------------------------------- |
| Kotlin 리플렉션은 클래스 정보를 동적으로 처리하는 기능 |
| `kotlin-reflect` 라이브러리 필요         |
| 객체 생성, 프로퍼티 조회, 어노테이션 활용에 유용      |
| 퍼포먼스와 유지보수 이슈로 과도한 사용은 지양         |
