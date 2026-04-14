---
title: (Kotlin/코틀린) Collection 확장함수 실전 모음 - 그룹핑/집계/검색/변환/Set·Map 연산
tags: [ Kotlin ]
style: fill
color: dark
description: (Kotlin/코틀린) groupBy, zip, flatten, Set 집합 연산, Map 조작, 검색/탐색, 누적 합산, 다중 정렬 등 컬렉션 확장함수를 실전 예제와 함께 정리합니다.
---

## 개요

- Kotlin 표준 라이브러리는 컬렉션에 대한 풍부한 확장함수를 제공합니다.
- 하지만 실무에서 꼭 필요한 패턴들은 표준 함수를 조합하거나 커스텀 확장함수로 만들어야 합니다.
- 이 글에서는 그룹핑, 집계, Zip/Flatten, Set 집합 연산, Map 조작, 검색/탐색, 누적 합산, 다중 정렬 등을 카테고리별로 정리합니다.

---

## 1. 그룹핑 & 집계

### 표준 groupBy / groupingBy

```kotlin
data class Order(val userId: Int, val amount: Int, val status: String)

val orders = listOf(
    Order(1, 3000, "완료"),
    Order(2, 5000, "완료"),
    Order(1, 2000, "취소"),
    Order(3, 8000, "완료"),
    Order(2, 1000, "취소"),
)

// 유저별 주문 목록
val byUser: Map<Int, List<Order>> = orders.groupBy { it.userId }
// {1=[Order(1,3000,완료), Order(1,2000,취소)], ...}

// 유저별 주문 건수
val countByUser: Map<Int, Int> = orders.groupingBy { it.userId }.eachCount()
// {1=2, 2=2, 3=1}

// 유저별 총 결제 금액
val sumByUser: Map<Int, Int> = orders.groupBy { it.userId }
    .mapValues { (_, list) -> list.sumOf { it.amount } }
// {1=5000, 2=6000, 3=8000}
```

### 커스텀 그룹핑 확장함수

```kotlin
/** 키 기준 그룹핑 후 각 그룹 합산 */
inline fun <T, K> Iterable<T>.groupSumBy(
    keySelector: (T) -> K,
    valueSelector: (T) -> Int
): Map<K, Int> = groupBy(keySelector).mapValues { (_, v) -> v.sumOf(valueSelector) }

/** 키 기준 그룹핑 후 각 그룹 평균 */
inline fun <T, K> Iterable<T>.groupAverageBy(
    keySelector: (T) -> K,
    valueSelector: (T) -> Double
): Map<K, Double> = groupBy(keySelector).mapValues { (_, v) -> v.map(valueSelector).average() }

/** 상위 N개 그룹만 반환 (합산 기준 내림차순) */
inline fun <T, K> Iterable<T>.topNGroupsBy(
    n: Int,
    keySelector: (T) -> K,
    valueSelector: (T) -> Int
): Map<K, Int> = groupSumBy(keySelector, valueSelector)
    .entries.sortedByDescending { it.value }.take(n)
    .associate { it.key to it.value }
```

**사용 예시**

```kotlin
val amountByUser = orders.groupSumBy({ it.userId }, { it.amount })
println(amountByUser) // {1=5000, 2=6000, 3=8000}

val top2 = orders.topNGroupsBy(2, { it.userId }, { it.amount })
println(top2) // {3=8000, 2=6000}
```

---

## 2. Zip / Combine

### 표준 zip

```kotlin
val names = listOf("Alice", "Bob", "Charlie")
val scores = listOf(90, 85, 92)

// 두 리스트를 묶어 Pair 리스트로
val zipped: List<Pair<String, Int>> = names.zip(scores)
// [(Alice, 90), (Bob, 85), (Charlie, 92)]

// 변환 함수 적용
val result = names.zip(scores) { name, score -> "$name: $score점" }
// ["Alice: 90점", "Bob: 85점", "Charlie: 92점"]

// 인접 요소 비교 (zipWithNext)
val diffs = scores.zipWithNext { a, b -> b - a }
// [-5, 7]
```

### 커스텀 Zip 확장함수

