---
title: (Kotlin/코틀린) 행동 패턴 — 이터레이터 패턴 (Iterator)
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin에서 Iterator 패턴을 구현하는 방법과 커스텀 Iterable, sequence, 실전 트리·범위 순회 예제를 정리합니다.
---

---

## 1. 이터레이터 패턴이란?

**컬렉션의 내부 구조를 노출하지 않고** 원소를 순서대로 접근할 수 있는 방법을 제공하는 패턴입니다.
List든 Tree든 Graph든 같은 방식(hasNext/next)으로 순회할 수 있습니다.

```
Iterable<T>            Iterator<T>
  iterator() ───→      hasNext(): Boolean  ← 다음 원소가 있는지 확인
                        next(): T           ← 다음 원소를 반환하고 커서 이동
```

Kotlin의 `for` 루프는 `Iterable<T>`를 구현한 모든 객체에서 동작합니다.

---

## 2. Kotlin의 내장 Iterator

Kotlin 컬렉션은 이미 이터레이터 패턴이 내장되어 있습니다.

```kotlin
val list = listOf(1, 2, 3)

// iterator()를 명시적으로 사용하는 방법
val iter = list.iterator()
while (iter.hasNext()) {
    println(iter.next())  // next()는 현재 원소 반환 + 커서 이동
}

// for 루프는 내부적으로 위와 동일 (컴파일러가 자동 변환)
for (item in list) println(item)

// withIndex(): 인덱스와 값을 함께 순회
for ((index, value) in list.withIndex()) {
    println("$index: $value")  // 0: 1, 1: 2, 2: 3
}
```

---

## 3. 커스텀 Iterator 구현

직접 `Iterable<T>`를 구현하면 `for` 루프에서 사용할 수 있습니다.
`object : Iterator<Int> { ... }` 형태로 익명 객체를 바로 반환합니다.

```kotlin
// 보폭(step)을 지정해 범위를 순회하는 커스텀 클래스
class StepRange(val start: Int, val end: Int, val step: Int = 1) : Iterable<Int> {
    override fun iterator() = object : Iterator<Int> {
        private var current = start  // 현재 위치

        // current가 end를 넘지 않았으면 다음 원소가 있음
        override fun hasNext() = current <= end

        // current 반환 후 step만큼 이동 (also: 반환값에는 영향 없이 부작용 실행)
        override fun next() = current.also { current += step }
    }
}

// 1부터 20까지 3씩 건너뜀
for (n in StepRange(1, 20, 3)) print("$n ")
// 출력: 1 4 7 10 13 16 19
```

> **포인트**: `Iterable<T>`를 구현하면 `for` 루프뿐 아니라 `map`, `filter`, `toList()` 등 모든 컬렉션 확장 함수도 사용할 수 있습니다.

---

## 4. 트리 순회 이터레이터

트리 구조를 BFS(너비 우선 탐색)로 순회하는 이터레이터입니다.
트리 내부 구조(배열, 링크드 리스트 등)를 몰라도 동일하게 순회할 수 있습니다.

```kotlin
data class TreeNode<T>(val value: T, val children: List<TreeNode<T>> = emptyList())

class BfsIterator<T>(root: TreeNode<T>) : Iterator<T> {
    // ArrayDeque를 큐(Queue)처럼 사용 — BFS의 핵심 자료구조
    private val queue = ArrayDeque<TreeNode<T>>().also { it.add(root) }

    // 큐가 비어있지 않으면 아직 순회할 노드가 있음
    override fun hasNext() = queue.isNotEmpty()

    override fun next(): T {
        val node = queue.removeFirst()  // 큐 앞에서 꺼냄
        // 꺼낸 노드의 자식들을 큐 뒤에 추가 → 다음에 처리됨
        queue.addAll(node.children)
        return node.value
    }
}

// 확장 함수로 편하게 사용
fun <T> TreeNode<T>.bfsIterator() = BfsIterator(this)

// 트리 구조:
//       1
//      / \
//     2   3
//    / \   \
//   4   5   6
val tree = TreeNode(1, listOf(
    TreeNode(2, listOf(TreeNode(4), TreeNode(5))),
    TreeNode(3, listOf(TreeNode(6)))
))

val iter = tree.bfsIterator()
while (iter.hasNext()) print("${iter.next()} ")
// 출력: 1 2 3 4 5 6  (레벨별로 왼쪽→오른쪽 순서)
```

