---
title: Kotlin 상속 제어 키워드 정리 - open, final, abstract 완전 이해하기
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin에서 상속을 제어하는 키워드인 open, final, abstract의 역할과 차이를 명확하게 설명하고, 클래스 설계 시 유의할 점을 예제와 함께 정리합니다.
---

## ✨ 개요

Kotlin은 Java보다 더 명확한 클래스 상속 제어 방식을 채택하고 있습니다. 

기본적으로 Kotlin의 클래스는 final(상속 불가)이며, 명시적으로 열어줘야 상속이 가능합니다.

따라서 open, final, abstract 키워드를 적절히 사용하는 것이 클래스 설계의 핵심입니다.

---

## 1. 🧪 open - 상속을 허용하는 클래스 또는 멤버

- Kotlin에서 클래스/메서드는 기본적으로 final임
- open 키워드를 붙여야 하위 클래스에서 상속 또는 오버라이딩 가능

```kotlin
open class Animal {
    open fun sound() = println("Unknown sound")
}

class Dog : Animal() {
    override fun sound() = println("Bark")
}
```

---

## 2. ✅ 🔒 final - 상속 및 오버라이딩 금지 (기본값)

- Kotlin에서는 final이 기본값이므로 따로 명시하지 않아도 됨
- 의도적으로 오버라이딩을 막고 싶을 때 명시적으로 사용 가능

```kotlin
open class Vehicle {
    final fun run() = println("Running")
}
```

---

## 3. ✅ abstract - 상속을 전제로 한 추상 설계

- abstract 클래스는 인스턴스화할 수 없고, 반드시 하위 클래스에서 구현해야 함
- 내부에 abstract 메서드를 포함할 수 있으며, 반드시 open으로 간주됨

```kotlin
abstract class Shape {
    abstract fun draw()
}

class Circle : Shape() {
    override fun draw() = println("Draw Circle")
}
```

---

## 4. 🧾 결론

| 키워드      | 역할                         | 기본 여부 | 사용 위치               |
|-------------|------------------------------|-----------|------------------------|
| `open`      | 상속 또는 오버라이딩 허용         | ❌        | 클래스, 함수, 프로퍼티     |
| `final`     | 상속 또는 오버라이딩 금지         | ✅ (기본값)| 클래스, 함수, 프로퍼티     |
| `abstract`  | 추상 클래스/함수 정의, 반드시 구현 필요 | ❌        | 클래스, 함수 (추상 클래스 내부) |

이러한 상속 제어 키워드를 통해 Kotlin에서는 명확하고 안전한 클래스 설계를 유도합니다. 기본적으로 닫힌 클래스를 기반으로 필요한 경우만 열어두는 습관이 안정성과 유지보수성을 높이는 데 효과적입니다.