---
title: (Kotlin/코틀린) 행동 패턴 — 방문자 패턴 (Visitor)
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin으로 방문자 패턴을 구현하는 방법과 AST 처리, sealed class와의 조합, 이중 디스패치 원리를 정리합니다.
---

---

## 1. 방문자 패턴이란?

**객체 구조와 알고리즘을 분리**해, 객체를 수정하지 않고 새 연산을 추가할 수 있는 패턴입니다.
"새 기능을 추가할 때 기존 클래스는 건드리지 않겠다"는 OCP를 실현합니다.

```
Element  ←──── accept(Visitor)    ← 방문 허용
  ├── ConcreteElementA → visitor.visit(this)
  └── ConcreteElementB → visitor.visit(this)

Visitor  ←──── visit(ElementA), visit(ElementB)  ← 새 연산 추가
  ├── AreaVisitor     (넓이 계산)
  ├── PrintVisitor    (출력)
  └── ExportVisitor   (내보내기)
```

**이중 디스패치(Double Dispatch)**: Element는 어떤 Visitor인지 모르고, Visitor는 어떤 Element인지 모릅니다. `accept → visit` 두 번의 동적 디스패치로 "어떤 Element에 어떤 연산을"을 결정합니다.

---

## 2. 기본 구현

도형(Element)과 연산(Visitor)을 분리합니다. 새 연산(Visitor)이 필요할 때 도형 클래스는 수정하지 않습니다.

```kotlin
// 방문자 인터페이스 — 각 도형 타입별 visit 메서드를 모두 선언
interface ShapeVisitor {
    fun visit(circle: Circle): Double
    fun visit(rectangle: Rectangle): Double
    fun visit(triangle: Triangle): Double
}

// 요소 인터페이스 — 방문자를 받아들이는 accept만 선언
interface Shape {
    // accept: "나를 방문해도 좋아요" — visitor에게 자신을 넘김
    fun accept(visitor: ShapeVisitor): Double
}

// 각 도형은 자신의 타입에 맞는 visitor.visit()을 호출 (이중 디스패치)
data class Circle(val radius: Double) : Shape {
    override fun accept(visitor: ShapeVisitor) = visitor.visit(this)
    // visitor.visit(this): this가 Circle이므로 visit(circle)이 호출됨
}

data class Rectangle(val width: Double, val height: Double) : Shape {
    override fun accept(visitor: ShapeVisitor) = visitor.visit(this)
}

data class Triangle(val base: Double, val height: Double) : Shape {
    override fun accept(visitor: ShapeVisitor) = visitor.visit(this)
}

// 방문자 1: 넓이 계산 — 도형 클래스 수정 없이 새 연산 추가
object AreaVisitor : ShapeVisitor {
    override fun visit(c: Circle)    = Math.PI * c.radius * c.radius   // πr²
    override fun visit(r: Rectangle) = r.width * r.height              // 가로 × 세로
    override fun visit(t: Triangle)  = 0.5 * t.base * t.height         // ½ × 밑 × 높이
}

// 방문자 2: 둘레 계산 — 도형 클래스는 그대로, 방문자만 추가
object PerimeterVisitor : ShapeVisitor {
    override fun visit(c: Circle)    = 2 * Math.PI * c.radius          // 2πr
    override fun visit(r: Rectangle) = 2 * (r.width + r.height)        // 2(가로 + 세로)
    override fun visit(t: Triangle)  = Double.NaN  // 빗변 정보 없으므로 NaN
}

// 사용: 같은 도형 컬렉션에 다른 방문자를 적용
val shapes: List<Shape> = listOf(Circle(5.0), Rectangle(4.0, 6.0), Triangle(3.0, 4.0))

val totalArea = shapes.sumOf { it.accept(AreaVisitor) }
println("총 넓이: ${"%.2f".format(totalArea)}")  // π*25 + 24 + 6 = 108.54

// 방문자만 바꾸면 전혀 다른 연산 수행
shapes.forEach { println("둘레: ${it.accept(PerimeterVisitor)}") }
```

> **포인트**: 새 연산(방문자)을 추가할 때 `ShapeVisitor` 인터페이스를 구현하는 클래스만 추가하면 됩니다. 기존 `Circle`, `Rectangle`, `Triangle`은 전혀 수정하지 않습니다.

---

## 3. sealed class + when (Kotlin 방식)

Kotlin에서는 `sealed class`와 `when`으로 방문자 패턴을 더 간결하게 대체할 수 있습니다.
`when`이 exhaustive(모든 경우 처리)하도록 컴파일러가 강제합니다.

