---
title: (Kotlin/코틀린) fold vs reduce vs scan 차이
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin 컬렉션의 fold, reduce, scan 함수의 동작 원리와 차이점, 실전 활용 패턴을 정리합니다.
---

---

## 1. 세 함수 한눈에 비교

| 함수 | 초기값 | 빈 컬렉션 | 반환 타입 | 중간값 |
|------|--------|----------|----------|--------|
| `fold` | 있음 | 초기값 반환 | 자유롭게 지정 | 없음 (최종값만) |
| `reduce` | 없음 | 예외 발생 | 컬렉션 원소 타입 | 없음 (최종값만) |
| `scan` | 있음 | [초기값] 반환 | 자유롭게 지정 | 있음 (모든 중간값) |

---

## 2. fold — 초기값부터 시작하는 누적

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

// 합계 (초기값 0)
val sum = numbers.fold(0) { acc, n -> acc + n }
println(sum)  // 15

// 곱 (초기값 1)
val product = numbers.fold(1) { acc, n -> acc * n }
println(product)  // 120

// 문자열 연결 (초기값 "")
val joined = numbers.fold("") { acc, n -> "$acc$n," }
println(joined)  // "1,2,3,4,5,"
```

```
fold 동작:
acc=0 → acc+1=1
acc=1 → acc+2=3
acc=3 → acc+3=6
acc=6 → acc+4=10
acc=10 → acc+5=15  ← 최종값
```

### 타입 변환에 fold 활용

```kotlin
data class Item(val name: String, val price: Int, val quantity: Int)

val cart = listOf(
    Item("노트북", 1200000, 1),
    Item("마우스", 35000, 2),
    Item("키보드", 85000, 1)
)

// Int → Long으로 타입 변환하며 합산
val totalPrice = cart.fold(0L) { acc, item ->
    acc + item.price.toLong() * item.quantity
}
println("총액: ${totalPrice}원")  // 1355000원

// List<Item> → Map<String, Int>으로 변환
val priceMap = cart.fold(mutableMapOf<String, Int>()) { acc, item ->
    acc[item.name] = item.price
    acc
}
```

---

## 3. reduce — 첫 번째 원소를 초기값으로 사용

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

val sum = numbers.reduce { acc, n -> acc + n }
println(sum)  // 15

val max = numbers.reduce { acc, n -> maxOf(acc, n) }
println(max)  // 5
```

```
reduce 동작:
acc=1(첫 원소) → acc+2=3
acc=3 → acc+3=6
acc=6 → acc+4=10
acc=10 → acc+5=15
```

### 빈 컬렉션 주의

```kotlin
val empty = emptyList<Int>()

// reduce: 빈 컬렉션에서 UnsupportedOperationException 발생
// empty.reduce { acc, n -> acc + n }  // ⚠️ 예외

// fold: 빈 컬렉션에서 초기값 반환 (안전)
val result = empty.fold(0) { acc, n -> acc + n }
println(result)  // 0

// reduceOrNull: 빈 컬렉션에서 null 반환
val nullableResult = empty.reduceOrNull { acc, n -> acc + n }
println(nullableResult)  // null
```

---

## 4. scan — 모든 중간 누적값 반환

`fold`와 동일하게 동작하지만, **각 단계의 중간값을 모두 포함한 List를 반환**합니다.

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

val runningSum = numbers.scan(0) { acc, n -> acc + n }
println(runningSum)  // [0, 1, 3, 6, 10, 15]
//                      ↑ 초기값 포함

val runningProduct = numbers.scan(1) { acc, n -> acc * n }
println(runningProduct)  // [1, 1, 2, 6, 24, 120]
```

```
scan 동작:
[0]           ← 초기값
[0, 0+1=1]
[0, 1, 1+2=3]
[0, 1, 3, 3+3=6]
[0, 1, 3, 6, 6+4=10]
[0, 1, 3, 6, 10, 10+5=15]
```

---

## 5. 실전 패턴 — 누적 합계 차트 데이터

```kotlin
data class DailySales(val date: String, val amount: Int)

