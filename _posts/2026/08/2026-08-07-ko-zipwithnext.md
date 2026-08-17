---
title: (Kotlin/코틀린) windowed, chunked, zipWithNext 완전 정리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin 컬렉션의 windowed, chunked, zipWithNext 함수로 슬라이딩 윈도우, 배치 처리, 인접 원소 비교를 구현하는 방법을 정리합니다.
---

---

## 1. chunked — 고정 크기 배치로 분할

리스트를 **지정한 크기의 청크(Chunk)** 로 나눕니다. 마지막 청크는 더 작을 수 있습니다.

```kotlin
val numbers = (1..10).toList()

val chunks = numbers.chunked(3)
println(chunks)
// [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10]]

// 변환 함수와 함께
val sums = numbers.chunked(3) { chunk -> chunk.sum() }
println(sums)  // [6, 15, 24, 10]
```

```
chunked(3) 동작:
[1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
→ [1,2,3]  [4,5,6]  [7,8,9]  [10]
    ↑ 이동 없이 순차 분할
```

---

## 2. windowed — 슬라이딩 윈도우

**창(window)을 슬라이딩**하며 겹치는 부분 포함 모든 윈도우를 반환합니다.

```kotlin
val numbers = (1..7).toList()

// size=3, step=1 (기본값)
val windows = numbers.windowed(3)
println(windows)
// [[1,2,3], [2,3,4], [3,4,5], [4,5,6], [5,6,7]]

// step 변경: 얼마씩 이동할지
val step2 = numbers.windowed(size = 3, step = 2)
println(step2)
// [[1,2,3], [3,4,5], [5,6,7]]

// partialWindows: 마지막 불완전한 윈도우 포함
val partial = numbers.windowed(size = 3, step = 1, partialWindows = true)
println(partial)
// [[1,2,3], [2,3,4], [3,4,5], [4,5,6], [5,6,7], [6,7], [7]]
```

```
windowed(3) 동작:
[1, 2, 3, 4, 5, 6, 7]
 ↑___↑         → [1,2,3]
    ↑___↑      → [2,3,4]
       ↑___↑   → [3,4,5] ...
```

### chunked vs windowed

```
chunked(3):   [1,2,3] [4,5,6] [7,8,9]  ← 겹치지 않음
windowed(3):  [1,2,3] [2,3,4] [3,4,5]  ← 슬라이딩, 겹침
```

---

## 3. zipWithNext — 인접 원소 쌍

각 원소와 **다음 원소를 쌍(Pair)으로** 묶습니다.

```kotlin
val numbers = listOf(1, 3, 6, 10, 15)

val pairs = numbers.zipWithNext()
println(pairs)
// [(1, 3), (3, 6), (6, 10), (10, 15)]

// 변환 함수와 함께: 인접 두 원소의 차이
val differences = numbers.zipWithNext { a, b -> b - a }
println(differences)  // [2, 3, 4, 5]
```

```
zipWithNext 동작:
[1, 3, 6, 10, 15]
 ↑  ↑
 ↑___↑ → (1,3)
    ↑___↑ → (3,6)
       ↑____↑ → (6,10)
           ↑____↑ → (10,15)
```

---

## 4. 실전 패턴 — 이동 평균 계산

```kotlin
val prices = listOf(100.0, 102.5, 98.0, 105.0, 103.5, 107.0, 110.0)

// 3일 이동 평균
val movingAverage = prices.windowed(3) { window ->
    window.average()
}

prices.drop(2).zip(movingAverage).forEach { (price, avg) ->
    println("가격: $price, 3일 평균: ${"%.2f".format(avg)}")
}
// 가격: 98.0, 3일 평균: 100.17
// 가격: 105.0, 3일 평균: 101.83
// ...
```

---

## 5. 실전 패턴 — 배치 API 처리

```kotlin
// API가 한 번에 최대 50개만 처리 가능한 경우
suspend fun deleteUsers(userIds: List<Int>) {
    userIds
        .chunked(50)  // 50개씩 나누기
        .forEach { batch ->
            api.deleteUsers(batch)
            println("${batch.size}개 삭제 완료")
        }
}

// DB 배치 insert
suspend fun insertAll(items: List<Item>) {
    items.chunked(100).forEach { chunk ->
        db.insertAll(chunk)
    }
}
```

---

## 6. 실전 패턴 — 연속 변화 감지

```kotlin
data class SensorData(val timestamp: Long, val value: Double)

val readings = listOf(
    SensorData(1000, 36.5),
    SensorData(2000, 36.7),
    SensorData(3000, 38.2),  // 급격한 상승
    SensorData(4000, 38.1),
    SensorData(5000, 37.0)
)

// 인접 측정값 차이로 급격한 변화 감지
val alerts = readings.zipWithNext { prev, curr ->
    val diff = curr.value - prev.value
    if (Math.abs(diff) > 1.0) {
        "⚠️ 급변: ${prev.value} → ${curr.value} (${if (diff > 0) "+" else ""}${"%.1f".format(diff)})"
    } else null
}.filterNotNull()

alerts.forEach { println(it) }
// ⚠️ 급변: 36.7 → 38.2 (+1.5)
```

---

## 7. 실전 패턴 — 페이지 슬라이더

```kotlin
data class Page(val number: Int, val items: List<String>)

val allItems = (1..20).map { "Item $it" }

// 페이지당 5개씩
val pages = allItems.chunked(5).mapIndexed { index, items ->
    Page(index + 1, items)
}

println("총 ${pages.size}페이지")  // 4페이지

// 슬라이딩 윈도우로 페이지 네비게이션 컨텍스트 생성
val pageContext = pages.windowed(3, partialWindows = true)
pageContext.forEach { window ->
    val current = window[window.size / 2]
    println("현재: Page ${current.number}, 컨텍스트: ${window.map { it.number }}")
}
```

---

## 8. windowed로 연속 중복 감지

```kotlin
val sequence = listOf(1, 1, 2, 3, 3, 3, 4, 5, 5)

// 인접한 두 값이 같은 위치 찾기
val duplicatePositions = sequence.zipWithNext().mapIndexedNotNull { index, (a, b) ->
    if (a == b) index else null
}
println(duplicatePositions)  // [0, 3, 4, 7]

// 연속 구간 그룹화
val grouped = sequence.zipWithNext { a, b -> if (a == b) a else null }
    .filterNotNull()
    .distinct()
println("중복 값: $grouped")  // [1, 3, 5]
```

---

## 9. 정리

| 함수 | 동작 | 주요 용도 |
|------|------|----------|
| `chunked(n)` | n개씩 겹치지 않게 분할 | 배치 처리, 페이지네이션 |
| `windowed(n)` | n개 창을 슬라이딩 | 이동 평균, 시계열 분석 |
| `windowed(n, step)` | step씩 이동하며 윈도우 | 다운샘플링 |
| `zipWithNext()` | 인접 원소 쌍 | 변화량 계산, 중복 감지 |
| `zipWithNext { a, b -> }` | 인접 원소를 변환 | 차이값, 비교 결과 |

- `chunked` ≈ `windowed(n, step=n)` (겹치지 않는 윈도우)
- `zipWithNext()` ≈ `windowed(2).map { (a, b) -> a to b }`