```kotlin
/** 3개 리스트를 Triple로 묶기 */
fun <A, B, C> zip3(a: List<A>, b: List<B>, c: List<C>): List<Triple<A, B, C>> =
    (0 until minOf(a.size, b.size, c.size)).map { Triple(a[it], b[it], c[it]) }

/** 인덱스 포함하여 zip */
fun <T, R> List<T>.zipWithIndex(transform: (index: Int, T) -> R): List<R> =
    mapIndexed { i, t -> transform(i, t) }

/** 두 리스트 길이가 달라도 안전하게 zip (짧은 쪽 기준, 없으면 null) */
fun <A, B> List<A>.zipOrNull(other: List<B>): List<Pair<A?, B?>> {
    val size = maxOf(this.size, other.size)
    return (0 until size).map { i -> getOrNull(i) to other.getOrNull(i) }
}

/** Pair 리스트를 두 리스트로 분리 */
fun <A, B> List<Pair<A, B>>.unzipToLists(): Pair<List<A>, List<B>> = unzip()
```

**사용 예시**

```kotlin
val labels = listOf("1위", "2위", "3위")
val items = listOf("Gold", "Silver")
println(labels.zipOrNull(items))
// [(1위, Gold), (2위, Silver), (3위, null)]

val triples = zip3(listOf(1, 2), listOf("a", "b"), listOf(true, false))
println(triples)
// [(1, a, true), (2, b, false)]

val (first, second) = listOf("A" to 1, "B" to 2, "C" to 3).unzipToLists()
println(first)  // [A, B, C]
println(second) // [1, 2, 3]
```

---

## 3. Flatten / FlatMap

```kotlin
/** 중첩 리스트 1단계 펼치기 */
val nested = listOf(listOf(1, 2), listOf(3, 4), listOf(5))
val flat = nested.flatten()
println(flat) // [1, 2, 3, 4, 5]

/** 변환 + 펼치기 */
data class Team(val name: String, val members: List<String>)
val teams = listOf(
    Team("Android", listOf("Alice", "Bob")),
    Team("iOS", listOf("Charlie", "Dave"))
)
val allMembers = teams.flatMap { it.members }
println(allMembers) // [Alice, Bob, Charlie, Dave]
```

### 커스텀 Flatten 확장함수

```kotlin
/** null을 제외하고 flatMap */
inline fun <T, R : Any> Iterable<T>.flatMapNotNull(transform: (T) -> List<R?>): List<R> =
    flatMap(transform).filterNotNull()

/** 조건을 만족하는 요소만 flatMap */
inline fun <T, R> Iterable<T>.flatMapIf(
    predicate: (T) -> Boolean,
    transform: (T) -> List<R>
): List<R> = filter(predicate).flatMap(transform)

/** 중첩 맵의 모든 값을 하나의 리스트로 */
fun <K, V> Map<K, List<V>>.flattenValues(): List<V> = values.flatten()

/** 그룹별 대표값만 추출 (첫 번째 요소) */
inline fun <T, K> Iterable<T>.flatGroupFirstBy(keySelector: (T) -> K): List<T> =
    groupBy(keySelector).values.mapNotNull { it.firstOrNull() }
```

**사용 예시**

```kotlin
val teamMembersMap = mapOf(
    "Android" to listOf("Alice", "Bob"),
    "iOS" to listOf("Charlie")
)
println(teamMembersMap.flattenValues()) // [Alice, Bob, Charlie]

val data = listOf(
    listOf("a", null, "b"),
    listOf(null, "c")
)
println(data.flatMapNotNull { it }) // [a, b, c]
```

---

## 4. Set 집합 연산

```kotlin
val setA = setOf(1, 2, 3, 4, 5)
val setB = setOf(3, 4, 5, 6, 7)

// 합집합
val union = setA union setB
println(union) // [1, 2, 3, 4, 5, 6, 7]

// 교집합
val intersect = setA intersect setB
println(intersect) // [3, 4, 5]

// 차집합 (A - B)
val subtract = setA subtract setB
println(subtract) // [1, 2]
```

### 커스텀 Set 확장함수

```kotlin
/** 대칭 차집합 (A에만 있거나 B에만 있는 것) */
fun <T> Set<T>.symmetricDifference(other: Set<T>): Set<T> =
    (this - other) + (other - this)

/** 두 Set이 겹치는 부분이 있는지 */
fun <T> Set<T>.overlaps(other: Set<T>): Boolean =
    (this intersect other).isNotEmpty()

/** 특정 기준으로 중복 제거 후 Set 반환 */
inline fun <T, K> Iterable<T>.toSetBy(keySelector: (T) -> K): Set<T> =
    associateBy(keySelector).values.toSet()

/** List → Set 변환 후 포함 여부 빠르게 확인 */
fun <T> Iterable<T>.toFastLookupSet(): Set<T> = toHashSet()
```

