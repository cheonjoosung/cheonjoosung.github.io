---
title: 객체지향 설계의 5대 원칙을 쉽게 이해하기 SOLID
tags: [ Java, Kotlin ]
style: fill
color: dark
description: 객체지향 설계에서 가장 중요한 SOLID 원칙을 실제 예제와 함께 설명하고, 각 원칙의 의미와 실무 적용 방법을 정리합니다.
---

## ✨ 개요

SOLID 원칙은 객체지향 프로그래밍의 기본이자, 유지보수성과 확장성을 갖춘 소프트웨어 설계의 핵심 지침입니다.

이 다섯 가지 원칙은 개발자가 더 나은 코드 구조를 만들고, 변화에 유연하게 대응하도록 도와줍니다.

---

## 1. ✅ SRP - 단일 책임 원칙 (Single Responsibility Principle)

- 클래스는 하나의 책임만 가져야 합니다.
- 책임이 여러 개라면 변경 이유도 여러 개가 되므로 유지보수가 어려워집니다.
- **왜 중요한가?**
  + 하나의 클래스가 여러 책임을 지게 되면, 한 책임의 변화가 다른 기능에도 영향을 줄 수 있어 버그 발생 확률이 높아집니다.
  + 책임 분리를 통해 수정 범위를 좁혀서 안정성을 높일 수 있습니다.

```kotlin
class ReportPrinter {
  fun print(report: Report) { /* 출력 로직 */ }
}
class ReportSaver {
  fun save(report: Report) { /* 저장 로직 */ }
}
```

---

## 2. ✅ OCP - 개방/폐쇄 원칙 (Open/Closed Principle)

- 소프트웨어 요소는 확장에는 열려 있고, 수정에는 닫혀 있어야 합니다.
- 기존 코드를 변경하지 않고 새로운 기능을 추가할 수 있어야 합니다.
- **왜 중요한가?**
  + 수정할 때마다 기존 코드를 건드리면 다른 기능에 영향을 줄 수 있어 위험합니다.
  + 새로운 요구사항이 생겨도 기존 코드는 그대로 두고 새로운 코드만 추가하면 되므로 안정성과 생산성이 올라갑니다.

```kotlin
interface Shape { fun draw() }
class Circle : Shape { override fun draw() { /* ... */ } }
class Rectangle : Shape { override fun draw() { /* ... */ } }
fun render(shapes: List<Shape>) {
  shapes.forEach { it.draw() }
}
```

---

## 3 ✅. LSP - 리스코프 치환 원칙 (Liskov Substitution Principle)

- 하위 클래스는 상위 클래스의 자리에 자유롭게 교체될 수 있어야 합니다.
- 하위 클래스가 부모의 행위를 무시하거나 위반하지 않아야 합니다.
- **왜 중요한가?**
  + LSP를 지키지 않으면 다형성을 활용할 수 없고, 코드 재사용성과 유지보수성이 급격히 떨어집니다.
  + 부모 클래스에 기대한 동작이 하위 클래스에서 깨지는 순간, 전체 시스템의 신뢰성이 무너집니다.

```kotlin
open class Rectangle {
  var width: Int = 0
  var height: Int = 0

  open fun setWidth(w: Int) { width = w }
  open fun setHeight(h: Int) { height = h }
  fun getArea(): Int = width * height
}

class Square : Rectangle() {
  override fun setWidth(w: Int) {
    width = w
    height = w // 정사각형은 너비 = 높이
  }

  override fun setHeight(h: Int) {
    width = h
    height = h // 정사각형은 높이 = 너비
  }
}

fun printArea(rect: Rectangle) {
  rect.setWidth(5)
  rect.setHeight(4)
  println("Expected area: 20, Actual area: ${rect.getArea()}")
}

fun main() {
  printArea(Rectangle())  // 출력: Expected area: 20, Actual area: 20
  printArea(Square())     // 출력: Expected area: 20, Actual area: 16 → LSP 위반
}
```

---

## 4 ✅. ISP - 인터페이스 분리 원칙 (Interface Segregation Principle)

- 하나의 커다란 인터페이스보다, 여러 개의 구체적인 인터페이스가 낫습니다.
- 클라이언트는 자신이 사용하지 않는 메서드에 의존하지 않아야 합니다.
- **왜 중요한가?**
  + 인터페이스가 너무 많거나 무거우면, 구현 클래스는 필요하지 않은 메서드까지 강제로 구현해야 하는 문제가 발생합니다.
  + 인터페이스를 기능별로 잘게 나누면 유연성과 재사용성이 높아집니다.

```kotlin
interface Printable { fun print() }
interface Scannable { fun scan() }
class Printer : Printable
class MultiFunctionPrinter : Printable, Scannable
```

---

## 5.✅ DIP - 의존 역전 원칙 (Dependency Inversion Principle)

- 고수준 모듈은 저수준 모듈에 의존해서는 안 되며, 둘 다 추상화에 의존해야 합니다.
- 세부사항이 아닌 인터페이스에 의존하도록 설계해야 합니다.
- **왜 중요한가?**
  + DIP를 지키면 유연한 구조를 만들 수 있어 테스트, 유지보수, 확장에 유리합니다. 
  + 특정 구현체에 의존하는 코드는 재사용하기 어렵고, 변경 시 파급효과도 큽니다. 
  + 추상화에 의존함으로써 느슨한 결합을 유지할 수 있습니다.

```kotlin
interface AuthService { fun login() }
class GoogleAuth : AuthService { override fun login() { /* Google 로그인 */ } }
class LoginManager(private val authService: AuthService) {
    fun authenticate() = authService.login()
}
```

---

## 6.🧠 **결론**

- SOLID 원칙을 따르면 유지보수성, 테스트 용이성, 확장성 높은 설계가 가능해집니다.
- 객체지향 설계에 대한 깊은 이해 없이도 SOLID 원칙을 실천함으로써 실질적인 코드 품질 향상을 기대할 수 있습니다.
- 처음엔 어렵더라도 작은 리팩토링부터 시작해 보는 것이 중요합니다.