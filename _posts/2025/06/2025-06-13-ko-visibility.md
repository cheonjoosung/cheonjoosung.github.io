---
title: Kotlin 접근 제어자 완벽 정리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 접근 제어자인 public, private, protected, internal 키워드의 차이와 사용 시 주의사항을 쉽게 설명합니다.
---

## ✨ 개요

Kotlin은 Java와 유사하지만 더 직관적인 접근 제어자(Visibility Modifiers) 를 제공합니다.

접근 제어자는 클래스, 함수, 변수의 접근 가능 범위를 명시하여 캡슐화(encapsulation) 와 모듈성(modularity) 을 유지하는 데 핵심적인 역할을 합니다.

---

## 1. ✅ public (기본값)

- 어디서든 접근 가능 (같은 모듈 + 외부 모듈 모두 접근 가능)
- 별도로 지정하지 않으면 기본으로 적용됨
- 클래스, 함수, 프로퍼티, 생성자 등에 사용 가능

```kotlin
public class Car {
    public fun drive() { }
}
```

---

## 2. ✅ private

- 선언된 클래스 내부 또는 파일 내부에서만 접근 가능
- top-level에서 사용할 경우 파일 내에서만 접근 가능 (Java에는 없는 개념)
- 구현 세부사항을 숨기고 싶을 때 유용

```kotlin
private fun logMessage() { }

class Car {
    private val engine = Engine()
}
```

---

## 3. ✅ protected

- 선언된 클래스와 그 하위 클래스에서만 접근 가능
- top-level에서는 사용할 수 없음 (클래스 내부에서만 사용 가능)


```kotlin
open class Vehicle {
    protected fun start() {}
}

class Car : Vehicle() {
    fun ready() {
        start() // 가능
    }
}
```

---

## 4. ✅ internal

- 같은 모듈(module) 내에서만 접근 가능
- 외부 라이브러리에서는 보이지 않음
- 모듈 단위 설계 시 유용

```kotlin
internal class InternalLogger {
    internal fun log() {}
}
```

---

## 5. ✅ 결론

| 제어자      | 접근 범위                            | 사용 위치                      |
|-------------|----------------------------------------|-------------------------------|
| `public`    | 어디서든 접근 가능                       | 기본값, 클래스/함수/변수 등     |
| `private`   | 클래스 내부 또는 같은 파일 내              | 세부 구현 은닉 목적              |
| `protected` | 클래스 및 하위 클래스에서만 접근 가능         | 클래스 내부에서만 사용 가능       |
| `internal`  | 같은 모듈 내에서만 접근 가능                | 모듈 단위 라이브러리 설계에 유용   |