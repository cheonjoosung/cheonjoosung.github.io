---
title: Kotlin init 블록과 생성자(constructor) 완벽 정리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin 클래스의 초기화 과정에서 사용되는 init 블록과 constructor의 차이, 순서, 역할에 대해 이해하기 쉽게 설명합니다.
---

## ✨ 개요

Kotlin에서 클래스의 초기화 과정은 매우 명확하게 설계되어 있으며, 이를 위해 constructor(생성자)와 init 블록이 함께 사용됩니다. 이 두 가지는 객체 생성 시 초기 상태를 설정하는 데 중요한 역할을 합니다.

---

## 1. 🔧 기본 구조와 문법

```kotlin
class User(val name: String, val age: Int) {
    init {
        println("Init block: name = $name, age = $age")
    }
}

val user = User("Alice", 30)
// 출력: Init block: name = Alice, age = 30
```
- 주 생성자 선언 시, init 블록은 생성자 본문이 없는 대신 초기화 로직을 포함
- 주 생성자 매개변수는 init 블록에서 직접 접근 가능

---

## 2. ✅ 생성자의 종류와 역할

- 주 생성자 (Primary Constructor)
  + 클래스 헤더에 정의되는 기본 생성자
  + 클래스 본문 없이도 속성 초기화 가능
  + > class Car(val brand: String)
    
- 보조 생성자 (Secondary Constructor)
  + constructor 키워드 사용
  + 반드시 this()를 통해 주 생성자를 호출해야 함
  + ```kotlin
    class Car(val brand: String) {
      constructor(brand: String, year: Int) : this(brand) {
        println("보조 생성자 호출됨: $year")
      }
    }
    ```

---

## 3. 🔄 실행 순서 요약

- 프로퍼티 선언부 초기화
- init 블록 실행 (순서대로)
- 보조 생성자 본문 실행 (있는 경우)
```kotlin
class Sample(val name: String) {
    val tag = "[Sample]"
    init { println("Step 1: $tag init") }

    constructor(name: String, age: Int) : this(name) {
        println("Step 2: 보조 생성자 실행")
    }
}
```

## 4. 🔄 차이점

| 항목    | `init` 블록       | `constructor()`         |
| ----- | --------------- | ----------------------- |
| 실행 시점 | 객체 생성 시 자동 실행   | 명시적으로 호출 필요             |
| 목적    | 공통 초기화, 검증 로직   | 다양한 생성 방법 제공            |
| 개수    | 여러 개 가능         | 여러 개 가능 (오버로드)          |
| 호출 순서 | 생성자 → `init` 순서 | 보조 생성자 → 주 생성자 → `init` |

---

## 5. 🧾 결론

- init 블록은 생성자 호출 시 초기화 로직을 정의하는 곳으로, 주 생성자와 함께 필수적인 초기 작업을 수행합니다.
- 보조 생성자는 다양한 초기화 경로를 제공하지만, 반드시 주 생성자를 호출해야 하므로 init 블록 이후 실행됩니다.
- 실행 순서를 이해하고, 복잡한 초기화가 필요한 경우에는 init과 constructor를 적절히 조합해 사용해야 안정적이고 명확한 클래스 설계가 가능합니다.