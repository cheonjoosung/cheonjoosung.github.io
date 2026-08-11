---
title: (Kotlin/코틀린) groupBy, partition, associate 실전 활용
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin 컬렉션의 groupBy, partition, associate, groupingBy 등 그룹화·변환 함수의 동작 원리와 실전 패턴을 정리합니다.
---

---

## 1. groupBy — 키 기준 그룹화

리스트를 **Map<K, List<V>>** 형태로 그룹화합니다.

```kotlin
data class Student(val name: String, val grade: Int, val score: Int)

val students = listOf(
    Student("Alice", 1, 90),
    Student("Bob", 2, 85),
    Student("Charlie", 1, 78),
    Student("Dave", 2, 92),
    Student("Eve", 1, 88)
)

// 학년별 그룹화
val byGrade: Map<Int, List<Student>> = students.groupBy { it.grade }
// {1=[Alice, Charlie, Eve], 2=[Bob, Dave]}

byGrade.forEach { (grade, list) ->
    println("$grade학년: ${list.map { it.name }}")
}
```

### 값 변환과 함께 사용

```kotlin
// groupBy(keySelector, valueTransform)
val gradeToNames: Map<Int, List<String>> = students.groupBy(
    keySelector = { it.grade },
    valueTransform = { it.name }
)
// {1=[Alice, Charlie, Eve], 2=[Bob, Dave]}
```

---

## 2. partition — 조건으로 두 그룹 분리

`Pair<List<T>, List<T>>`로 **조건 만족 / 불만족** 두 목록으로 나눕니다.

```kotlin
val (passed, failed) = students.partition { it.score >= 85 }

println("합격: ${passed.map { it.name }}")  // Alice, Bob, Dave, Eve
println("불합격: ${failed.map { it.name }}")  // Charlie
```

```kotlin
// 실전: API 응답 성공/실패 분리
data class ApiResult(val id: Int, val success: Boolean, val data: String?)

val results = listOf(
    ApiResult(1, true, "data1"),
    ApiResult(2, false, null),
    ApiResult(3, true, "data3")
)

val (successes, failures) = results.partition { it.success }
println("성공: ${successes.size}건, 실패: ${failures.size}건")
```

---

## 3. associate — 리스트를 Map으로 변환

`List<T>`를 `Map<K, V>`로 변환합니다.

```kotlin
// associate: (T) -> Pair<K, V>
val nameToScore: Map<String, Int> = students.associate { it.name to it.score }
// {Alice=90, Bob=85, Charlie=78, Dave=92, Eve=88}

// associateBy: 키만 지정, 값은 원본
val nameToStudent: Map<String, Student> = students.associateBy { it.name }

// associateWith: 값만 지정, 키는 원본
val studentToScore: Map<Student, Int> = students.associateWith { it.score }
```

### 중복 키 처리

```kotlin
val data = listOf("apple", "banana", "avocado", "blueberry")

// 중복 키 발생 시 마지막 값이 남음
val byFirstChar = data.associateBy { it.first() }
// {a=avocado, b=blueberry}  ← apple, banana 덮어씌워짐

// 중복을 유지하려면 groupBy 사용
val grouped = data.groupBy { it.first() }
// {a=[apple, avocado], b=[banana, blueberry]}
```

---

## 4. groupingBy — 집계 연산

`Grouping<T, K>` 객체를 반환해 **집계 연산**을 체이닝할 수 있습니다.

```kotlin
// 학년별 학생 수
val countByGrade = students.groupingBy { it.grade }.eachCount()
// {1=3, 2=2}

// 학년별 최고 점수
val maxScoreByGrade = students.groupingBy { it.grade }
    .fold(0) { acc, student -> maxOf(acc, student.score) }
// {1=90, 2=92}

// 학년별 평균 점수
val avgScoreByGrade = students
    .groupBy { it.grade }
    .mapValues { (_, list) -> list.map { it.score }.average() }
// {1=85.33, 2=88.5}
```

---

## 5. 실전 패턴 — 주문 통계

```kotlin
data class Order(
    val id: Int,
    val category: String,
    val amount: Double,
    val isPaid: Boolean
)

val orders = listOf(
    Order(1, "음식", 15000.0, true),
    Order(2, "의류", 45000.0, true),
    Order(3, "음식", 8000.0, false),
    Order(4, "전자", 120000.0, true),
    Order(5, "의류", 32000.0, false)
)

// 1. 카테고리별 총 매출
val revenueByCategory = orders
    .filter { it.isPaid }
    .groupBy { it.category }
    .mapValues { (_, list) -> list.sumOf { it.amount } }
// {음식=15000.0, 의류=45000.0, 전자=120000.0}

// 2. 결제 여부별 주문 수와 금액
val (paid, unpaid) = orders.partition { it.isPaid }
println("결제완료: ${paid.size}건 / ${paid.sumOf { it.amount }}원")
println("미결제: ${unpaid.size}건 / ${unpaid.sumOf { it.amount }}원")

// 3. 카테고리별 주문 ID 맵
val idsByCategory = orders.groupBy(
    keySelector = { it.category },
    valueTransform = { it.id }
)
// {음식=[1, 3], 의류=[2, 5], 전자=[4]}
```

---

## 6. 실전 패턴 — 사용자 권한 분류

```kotlin
enum class Role { ADMIN, EDITOR, VIEWER }

data class User(val id: Int, val name: String, val role: Role)

val users = listOf(
    User(1, "Alice", Role.ADMIN),
    User(2, "Bob", Role.EDITOR),
    User(3, "Charlie", Role.VIEWER),
    User(4, "Dave", Role.EDITOR),
    User(5, "Eve", Role.ADMIN)
)

// 역할별 사용자 ID 맵
val idsByRole: Map<Role, List<Int>> = users.groupBy(
    keySelector = { it.role },
    valueTransform = { it.id }
)

// 관리자 여부로 분리
val (admins, others) = users.partition { it.role == Role.ADMIN }

// 이름 → 역할 맵 (빠른 조회)
val roleByName: Map<String, Role> = users.associate { it.name to it.role }
println(roleByName["Alice"])  // ADMIN
```

---

## 7. associate vs groupBy 선택 기준

| 상황 | 선택 |
|------|------|
| 키가 고유 (중복 없음) | `associateBy` |
| 키가 중복될 수 있음 | `groupBy` |
| 키-값 모두 변환 | `associate { k to v }` |
| 두 그룹으로만 나눔 | `partition` |
| 집계 (count, sum, max) | `groupingBy` |

---

## 8. 정리

- **`groupBy`**: 키 기준 `Map<K, List<V>>` 반환, 중복 허용
- **`partition`**: 조건 기준 두 리스트로 분리 (`Pair<List, List>`)
- **`associate`** / **`associateBy`** / **`associateWith`**: 리스트 → Map 변환, 키 중복 시 마지막 값 유지
- **`groupingBy`**: `eachCount()`, `fold()` 등 집계 연산에 특화
- 중복 키가 있을 때 `associateBy` 대신 `groupBy` 사용 주의