**사용 예시**

```kotlin
println(setA.symmetricDifference(setB)) // [1, 2, 6, 7]
println(setA.overlaps(setB))             // true
println(setOf(1, 2).overlaps(setOf(3, 4))) // false

data class User(val id: Int, val name: String)
val users = listOf(User(1, "A"), User(2, "B"), User(1, "A_dup"))
println(users.toSetBy { it.id }.map { it.name }) // [A, B]
```

---

## 5. Map 조작

### 표준 Map 연산

```kotlin
val map1 = mapOf("a" to 1, "b" to 2)
val map2 = mapOf("b" to 3, "c" to 4)

// 키/값만 변환
val upperKeys = map1.mapKeys { (k, _) -> k.uppercase() }  // {A=1, B=2}
val doubledValues = map1.mapValues { (_, v) -> v * 2 }     // {a=2, b=4}

// 조건으로 필터
val filtered = map1.filterValues { it > 1 } // {b=2}
val filteredK = map1.filterKeys { it != "a" } // {b=2}
```

### 커스텀 Map 확장함수

```kotlin
/** 두 Map 병합 (충돌 시 병합 함수 적용) */
fun <K, V> Map<K, V>.mergeWith(other: Map<K, V>, resolve: (V, V) -> V): Map<K, V> =
    (keys + other.keys).associateWith { key ->
        val v1 = this[key]
        val v2 = other[key]
        when {
            v1 == null -> v2!!
            v2 == null -> v1
            else -> resolve(v1, v2)
        }
    }

/** Map 뒤집기 (value → key) */
fun <K, V> Map<K, V>.inverted(): Map<V, K> =
    entries.associate { (k, v) -> v to k }

/** 특정 키들만 추출한 Map */
fun <K, V> Map<K, V>.pick(vararg keys: K): Map<K, V> =
    filterKeys { it in keys }

/** 특정 키들을 제외한 Map */
fun <K, V> Map<K, V>.omit(vararg keys: K): Map<K, V> =
    filterKeys { it !in keys }

/** null 값 제거 */
fun <K, V : Any> Map<K, V?>.filterNotNullValues(): Map<K, V> =
    entries.mapNotNull { (k, v) -> v?.let { k to it } }.toMap()

/** Map을 정렬된 리스트로 변환 */
fun <K, V : Comparable<V>> Map<K, V>.toSortedListByValue(descending: Boolean = false): List<Pair<K, V>> =
    entries.map { it.toPair() }
        .let { if (descending) it.sortedByDescending { p -> p.second } else it.sortedBy { p -> p.second } }

/** 값 기준 상위 N개 */
fun <K, V : Comparable<V>> Map<K, V>.topN(n: Int): Map<K, V> =
    toSortedListByValue(descending = true).take(n).toMap()
```

**사용 예시**

```kotlin
val scores1 = mapOf("Alice" to 80, "Bob" to 70)
val scores2 = mapOf("Bob" to 90, "Charlie" to 85)

// 점수 합산으로 병합
val merged = scores1.mergeWith(scores2) { a, b -> a + b }
println(merged) // {Alice=80, Bob=160, Charlie=85}

val codeMap = mapOf(200 to "OK", 404 to "Not Found", 500 to "Error")
println(codeMap.inverted()) // {OK=200, Not Found=404, Error=500}
println(codeMap.pick(200, 404)) // {200=OK, 404=Not Found}
println(codeMap.omit(500))      // {200=OK, 404=Not Found}

val ranking = mapOf("Alice" to 90, "Bob" to 70, "Charlie" to 85)
println(ranking.topN(2)) // {Alice=90, Charlie=85}
```

---

## 6. 검색 / 탐색