```kotlin
// 수식 트리 (AST)를 sealed class로 표현
sealed class Expr {
    data class Num(val value: Double) : Expr()                  // 숫자 리터럴
    data class Add(val left: Expr, val right: Expr) : Expr()    // 덧셈
    data class Mul(val left: Expr, val right: Expr) : Expr()    // 곱셈
    data class Neg(val expr: Expr) : Expr()                     // 음수화
}

// "방문자" 역할을 확장 함수로 구현 — 새 연산 추가 시 확장 함수만 추가
fun Expr.eval(): Double = when (this) {
    is Expr.Num -> value                         // 숫자는 그대로 반환
    is Expr.Add -> left.eval() + right.eval()    // 재귀적으로 좌/우 계산 후 합산
    is Expr.Mul -> left.eval() * right.eval()    // 재귀적으로 좌/우 계산 후 곱셈
    is Expr.Neg -> -expr.eval()                  // 재귀적으로 계산 후 부호 반전
}

// 또 다른 연산: 수식을 문자열로 표현
fun Expr.print(): String = when (this) {
    is Expr.Num -> value.toString()
    is Expr.Add -> "(${left.print()} + ${right.print()})"
    is Expr.Mul -> "(${left.print()} * ${right.print()})"
    is Expr.Neg -> "(-${expr.print()})"
}

// (3 + 4) * (-2) 를 Expr 트리로 표현
val expr = Expr.Mul(
    Expr.Add(Expr.Num(3.0), Expr.Num(4.0)),
    Expr.Neg(Expr.Num(2.0))
)

println(expr.print())  // ((3.0 + 4.0) * (-2.0))
println(expr.eval())   // -14.0  (7 * -2)
```

---

## 4. 실전 예제 — 문서 구조 렌더링

같은 문서 구조를 HTML과 Markdown으로 렌더링합니다. 각 렌더러가 방문자 역할을 합니다.

```kotlin
// 방문자 인터페이스 — 각 문서 요소 타입별 visit 선언
interface DocumentVisitor {
    fun visit(heading: Heading)
    fun visit(paragraph: Paragraph)
    fun visit(codeBlock: CodeBlock)
}

interface DocumentElement {
    fun accept(visitor: DocumentVisitor)
}

// 제목: level은 h1~h6에 해당
data class Heading(val level: Int, val text: String) : DocumentElement {
    override fun accept(visitor: DocumentVisitor) = visitor.visit(this)
}

data class Paragraph(val text: String) : DocumentElement {
    override fun accept(visitor: DocumentVisitor) = visitor.visit(this)
}

// 코드 블록: 언어 정보 포함 (문법 강조에 사용)
data class CodeBlock(val language: String, val code: String) : DocumentElement {
    override fun accept(visitor: DocumentVisitor) = visitor.visit(this)
}

// 방문자 A: HTML 렌더러
object HtmlRenderer : DocumentVisitor {
    override fun visit(h: Heading) = println("<h${h.level}>${h.text}</h${h.level}>")
    override fun visit(p: Paragraph) = println("<p>${p.text}</p>")
    override fun visit(c: CodeBlock) =
        println("<pre><code class=\"${c.language}\">${c.code}</code></pre>")
}

// 방문자 B: Markdown 렌더러 — 같은 문서 구조, 다른 출력 형식
object MarkdownRenderer : DocumentVisitor {
    // "#".repeat(level): h1=# h2=## h3=### ...
    override fun visit(h: Heading) = println("${"#".repeat(h.level)} ${h.text}")
    override fun visit(p: Paragraph) = println(p.text)
    // 언어 정보를 코드 펜스에 포함
    override fun visit(c: CodeBlock) = println("```${c.language}\n${c.code}\n```")
}

val doc: List<DocumentElement> = listOf(
    Heading(1, "Kotlin 가이드"),
    Paragraph("Kotlin은 JVM 언어입니다."),
    CodeBlock("kotlin", "fun main() = println(\"Hello\")")
)

println("=== HTML ===")
doc.forEach { it.accept(HtmlRenderer) }
// <h1>Kotlin 가이드</h1>
// <p>Kotlin은 JVM 언어입니다.</p>
// <pre><code class="kotlin">fun main() = println("Hello")</code></pre>

println("\n=== Markdown ===")
doc.forEach { it.accept(MarkdownRenderer) }
// # Kotlin 가이드
// Kotlin은 JVM 언어입니다.
// ```kotlin ...```
```

---

## 5. 방문자 vs sealed class 선택 기준

```
방문자 패턴이 유리한 경우:
✓ 요소 타입은 고정, 연산(방문자)이 자주 추가되는 경우
  → 새 방문자 클래스만 추가, 기존 요소 수정 없음
✓ 요소 클래스를 수정할 수 없을 때 (라이브러리 코드)

sealed class + when이 유리한 경우:
✓ Kotlin 고유 코드 — when이 누락 상태를 컴파일 시점에 감지
✓ 타입(요소)이 추가될 가능성이 있을 때
  → when에 분기 추가만 하면 됨 (방문자 패턴은 모든 방문자 수정 필요)
✓ 연산을 확장 함수로 분리하면 OCP도 만족
```

---

## 6. 정리

- 방문자 패턴: 객체 구조(요소)와 연산(방문자)을 분리 → OCP 실현
- **이중 디스패치**: `element.accept(visitor)` → `visitor.visit(element)` 두 번의 동적 바인딩
- Kotlin `sealed class + when`: 컴파일러 검증 포함, 더 간결하고 관용적인 대안
- 실전 활용: 컴파일러 AST 처리, 문서 렌더링(HTML/PDF/Markdown), 세금/할인 계산
