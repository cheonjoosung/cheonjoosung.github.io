---
title: (Kotlin/코틀린) operator rangeTo, contains 완전 정리
tags: [ Kotlin ]
style: fill
color: dark
description: Kotlin의 operator fun rangeTo와 contains를 구현해 커스텀 타입에서 .. 범위 연산자와 in 검사를 사용하는 방법을 정리합니다.
---

---

## 1. rangeTo와 contains란?

| 문법 | 변환 | 연산자 |
|------|------|--------|
| `a..b` | `a.rangeTo(b)` | `operator fun rangeTo` |
| `x in range` | `range.contains(x)` | `operator fun contains` |
| `x !in range` | `!range.contains(x)` | 동일 |

표준 타입(`Int`, `Char`, `LocalDate` 등)은 이미 구현되어 있습니다.  
커스텀 클래스에도 동일하게 정의할 수 있습니다.

---

## 2. rangeTo 구현

```kotlin
data class Version(val major: Int, val minor: Int) : Comparable<Version> {
    override fun compareTo(other: Version): Int =
        compareValuesBy(this, other, { it.major }, { it.minor })
}

operator fun Version.rangeTo(other: Version): ClosedRange<Version> =
    object : ClosedRange<Version> {
        override val start = this@rangeTo
        override val endInclusive = other
    }

val v1_0 = Version(1, 0)
val v2_0 = Version(2, 0)
val v1_5 = Version(1, 5)

val range = v1_0..v2_0
println(v1_5 in range)   // true
println(Version(3, 0) in range)  // false
```

---

## 3. contains 구현

`in` 연산자만 별도로 정의할 수도 있습니다.

```kotlin
class IpRange(private val start: String, private val end: String) {
    private fun ipToLong(ip: String): Long =
        ip.split(".").fold(0L) { acc, s -> acc * 256 + s.toLong() }

    operator fun contains(ip: String): Boolean {
        val ipLong = ipToLong(ip)
        return ipLong in ipToLong(start)..ipToLong(end)
    }
}

val range = IpRange("192.168.0.1", "192.168.0.255")
println("192.168.0.100" in range)  // true
println("10.0.0.1" in range)       // false
```

---

## 4. 날짜 범위 — 실전 패턴

```kotlin
data class DateRange(val from: LocalDate, val to: LocalDate) {
    operator fun contains(date: LocalDate): Boolean =
        date in from..to  // LocalDate는 Comparable 구현

    operator fun contains(other: DateRange): Boolean =
        other.from >= from && other.to <= to
}

operator fun LocalDate.rangeTo(other: LocalDate) = DateRange(this, other)

val summer = LocalDate.of(2026, 6, 1)..LocalDate.of(2026, 8, 31)

println(LocalDate.of(2026, 7, 15) in summer)   // true
println(LocalDate.of(2026, 9, 1) in summer)    // false
```

---

## 5. when 표현식과 결합

`in` 연산자를 구현하면 `when` 표현식에서도 자연스럽게 사용됩니다.

```kotlin
val score = 85
val grade = when (score) {
    in 90..100 -> "A"
    in 80..89  -> "B"
    in 70..79  -> "C"
    else       -> "F"
}
println(grade)  // B
```

커스텀 범위도 동일하게 활용됩니다.

```kotlin
val version = Version(1, 5)
val message = when (version) {
    in Version(1, 0)..Version(1, 9) -> "v1.x 지원"
    in Version(2, 0)..Version(2, 9) -> "v2.x 지원"
    else -> "미지원 버전"
}
println(message)  // v1.x 지원
```

---

## 6. 컬렉션의 contains와 혼동 주의

```kotlin
val list = listOf(1, 2, 3)
println(2 in list)  // list.contains(2) → true

// 컬렉션에서는 operator fun contains가 이미 구현되어 있음
// 커스텀 클래스에서는 직접 정의 필요
```

---

## 7. 정리

- `operator fun rangeTo(other: T): ClosedRange<T>` → `a..b` 문법 활성화
- `operator fun contains(value: T): Boolean` → `x in obj` 문법 활성화
- 커스텀 타입이 `Comparable`을 구현하면 `ClosedRange`와 자동 연동
- `when` 표현식의 `in` 조건, for 루프 순회 등과 자연스럽게 결합