```kotlin
/** 조건을 만족하는 첫 번째 인덱스와 값 쌍 반환 */
inline fun <T> List<T>.findIndexed(predicate: (T) -> Boolean): Pair<Int, T>? =
    asSequence().withIndex().firstOrNull { predicate(it.value) }?.let { it.index to it.value }

/** 조건을 만족하는 모든 인덱스 반환 */
inline fun <T> List<T>.indicesWhere(predicate: (T) -> Boolean): List<Int> =
    indices.filter { predicate(this[it]) }

/** 이진 탐색 (정렬된 리스트에서) */
fun <T : Comparable<T>> List<T>.binarySearch(target: T): Int {
    var low = 0; var high = size - 1
    while (low <= high) {
        val mid = (low + high) ushr 1
        when {
            this[mid] < target -> low = mid + 1
            this[mid] > target -> high = mid - 1
            else -> return mid
        }
    }
    return -1
}

/** 특정 값이 몇 번 등장하는지 */
fun <T> Iterable<T>.countOf(target: T): Int = count { it == target }

/** 조건별 카운트 */
inline fun <T> Iterable<T>.countWhere(predicate: (T) -> Boolean): Int = count(predicate)

/** 특정 조건의 요소가 몇 % 인지 */
inline fun <T> List<T>.percentageWhere(predicate: (T) -> Boolean): Double =
    if (isEmpty()) 0.0 else count(predicate).toDouble() / size * 100.0

/** none/any/all 조합 유틸 */
inline fun <T> Iterable<T>.noneOrAll(predicate: (T) -> Boolean): Boolean =
    none(predicate) || all(predicate)
```

**사용 예시**

```kotlin
val list = listOf(10, 30, 20, 50, 40)

println(list.findIndexed { it > 25 })       // (1, 30)
println(list.indicesWhere { it > 25 })      // [1, 3, 4]
println(list.sorted().binarySearch(40))     // 3
println(list.countOf(30))                   // 1
println(list.countWhere { it >= 30 })       // 3
println(list.percentageWhere { it >= 30 })  // 60.0

val flags = listOf(true, true, true)
println(flags.noneOrAll { it })             // true (모두 true)
```

---

## 7. 누적 합산 / 누적 변환

```kotlin
/** 누적 합 (Running Sum) */
fun List<Int>.runningSum(): List<Int> =
    runningFold(0) { acc, v -> acc + v }.drop(1)

fun List<Double>.runningSumDouble(): List<Double> =
    runningFold(0.0) { acc, v -> acc + v }.drop(1)

/** 누적 최댓값 */
fun <T : Comparable<T>> List<T>.runningMax(): List<T> {
    if (isEmpty()) return emptyList()
    return runningFold(first()) { acc, v -> if (v > acc) v else acc }.drop(1)
}

/** 누적 최솟값 */
fun <T : Comparable<T>> List<T>.runningMin(): List<T> {
    if (isEmpty()) return emptyList()
    return runningFold(first()) { acc, v -> if (v < acc) v else acc }.drop(1)
}

/** 이동 평균 (Moving Average) */
fun List<Double>.movingAverage(windowSize: Int): List<Double> =
    windowed(windowSize) { it.average() }

/** 각 요소의 전체 합 대비 비율 */
fun List<Int>.toRatios(): List<Double> {
    val total = sum().toDouble()
    return if (total == 0.0) map { 0.0 } else map { it / total }
}
```

**사용 예시**

```kotlin
val sales = listOf(100, 200, 150, 300, 250)

println(sales.runningSum())        // [100, 300, 450, 750, 1000]
println(sales.runningMax())        // [100, 200, 200, 300, 300]
println(sales.runningMin())        // [100, 100, 100, 100, 100]

val prices = listOf(10.0, 12.0, 11.0, 13.0, 14.0, 12.0)
println(prices.movingAverage(3))   // [11.0, 12.0, 12.67, 13.0]

println(listOf(1, 3, 6).toRatios()) // [0.1, 0.3, 0.6]
```

---

## 8. 다중 기준 정렬

```kotlin
/** 다중 기준 정렬 (여러 Comparator 순서대로 적용) */
fun <T> Iterable<T>.sortedByMultiple(vararg comparators: Comparator<T>): List<T> =
    sortedWith(comparators.reduce { acc, c -> acc.thenComparing(c) })

/** 특정 값이 우선 정렬되도록 */
inline fun <T, K> Iterable<T>.sortedWithPriority(
    prioritySelector: (T) -> K,
    priorityOrder: List<K>,
    crossinline thenBy: (T) -> Comparable<*>? = { null }
): List<T> = sortedWith(
    compareBy(
        { priorityOrder.indexOf(prioritySelector(it)).let { i -> if (i < 0) Int.MAX_VALUE else i } },
        { @Suppress("UNCHECKED_CAST") (thenBy(it) as? Comparable<Any?>) }
    )
)

/** null을 뒤로 보내는 정렬 */
inline fun <T, R : Comparable<R>> Iterable<T>.sortedByNullsLast(
    crossinline selector: (T) -> R?
): List<T> = sortedWith(compareBy(nullsLast(naturalOrder())) { selector(it) })

/** null을 앞으로 보내는 정렬 */
inline fun <T, R : Comparable<R>> Iterable<T>.sortedByNullsFirst(
    crossinline selector: (T) -> R?
): List<T> = sortedWith(compareBy(nullsFirst(naturalOrder())) { selector(it) })
```

