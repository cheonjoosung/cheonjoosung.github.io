---
title: (Kotlin/코틀린) Comparable & Comparator 구현
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin에서 Comparable과 Comparator를 구현해 객체를 정렬하는 방법과 compareBy, thenBy, sortedWith 등 실전 패턴을 정리합니다.
---

---

## 1. Comparable — 자연 정렬 순서 정의

`Comparable<T>`를 구현하면 `sorted()`, `<`, `>` 등을 바로 사용할 수 있습니다.

```kotlin
data class Student(val name: String, val score: Int) : Comparable<Student> {
    override fun compareTo(other: Student): Int {
        return this.score.compareTo(other.score)  // 점수 오름차순
    }
}

val students = listOf(
    Student("Alice", 90),
    Student("Bob", 75),
    Student("Charlie", 85)
)

println(students.sorted())
// [Student(name=Bob, score=75), Student(name=Charlie, score=85), Student(name=Alice, score=90)]

println(students.min())  // Student(name=Bob, score=75)
println(students.max())  // Student(name=Alice, score=90)
```

### compareTo 반환 규칙

```kotlin
override fun compareTo(other: T): Int {
    // 음수 → this가 other보다 작음 (앞에 위치)
    // 0    → 같음
    // 양수 → this가 other보다 큼 (뒤에 위치)
}
```

---

## 2. compareValuesBy — 간결한 compareTo 구현

여러 필드를 기준으로 비교할 때 `compareValuesBy`를 사용합니다.

```kotlin
data class Employee(
    val department: String,
    val name: String,
    val salary: Int
) : Comparable<Employee> {
    override fun compareTo(other: Employee): Int =
        compareValuesBy(this, other,
            { it.department },  // 1차: 부서 이름 오름차순
            { it.salary },      // 2차: 급여 오름차순
            { it.name }         // 3차: 이름 오름차순
        )
}
```

---

## 3. Comparator — 외부에서 정렬 기준 정의

`Comparable`을 구현하지 않고도 정렬 기준을 유연하게 지정합니다.

```kotlin
data class Product(val name: String, val price: Int, val rating: Double)

val products = listOf(
    Product("노트북", 1200000, 4.5),
    Product("마우스", 35000, 4.8),
    Product("키보드", 85000, 4.2),
    Product("모니터", 450000, 4.7)
)

// 가격 오름차순
val byPrice = Comparator<Product> { a, b -> a.price.compareTo(b.price) }
println(products.sortedWith(byPrice).map { it.name })
// [마우스, 키보드, 모니터, 노트북]

// 평점 내림차순
val byRatingDesc = Comparator<Product> { a, b -> b.rating.compareTo(a.rating) }
println(products.sortedWith(byRatingDesc).map { "${it.name}(${it.rating})" })
// [마우스(4.8), 모니터(4.7), 노트북(4.5), 키보드(4.2)]
```

---

## 4. compareBy / compareByDescending — 간결한 Comparator

```kotlin
// 단일 기준
val byPrice = compareBy<Product> { it.price }
val byPriceDesc = compareByDescending<Product> { it.price }

// 다중 기준 (1차, 2차, 3차...)
val multiSort = compareBy<Product>(
    { it.rating * -1 },  // 평점 내림차순 (음수 처리)
    { it.price }         // 가격 오름차순
)

println(products.sortedWith(multiSort).map { it.name })
```

---

## 5. thenBy / thenByDescending — 정렬 기준 체이닝

```kotlin
val sorted = products.sortedWith(
    compareByDescending<Product> { it.rating }
        .thenBy { it.price }       // 평점 같으면 가격 오름차순
        .thenBy { it.name }        // 가격도 같으면 이름 오름차순
)

sorted.forEach { println("${it.name}: 평점=${it.rating}, 가격=${it.price}") }
```

---

## 6. sortedBy / sortedByDescending — 단순 정렬

단일 기준 정렬은 이 함수들이 가장 간결합니다.

```kotlin
// 단일 기준
val byName = products.sortedBy { it.name }
val byPriceDesc = products.sortedByDescending { it.price }

// 문자열 정렬
val names = listOf("Charlie", "alice", "Bob")
println(names.sorted())               // [Bob, Charlie, alice] — 대소문자 구분
println(names.sortedBy { it.lowercase() })  // [alice, Bob, Charlie] — 대소문자 무시
```

---

## 7. 실전 패턴 — 복합 정렬 (쇼핑몰)

```kotlin
enum class SortOption { PRICE_ASC, PRICE_DESC, RATING_DESC, NAME_ASC }

fun sortProducts(products: List<Product>, option: SortOption): List<Product> {
    return when (option) {
        SortOption.PRICE_ASC   -> products.sortedBy { it.price }
        SortOption.PRICE_DESC  -> products.sortedByDescending { it.price }
        SortOption.RATING_DESC -> products.sortedWith(
            compareByDescending<Product> { it.rating }.thenBy { it.price }
        )
        SortOption.NAME_ASC    -> products.sortedBy { it.name.lowercase() }
    }
}
```

---

## 8. 실전 패턴 — 우선순위 큐

```kotlin
import java.util.PriorityQueue

data class Task(val name: String, val priority: Int, val deadline: Long)

// 우선순위 높은 것(숫자 클수록) → 마감 빠른 것 순
val taskQueue = PriorityQueue<Task>(
    compareByDescending<Task> { it.priority }
        .thenBy { it.deadline }
)

taskQueue.add(Task("이메일 답장", 1, System.currentTimeMillis() + 3600000))
taskQueue.add(Task("버그 수정", 3, System.currentTimeMillis() + 1800000))
taskQueue.add(Task("문서 작성", 2, System.currentTimeMillis() + 7200000))

while (taskQueue.isNotEmpty()) {
    val task = taskQueue.poll()
    println("처리: ${task.name} (우선순위: ${task.priority})")
}
// 처리: 버그 수정 (우선순위: 3)
// 처리: 문서 작성 (우선순위: 2)
// 처리: 이메일 답장 (우선순위: 1)
```

---

## 9. naturalOrder / reverseOrder

```kotlin
val numbers = listOf(3, 1, 4, 1, 5, 9, 2, 6)

println(numbers.sortedWith(naturalOrder()))   // [1, 1, 2, 3, 4, 5, 6, 9]
println(numbers.sortedWith(reverseOrder()))   // [9, 6, 5, 4, 3, 2, 1, 1]

// nullable 정렬 — null을 맨 앞 또는 맨 뒤로
val nullables = listOf("banana", null, "apple", null, "cherry")
println(nullables.sortedWith(nullsFirst(naturalOrder())))
// [null, null, apple, banana, cherry]
println(nullables.sortedWith(nullsLast(naturalOrder())))
// [apple, banana, cherry, null, null]
```

---

## 10. 정리

| 방법 | 용도 |
|------|------|
| `Comparable.compareTo` | 클래스의 기본 정렬 순서 정의 |
| `compareValuesBy` | 여러 필드 기준 compareTo 구현 |
| `compareBy { }` | 단/다중 기준 Comparator 생성 |
| `thenBy / thenByDescending` | 정렬 기준 체이닝 |
| `sortedBy { }` | 단일 기준 즉석 정렬 |
| `nullsFirst / nullsLast` | null 포함 컬렉션 정렬 |

- 클래스의 **고유 순서**가 있으면 `Comparable` 구현
- **상황에 따라 다른 정렬**이 필요하면 `Comparator` 사용
