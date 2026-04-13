---
title: (Kotlin/코틀린) 실무에서 바로 쓰는 Number 확장함수 모음
tags: [ Kotlin ]
style: fill
color: dark
description: (Kotlin/코틀린) Int, Long, Double, Float 등 숫자 타입에 대한 범위 체크, 포맷, 변환, 단위 계산, 안전 나눗셈 등 실무에서 자주 쓰는 Number 확장함수를 샘플 코드와 함께 정리합니다.
---

## 개요

- 숫자를 다루다 보면 반복되는 패턴이 많습니다.
  - 범위 체크, 소수점 포맷, 금액 표시, null 안전 파싱, 단위 변환 등
- 이를 확장함수로 만들어두면 코드가 훨씬 간결해집니다.
- 이 글에서는 실무에서 바로 꺼내 쓸 수 있는 Number 확장함수를 카테고리별로 정리합니다.

---

## 1. 범위 체크 / 검증

```kotlin
/** 양수인지 (0 제외) */
fun Int.isPositive(): Boolean = this > 0
fun Long.isPositive(): Boolean = this > 0L
fun Double.isPositive(): Boolean = this > 0.0

/** 음수인지 */
fun Int.isNegative(): Boolean = this < 0
fun Long.isNegative(): Boolean = this < 0L
fun Double.isNegative(): Boolean = this < 0.0

/** 0 이상인지 */
fun Int.isZeroOrPositive(): Boolean = this >= 0

/** 특정 범위 안에 있는지 (양 끝 포함) */
fun Int.isBetween(min: Int, max: Int): Boolean = this in min..max
fun Long.isBetween(min: Long, max: Long): Boolean = this in min..max
fun Double.isBetween(min: Double, max: Double): Boolean = this in min..max

/** 짝수/홀수 */
fun Int.isEven(): Boolean = this % 2 == 0
fun Int.isOdd(): Boolean = this % 2 != 0
```

**사용 예시**

```kotlin
println(5.isPositive())          // true
println((-3).isNegative())       // true
println(7.isBetween(1, 10))      // true
println(4.isEven())              // true
println(3.isOdd())               // true
```

---

## 2. 안전 파싱

```kotlin
/** 문자열 → Int, 실패 시 null */
fun String?.toIntOrNull2(): Int? = this?.trim()?.toIntOrNull()

/** 문자열 → Int, 실패 시 기본값 */
fun String?.toIntOrDefault(default: Int = 0): Int =
    this?.trim()?.toIntOrNull() ?: default

/** 문자열 → Double, 실패 시 기본값 */
fun String?.toDoubleOrDefault(default: Double = 0.0): Double =
    this?.trim()?.toDoubleOrNull() ?: default

/** 문자열 → Long, 실패 시 기본값 */
fun String?.toLongOrDefault(default: Long = 0L): Long =
    this?.trim()?.toLongOrNull() ?: default

/** 퍼센트 문자열 → Double ("85%" → 85.0) */
fun String?.percentToDouble(): Double =
    this?.trim()?.removeSuffix("%")?.toDoubleOrNull() ?: 0.0
```

**사용 예시**

```kotlin
println("42".toIntOrDefault())           // 42
println("abc".toIntOrDefault(-1))        // -1
println(null.toDoubleOrDefault(9.9))     // 9.9
println("85%".percentToDouble())         // 85.0
println("  123  ".toIntOrDefault())      // 123
```

---

## 3. 소수점 / 반올림 포맷

```kotlin
/** 소수점 n자리로 반올림한 Double 반환 */
fun Double.roundTo(decimals: Int): Double {
    val factor = Math.pow(10.0, decimals.toDouble())
    return Math.round(this * factor) / factor
}

/** 소수점 n자리 문자열로 포맷 */
fun Double.format(decimals: Int): String = "%.${decimals}f".format(this)
fun Float.format(decimals: Int): String = "%.${decimals}f".format(this)

/** 소수점 제거 — 정수면 정수처럼, 아니면 소수점 표시 */
fun Double.toCleanString(): String =
    if (this == Math.floor(this) && !this.isInfinite()) this.toLong().toString()
    else this.toString()
```

**사용 예시**

```kotlin
println(3.14159.roundTo(2))      // 3.14
println(3.14159.format(3))       // "3.142"
println(2.0.toCleanString())     // "2"
println(2.5.toCleanString())     // "2.5"
```

---

## 4. 금액 / 숫자 표시 포맷