**사용 예시**

```kotlin
data class Product(val name: String, val category: String, val price: Int)

val products = listOf(
    Product("iPad", "태블릿", 900000),
    Product("Galaxy S", "스마트폰", 1200000),
    Product("iPhone", "스마트폰", 1500000),
    Product("Galaxy Tab", "태블릿", 700000),
)

// 카테고리 오름차순 → 가격 내림차순
val sorted = products.sortedByMultiple(
    compareBy { it.category },
    compareByDescending { it.price }
)
sorted.forEach { println("${it.category} / ${it.name} / ${it.price}") }
// 스마트폰 / iPhone / 1500000
// 스마트폰 / Galaxy S / 1200000
// 태블릿 / iPad / 900000
// 태블릿 / Galaxy Tab / 700000

// 특정 카테고리를 맨 앞에
val prioritized = products.sortedWithPriority(
    prioritySelector = { it.category },
    priorityOrder = listOf("스마트폰", "태블릿"),
    thenBy = { it.price }
)
prioritized.forEach { println("${it.name}") }
// Galaxy S, iPhone, Galaxy Tab, iPad
```

---

## 9. 실전 조합 예제

### 예제 — 주문 통계 대시보드

```kotlin
data class Order(val userId: Int, val amount: Int, val status: String)

val orders = listOf(
    Order(1, 3000, "완료"), Order(2, 5000, "완료"),
    Order(1, 2000, "취소"), Order(3, 8000, "완료"),
    Order(2, 1000, "취소"), Order(3, 4000, "완료"),
)

// 1) 완료 주문만 필터 후 유저별 합산
val completedByUser = orders
    .filter { it.status == "완료" }
    .groupSumBy({ it.userId }, { it.amount })
println(completedByUser) // {1=3000, 2=5000, 3=12000}

// 2) 상위 2명
println(completedByUser.topN(2)) // {3=12000, 2=5000}

// 3) 전체 완료 금액 누적 합
val dailySales = listOf(3000, 5000, 12000)
println(dailySales.runningSum()) // [3000, 8000, 20000]

// 4) 각 유저 비율
println(listOf(3000, 5000, 12000).toRatios().map { "%.1f%%".format(it * 100) })
// [15.0%, 25.0%, 60.0%]

// 5) 유저 id → 총액 Map 뒤집기 (확인용)
val flipped = completedByUser.inverted()
println(flipped) // {3000=1, 5000=2, 12000=3}
```

---

## 10. 정리

| 카테고리 | 주요 함수 |
|----------|----------|
| 그룹핑/집계 | `groupBy`, `eachCount`, `groupSumBy`, `groupAverageBy`, `topNGroupsBy` |
| Zip/Combine | `zip`, `zipWithNext`, `zipOrNull`, `zip3`, `unzipToLists` |
| Flatten | `flatten`, `flatMap`, `flatMapNotNull`, `flatMapIf`, `flattenValues` |
| Set 연산 | `union`, `intersect`, `subtract`, `symmetricDifference`, `overlaps` |
| Map 조작 | `mergeWith`, `inverted`, `pick`, `omit`, `filterNotNullValues`, `topN` |
| 검색/탐색 | `findIndexed`, `indicesWhere`, `binarySearch`, `countOf`, `percentageWhere` |
| 누적 합산 | `runningSum`, `runningMax`, `runningMin`, `movingAverage`, `toRatios` |
| 다중 정렬 | `sortedByMultiple`, `sortedWithPriority`, `sortedByNullsLast` |

- 자주 쓰는 패턴은 `CollectionExt.kt` 파일로 모아 팀 전체에서 재사용하세요.
- 대량 데이터 처리 시에는 `asSequence()`를 활용해 중간 리스트 생성 비용을 줄이세요.

---

## 참고

- [Kotlin 공식 문서 - Collections Overview](https://kotlinlang.org/docs/collections-overview.html)
- [Kotlin 공식 문서 - Collection Operations](https://kotlinlang.org/docs/collection-operations.html)