val sales = listOf(
    DailySales("08-01", 150000),
    DailySales("08-02", 230000),
    DailySales("08-03", 180000),
    DailySales("08-04", 320000),
    DailySales("08-05", 270000)
)

// 일별 누적 매출 계산
val cumulativeSales = sales
    .map { it.amount }
    .scan(0) { acc, amount -> acc + amount }
    .drop(1)  // 초기값 0 제거
    .zip(sales) { cumulative, day ->
        "${day.date}: ${day.amount}원 (누적: ${cumulative}원)"
    }

cumulativeSales.forEach { println(it) }
// 08-01: 150000원 (누적: 150000원)
// 08-02: 230000원 (누적: 380000원)
// 08-03: 180000원 (누적: 560000원)
// 08-04: 320000원 (누적: 880000원)
// 08-05: 270000원 (누적: 1150000원)
```

---

## 6. 실전 패턴 — fold로 검증 체인

```kotlin
data class ValidationRule(val name: String, val check: (String) -> Boolean)

val passwordRules = listOf(
    ValidationRule("8자 이상") { it.length >= 8 },
    ValidationRule("대문자 포함") { it.any { c -> c.isUpperCase() } },
    ValidationRule("숫자 포함") { it.any { c -> c.isDigit() } },
    ValidationRule("특수문자 포함") { it.any { c -> !c.isLetterOrDigit() } }
)

fun validatePassword(password: String): List<String> {
    return passwordRules.fold(mutableListOf()) { errors, rule ->
        if (!rule.check(password)) errors.add("✗ ${rule.name}")
        errors
    }
}

val errors = validatePassword("kotlin")
if (errors.isEmpty()) println("비밀번호 OK")
else errors.forEach { println(it) }
// ✗ 8자 이상
// ✗ 대문자 포함
// ✗ 숫자 포함
// ✗ 특수문자 포함
```

---

## 7. foldRight / reduceRight

오른쪽에서 왼쪽으로 누적합니다.

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

// foldRight: 오른쪽부터 누적
val result = numbers.foldRight("") { n, acc -> "$n$acc" }
println(result)  // "12345"

// fold (왼쪽부터): acc="", n=1 → "1" → n=2 → "12" ...
val leftResult = numbers.fold("") { acc, n -> "$acc$n" }
println(leftResult)  // "12345"

// 순서가 중요한 경우
val subtractRight = numbers.foldRight(0) { n, acc -> n - acc }
println(subtractRight)  // 1-(2-(3-(4-(5-0)))) = 3

val subtractLeft = numbers.fold(0) { acc, n -> acc - n }
println(subtractLeft)  // ((((0-1)-2)-3)-4)-5 = -15
```

---

## 8. 선택 가이드

```
초기값이 필요한가?
    ↓ Yes → fold / scan
    ↓ No  → reduce (빈 컬렉션 주의) 또는 reduceOrNull

중간 과정도 필요한가?
    ↓ Yes → scan
    ↓ No  → fold / reduce

반환 타입이 원소 타입과 다른가?
    ↓ Yes → fold (reduce는 타입 변환 불가)
    ↓ No  → reduce 또는 fold 모두 가능
```

---

## 9. 정리

- **`fold(initial)`**: 초기값 + 모든 원소 누적 → 최종값 (타입 변환 가능)
- **`reduce`**: 첫 원소 초기값 + 나머지 누적 → 최종값 (빈 컬렉션 예외 주의)
- **`scan(initial)`**: fold와 동일하지만 초기값 포함 모든 중간값 리스트 반환
- `reduceOrNull`: 빈 컬렉션 안전 처리
- 타입이 바뀌거나 초기값이 필요하면 `fold`, 누적 추이가 필요하면 `scan`