```kotlin
import java.text.DecimalFormat
import java.text.NumberFormat
import java.util.Locale

/** 천 단위 콤마 포맷 (1234567 → "1,234,567") */
fun Int.toCommaString(): String = DecimalFormat("#,###").format(this)
fun Long.toCommaString(): String = DecimalFormat("#,###").format(this)
fun Double.toCommaString(): String = DecimalFormat("#,###.##").format(this)

/** 원화 표시 (1234567 → "₩1,234,567") */
fun Int.toWon(): String = "₩${this.toCommaString()}"
fun Long.toWon(): String = "₩${this.toCommaString()}"

/** 달러 표시 */
fun Double.toDollar(): String =
    NumberFormat.getCurrencyInstance(Locale.US).format(this)

/** K/M 단위 축약 (1500 → "1.5K", 2000000 → "2M") */
fun Long.toAbbreviatedString(): String = when {
    this >= 1_000_000L -> "${(this / 100_000L) / 10.0}M"
    this >= 1_000L     -> "${(this / 100L) / 10.0}K"
    else               -> this.toString()
}
fun Int.toAbbreviatedString(): String = this.toLong().toAbbreviatedString()
```

**사용 예시**

```kotlin
println(1234567.toCommaString())         // "1,234,567"
println(50000.toWon())                   // "₩50,000"
println(1299.99.toDollar())              // "$1,299.99"
println(1500.toAbbreviatedString())      // "1.5K"
println(2_000_000.toAbbreviatedString()) // "2.0M"
```

---

## 5. 안전 나눗셈 / 수학 연산

```kotlin
/** 0 나누기 방지 나눗셈, 0으로 나눌 경우 0.0 반환 */
fun Int.safeDivide(other: Int): Double =
    if (other == 0) 0.0 else this.toDouble() / other

fun Double.safeDivide(other: Double): Double =
    if (other == 0.0) 0.0 else this / other

/** 퍼센트 계산 (part / total * 100) */
fun Int.percentOf(total: Int): Double =
    if (total == 0) 0.0 else this.toDouble() / total * 100.0

/** 절댓값 */
fun Int.abs(): Int = kotlin.math.abs(this)
fun Double.abs(): Double = kotlin.math.abs(this)

/** 두 값 사이의 차이 */
fun Int.diff(other: Int): Int = kotlin.math.abs(this - other)
fun Double.diff(other: Double): Double = kotlin.math.abs(this - other)

/** n의 배수로 올림 */
fun Int.ceilToMultipleOf(multiple: Int): Int =
    if (multiple <= 0) this
    else if (this % multiple == 0) this
    else this + (multiple - this % multiple)

/** n의 배수로 내림 */
fun Int.floorToMultipleOf(multiple: Int): Int =
    if (multiple <= 0) this else (this / multiple) * multiple
```

**사용 예시**

```kotlin
println(10.safeDivide(0))            // 0.0
println(10.safeDivide(3))            // 3.3333...
println(3.percentOf(10))             // 30.0
println((-5).abs())                  // 5
println(7.diff(12))                  // 5
println(13.ceilToMultipleOf(5))      // 15
println(13.floorToMultipleOf(5))     // 10
```

---

## 6. 단위 변환

```kotlin
/** dp → px 변환 (Android Context 없이 density 직접 전달) */
fun Int.dpToPx(density: Float): Int = (this * density + 0.5f).toInt()
fun Float.dpToPx(density: Float): Float = this * density

/** px → dp 변환 */
fun Int.pxToDp(density: Float): Float = this / density
fun Float.pxToDp(density: Float): Float = this / density

/** 초 → 분:초 문자열 (90 → "1:30") */
fun Int.toMinuteSecond(): String {
    val m = this / 60
    val s = this % 60
    return "%d:%02d".format(m, s)
}

/** 밀리초 → 시:분:초 문자열 */
fun Long.toHourMinuteSecond(): String {
    val total = this / 1000
    val h = total / 3600
    val m = (total % 3600) / 60
    val s = total % 60
    return if (h > 0) "%d:%02d:%02d".format(h, m, s)
    else "%d:%02d".format(m, s)
}

/** 바이트 → 사람이 읽기 좋은 파일 크기 */
fun Long.toReadableFileSize(): String = when {
    this >= 1_073_741_824L -> "%.1f GB".format(this / 1_073_741_824.0)
    this >= 1_048_576L     -> "%.1f MB".format(this / 1_048_576.0)
    this >= 1_024L         -> "%.1f KB".format(this / 1_024.0)
    else                   -> "$this B"
}
```

