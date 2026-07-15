---
title: (Kotlin/코틀린) operator fun iterator — for 루프 커스터마이징
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 operator fun iterator를 구현해 커스텀 클래스를 for 루프에서 사용하는 방법과 Iterator 직접 구현 패턴을 정리합니다.
---

---

## 1. operator fun iterator란?

`iterator` 연산자를 정의하면 클래스 인스턴스를 **for 루프에서 직접 순회**할 수 있습니다.

```kotlin
for (item in collection) { ... }
// 내부적으로 → collection.iterator() 호출
```

표준 라이브러리의 `List`, `Set`, `Map` 등이 이 방식으로 for 루프를 지원합니다.

---

## 2. 기본 구현

```kotlin
class NumberRange(private val start: Int, private val end: Int) {

    operator fun iterator(): Iterator<Int> = object : Iterator<Int> {
        private var current = start

        override fun hasNext(): Boolean = current <= end
        override fun next(): Int = current++
    }
}

val range = NumberRange(1, 5)
for (n in range) {
    print("$n ")  // 1 2 3 4 5
}
```

---

## 3. Iterable 구현으로 더 풍부한 API

`Iterable<T>`를 구현하면 `forEach`, `map`, `filter` 등 컬렉션 연산도 사용 가능합니다.

```kotlin
class Matrix(private val data: List<List<Int>>) : Iterable<Int> {

    override operator fun iterator(): Iterator<Int> = object : Iterator<Int> {
        private var row = 0
        private var col = 0

        override fun hasNext(): Boolean = row < data.size

        override fun next(): Int {
            val value = data[row][col]
            col++
            if (col >= data[row].size) { col = 0; row++ }
            return value
        }
    }
}

val matrix = Matrix(listOf(listOf(1, 2), listOf(3, 4), listOf(5, 6)))

for (n in matrix) print("$n ")   // 1 2 3 4 5 6
println(matrix.sum())             // 21
println(matrix.filter { it > 3}) // [4, 5, 6]
```

---

## 4. 확장 함수로 iterator 추가

기존 클래스를 수정하지 않고 `iterator`를 외부에서 추가할 수 있습니다.

```kotlin
data class DateRange(val from: LocalDate, val to: LocalDate)

operator fun DateRange.iterator(): Iterator<LocalDate> = object : Iterator<LocalDate> {
    private var current = from

    override fun hasNext(): Boolean = !current.isAfter(to)
    override fun next(): LocalDate = current.also { current = current.plusDays(1) }
}

val week = DateRange(LocalDate.of(2026, 7, 1), LocalDate.of(2026, 7, 7))
for (date in week) {
    println(date)
}
// 2026-07-01 ~ 2026-07-07
```

---

## 5. sequence로 lazy 순회

대용량 데이터나 무한 시퀀스는 `sequence`를 활용하면 메모리 효율적입니다.

```kotlin
class InfiniteCounter(private val start: Int = 0) {
    operator fun iterator(): Iterator<Int> = sequence {
        var n = start
        while (true) yield(n++)
    }.iterator()
}

val counter = InfiniteCounter(1)
for (n in counter) {
    if (n > 5) break
    print("$n ")  // 1 2 3 4 5
}
```

---

## 6. 실전 — 트리 순회

```kotlin
data class TreeNode<T>(val value: T, val children: List<TreeNode<T>> = emptyList())

class BreadthFirstIterator<T>(root: TreeNode<T>) : Iterator<T> {
    private val queue = ArrayDeque<TreeNode<T>>().also { it.add(root) }

    override fun hasNext(): Boolean = queue.isNotEmpty()
    override fun next(): T {
        val node = queue.removeFirst()
        queue.addAll(node.children)
        return node.value
    }
}

fun <T> TreeNode<T>.bfs() = Iterable { BreadthFirstIterator(this) }

val tree = TreeNode(1, listOf(
    TreeNode(2, listOf(TreeNode(4), TreeNode(5))),
    TreeNode(3, listOf(TreeNode(6)))
))

for (value in tree.bfs()) print("$value ")  // 1 2 3 4 5 6
```

---

## 7. 정리

- `operator fun iterator(): Iterator<T>` 정의 → 클래스를 for 루프에서 사용 가능
- `Iterable<T>` 구현 시 `map`, `filter`, `sum` 등 컬렉션 API도 사용 가능
- 확장 함수로 외부에서 기존 클래스에 iterator 추가 가능
- 대용량·무한 시퀀스는 `sequence { yield(...) }`로 lazy 처리