---

## 5. sequence — 지연 평가 이터레이터

`sequence`는 필요할 때만 값을 생성하는 **지연 평가(lazy)** 이터레이터입니다.
무한 수열이나 대용량 데이터를 메모리 효율적으로 처리할 수 있습니다.

```kotlin
// 무한 피보나치 수열 — 모든 값을 미리 계산하지 않음
val fibonacci = sequence {
    var a = 0L; var b = 1L
    while (true) {     // 무한 루프지만 lazy이므로 메모리 문제 없음
        yield(a)       // 현재 값을 반환하고 다음 요청까지 일시 중단
        val next = a + b; a = b; b = next
    }
}

// take(10): 처음 10개만 요청 → 10개만 계산됨
println(fibonacci.take(10).toList())
// [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

// API 페이지네이션: 필요한 만큼만 가져옴
fun fetchPages(pageSize: Int = 10) = sequence {
    var page = 1
    while (true) {
        val items = fetchPage(page++, pageSize)  // API 호출
        if (items.isEmpty()) break               // 더 이상 데이터 없으면 종료
        yieldAll(items)  // 리스트 전체를 한 번에 yield
    }
}

// 처음 25개만 소비 — 실제 API는 3페이지까지만 호출됨 (10 + 10 + 5)
fetchPages().take(25).forEach { println(it) }
```

> **yield vs yieldAll**: `yield(value)`는 원소 하나를 반환, `yieldAll(iterable)`은 컬렉션의 모든 원소를 순서대로 반환합니다.

---

## 6. 실전 패턴 — 파일 시스템 순회

BFS 이터레이터를 sequence로 구현해 파일 시스템을 순회합니다.

```kotlin
import java.io.File

// File에 BFS 순회 확장 함수 추가
fun File.walkBfs(): Sequence<File> = sequence {
    val queue = ArrayDeque<File>()
    queue.add(this@walkBfs)  // 시작 디렉토리 추가

    while (queue.isNotEmpty()) {
        val file = queue.removeFirst()
        yield(file)  // 현재 파일/디렉토리를 하나씩 반환
        // 디렉토리라면 그 안의 파일들을 큐에 추가
        if (file.isDirectory) queue.addAll(file.listFiles() ?: emptyArray())
    }
}

// 특정 디렉토리에서 .kt 파일만 처음 5개 출력
File("/some/project").walkBfs()
    .filter { it.extension == "kt" }  // kt 파일만 필터
    .take(5)                           // 처음 5개만
    .forEach { println(it.path) }
```

---

## 7. operator fun iterator()

`operator fun iterator()`를 정의하면 일반 클래스를 `for` 루프에서 직접 사용할 수 있습니다.

```kotlin
class NumberMatrix(private val data: List<List<Int>>) {
    // operator 키워드로 for 루프 지원 추가
    // flatten(): 2차원 리스트를 1차원으로 펼침
    operator fun iterator() = data.flatten().iterator()
}

val matrix = NumberMatrix(listOf(listOf(1, 2, 3), listOf(4, 5, 6)))
for (n in matrix) print("$n ")  // 1 2 3 4 5 6 (행렬을 순서대로 순회)
```

---

## 8. 정리

| 방법 | 적합한 상황 |
|------|------------|
| `Iterator<T>` 직접 구현 | 복잡한 순회 로직 (트리, 그래프) |
| `Iterable<T>` 구현 | `for` 루프 + 컬렉션 함수 모두 지원 |
| `sequence { yield }` | 지연 평가, 무한 수열, 페이지네이션 |
| `operator fun iterator()` | 기존 클래스에 `for` 루프 지원만 추가 |

- `sequence`는 코루틴 기반 → `suspend` 함수나 IO 호출은 직접 불가 (대신 Flow 사용)
- 이터레이터 패턴: 컬렉션 구조를 몰라도 동일한 방식으로 순회 가능한 것이 핵심