**사용 예시**

```kotlin
val density = 2.0f
println(16.dpToPx(density))              // 32
println(90.toMinuteSecond())             // "1:30"
println(3661_000L.toHourMinuteSecond())  // "1:01:01"
println(1_500_000L.toReadableFileSize()) // "1.4 MB"
println(2_147_483_648L.toReadableFileSize()) // "2.0 GB"
```

---

## 7. 범위 제한 / 클램프

```kotlin
/** min~max 범위 안으로 강제 제한 */
fun Int.clamp(min: Int, max: Int): Int = this.coerceIn(min, max)
fun Long.clamp(min: Long, max: Long): Long = this.coerceIn(min, max)
fun Double.clamp(min: Double, max: Double): Double = this.coerceIn(min, max)
fun Float.clamp(min: Float, max: Float): Float = this.coerceIn(min, max)

/** 0.0 ~ 1.0 사이로 제한 (Progress, Alpha 등에 사용) */
fun Double.clamp01(): Double = this.coerceIn(0.0, 1.0)
fun Float.clamp01(): Float = this.coerceIn(0f, 1f)

/** 최솟값 보장 */
fun Int.atLeast(min: Int): Int = maxOf(this, min)
fun Double.atLeast(min: Double): Double = maxOf(this, min)

/** 최댓값 보장 */
fun Int.atMost(max: Int): Int = minOf(this, max)
fun Double.atMost(max: Double): Double = minOf(this, max)
```

**사용 예시**

```kotlin
println(150.clamp(0, 100))      // 100
println((-5).clamp(0, 100))     // 0
println(1.5.clamp01())          // 1.0
println((-0.3).clamp01())       // 0.0
println(3.atLeast(5))           // 5
println(10.atMost(7))           // 7
```

---

## 8. 실전 조합 예제

### 예제 1 — 할인율 표시

```kotlin
fun calculateDiscountLabel(original: Int, discounted: Int): String {
    val rate = discounted.percentOf(original)
    val saved = (original - discounted).toWon()
    return "할인율 ${(100 - rate).roundTo(1)}% (${saved} 절약)"
}

println(calculateDiscountLabel(50000, 35000))
// 할인율 30.0% (₩15,000 절약)
```

---

### 예제 2 — 진행률 바 계산

```kotlin
fun getProgressFraction(current: Int, total: Int): Float =
    current.percentOf(total).toFloat().div(100f).clamp01()

println(getProgressFraction(3, 10))   // 0.3
println(getProgressFraction(12, 10))  // 1.0 (최대 제한)
```

---

### 예제 3 — 동영상 재생 시간 표시

```kotlin
fun formatDuration(currentMs: Long, totalMs: Long): String =
    "${currentMs.toHourMinuteSecond()} / ${totalMs.toHourMinuteSecond()}"

println(formatDuration(65_000L, 3_600_000L))
// "1:05 / 1:00:00"
```

---

### 예제 4 — 파일 업로드 진행률

```kotlin
fun uploadStatus(uploadedBytes: Long, totalBytes: Long): String {
    val percent = uploadedBytes.toDouble().div(totalBytes.toDouble())
        .times(100).clamp(0.0, 100.0).roundTo(1)
    return "${uploadedBytes.toReadableFileSize()} / ${totalBytes.toReadableFileSize()} ($percent%)"
}

println(uploadStatus(512_000L, 2_048_000L))
// "500.0 KB / 2.0 MB (25.0%)"
```

---

## 9. 정리

| 카테고리 | 주요 함수 |
|----------|----------|
| 범위 체크 | `isPositive`, `isBetween`, `isEven`, `isOdd` |
| 안전 파싱 | `toIntOrDefault`, `toDoubleOrDefault`, `percentToDouble` |
| 소수점 포맷 | `roundTo`, `format`, `toCleanString` |
| 금액 포맷 | `toCommaString`, `toWon`, `toDollar`, `toAbbreviatedString` |
| 수학 연산 | `safeDivide`, `percentOf`, `abs`, `diff`, `ceilToMultipleOf` |
| 단위 변환 | `dpToPx`, `toMinuteSecond`, `toHourMinuteSecond`, `toReadableFileSize` |
| 범위 제한 | `clamp`, `clamp01`, `atLeast`, `atMost` |

- 프로젝트 공통 유틸 파일 (`NumberExt.kt`)로 모아두면 팀 전체가 일관성 있게 사용할 수 있습니다.
